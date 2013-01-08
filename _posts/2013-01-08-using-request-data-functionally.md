---
layout: post
title: Using request data functionally for concise, testable controllers
date: 2013-01-08 23:00:00
---

A common pattern in web applications is to take data from a request and validate it in some way, before using it to perform actions on the model layer. A reasonable example using value objects might look something like this:

    private $parameters;
    private $usersService;
    private $entityManager;

    public function signupAction() {

    	$user = $this->usersService->createUser(
    		new EmailAddress($this->parameters['emailAddress']),
    		new Password($this->parameters['password'])
    	);

    	$user->setFirstName(new Name($this->parameters['firstName']));
    	$user->setLastName(new Name($this->parameters['lastName']));

    	$this->entitymanager->flush();
    }

Should an invalid parameter be passed to a value object, a GuardException is thrown which is handled by the dispatcher to produce an appropriate validation error response.

The first issue is a fatal error when a parameter doesn't exist. This can be solved with relative ease by creating a "Safe Array" implementing ArrayAccess, falling back to an empty string when no value is available, which will of course fail construction of any value object.

A more pressing issue is that this function has several outcomes and consequently takes quite a lot of effort to test:

	test_GivenInvalidEmailAddress_WhenSignupAction_ThenGuardException
	test_GivenInvalidPassword_WhenSignupAction_ThenGuardException
	test_GivenInvalidFirstName_WhenSignupAction_ThenGuardException
	test_GivenInvalidLastName_WhenSignupAction_ThenGuardException
	test_GivenValidRequest_WhenSignupAction_ThenUserIsCreated

What's more, in order to test that an invalid last name throws an exception, all other fields need to be populated with valid values. When I find myself in this position it suggests that I've missed an opportunity to abstract away some logic which as it turns out, I have.

The problems is with the parameters object; it really only offers us a simple layer of indirection before the native parameters which is essential but could be so much better.

A more useful parameters service could perform the mapping from source data to a value object:

	class ValueObjectService {

		private $map;
		private $source;

		public function get($key)
		{

			if (!isset($this->map[$key])) {
				throw new ErrorException('Some bad code is requesting a parameter that is not mapped: ' . $key);
			}

			if (!isset($this->source[$key])) {
				throw new RuntimeException('A required parameter has not been provided: ' . $key);
			}

			$fqcn = $this->map[$key];

			return new $fqcn($source[$key]);

		}
	}

This is a great example of why I love php's unique ability to switch between dynamic "magic" code at the boundaries, and strict, reliable code within the domain.

The beauty of this is that we now only need to test for success in our signupAction since error cases are tested by our ValueObjectService.

Testing the error case in this service can be done with a single example since the concept of mapping from a parameter to a value object has been appropriately abstracted. We've also improved the quality of the exceptions, distinguishing between invalid and missing values.

Our revised method now looks like this:

    public function signupAction()
    {
    	$user = $this->usersService->createUser(
    		$this->parameterService->get('emailAddress'),
    		$this->parameterService->get('password')
    	);

    	$user->setFirstName($this->parameterService->get('firstName'));
    	$user->setLastName($this->parameterService->get('lastName'));

    	$this->entitymanager->flush();
    }

Unfortunately the requirements have changed; first name and last name are now optional fields. The most obvious solution would be to add a has() method to our parameter service, allowing us to check optional fields before requesting them. However this solution reintroduces all the complexity we've just recovered from.

	if ($this->parameterService->has('firstName')) {
		$user->setFirstName($this->parameterService->get('firstName'));
	}

A more graceful solution pulls inspiration from functional programming and I'm not sure a method prefix has been coined for this yet but I refer to it as "giving".

	class ValueObjectService {

		private $map;
		private $source;

		public function give($key, $recipient)
		{
			if (!isset($this->map[$key])) {
				throw new ErrorException('Some bad code is requesting a parameter that is not mapped: ' . $key);
			}

			if (!isset($this->source[$key])) {
				return;
			}

			$fqcn = $this->map[$key];
			$valueObject = new $fqcn($source[$key]);

			$setter = 'set' . ucfirst($key);
			if (is_object($recipient) && has_method($recipient, $setter) {
				$recipient->$setter($valueObject);
				return;
			}

			if (is_callable($recipient)) {
				call_user_func($recipient, $valueObject);
				return;
			}

			throw new ErrorException('Some bad code has passed an invalid recipient.');
		}

	}

The give parameter will forward the created value object for $key to the recipient which can either be an object with the appropriate setter, or any callable.

So now we have a mechanism to ask our parameter service to pass along its value to a recipient if its source has the appropriate data, otherwise nothing happens, abstracting away the duplicated optional logic.

And so our final method is both readable and incredibly easy to test, something that is often very difficult to achieve on "controller" actions.

    public function signupAction()
    {
    	$user = $this->usersService->createUser(
    		$this->parameterService->get('emailAddress'),
    		$this->parameterService->get('password')
    	);

    	$this->parameterService->give('firstName', $user);
    	$this->parameterService->give('lastName', $user);

    	$this->entitymanager->flush();
    }