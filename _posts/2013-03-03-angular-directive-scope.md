---
layout: post
title: Angular directive scope
date: 2013-03-03 18:00:00
---

Directives are one of the many strengths of the AngularJS framework, providing a great way to add reusable behaviour to html. Unfortunately this comes at the cost of considerable complexity; while it is fairly simple to implement a single directive, the number of options available and the unfamiliar terminology make it difficult to know the best approach to ensure the directive is as reusable as possible.

I've been looking into the scope options, and [the official guide](http://docs.angularjs.org/guide/directive) does a decent job of explaining the attribute magic that comes with the isolate scope, but fails to explain the repercussions of different scope settings. For example, consider the following:

{% highlight html %}
<div ng-controller="ExampleCtrl">
    <div directive-one directive-two></div>
</div>
{% endhighlight %}

|Directive One|Directive Two|Result|
|:-----------:|:-----------:|------|
|undefined    |undefined    |Both use controller scope|
|undefined    |true         |Both use scope extending controller scope|
|undefined    |{}           |Both use isolate scope|
|true         |true         |Both use scope extending controller scope|
|true         |{}           |Error|
|{}           |{}           |Error|

Based on this information, here is some advice for choosing a directive scope option:

__Scope option undefined:__

 * Most reusable directive.
 * It is not safe to read from controller scope.
 * If you must write to scope, use prefixed variable names to avoid clobbering controller scope.
 * Never use directives to modify controller scope directly.

__Scope option isolate:__

 * Most restrictive directive.
 * Useful for complete behaviours that are not enhanced by other directives.
 * Using with undefined scope directives is allowed, but suggests this is not a lone directive. Prefer new instead.

__Scope option true:__

 * Use when directive needs to read from the controller scope and offer the possibility to be used with other directives.
 * Accessing models and evaluating expressions stored in attributes can still be achieved using the $parse service. For example:

{% highlight html %}
<div my-directive="user.name"></div>
{% endhighlight %}

{% highlight js %}
angular.module('example').directive('myDirective', function ($parse) {

    return {
        link: function (scope, element, attrs) {

            var myModelGetter = $parse(attrs.myDirective);

            myModelGetter(scope); // value of user.name
            myModelGetter.assign(scope, 'new name');

            scope.$watch(myModelGetter, function (value) {
                // do something with new value
            });
        }
    };

});
{% endhighlight %}

As a closing thought I think the system would be improved if isolate scopes were truly isolate and never shared with other directives on the same element. This would remove the two error cases, and there is already a suitable channel of communication by requiring a directive's controller.