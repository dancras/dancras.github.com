---
layout: post
title: Recursively widen an interface for testing with TypeScript, Part Two
date: 2022-03-28 17:45:00
---

In [Part One](/2022/03/14/recursively-widen-an-interface-for-testing-with-typescript-part-one.html) I explored a custom [Utility Type](https://www.typescriptlang.org/docs/handbook/utility-types.html) I've been using on recent projects. In this part I'll be digging into the implementation and explaining some advanced features of TypeScript along the way.

> TL;DR: Here is the Type Definition. In unpacking it I'll cover the Ternary Operator, [Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html), the `infer` keyword, Recursive Types, [Mapped Types](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html), and [Generic Constraints](https://www.typescriptlang.org/docs/handbook/2/generics.html#generic-constraints).

```ts
export type ForTesting<T> =
    (T extends Subject<infer C> ? Subject<ForTesting<C>> :
        (T extends Observable<infer C> ? Subject<ForTesting<C>> :
            (T extends [unknown?, unknown?, unknown?, unknown?, unknown?] ? { [K in keyof T]: ForTesting<T[K]> } :
                (T extends Array<infer C> ? Array<ForTesting<C>> :
                    (T extends (...args: infer A) => infer R ? SpyInstanceFn<A, R> :
                        (T extends { [K in keyof T]: unknown } ? { [K in keyof T]: ForTesting<T[K]> } :
                            T
                        )
                    )
                )
            )
        )
    );
```

Understanding each of the expressions used to define the `ForTesting` type will give you more tools when defining your own utility types. Good utility types let you give more information to the type system, and maintain strong type support even in sophisticated code structures.

The first thing to notice is that the definition is just a sequence of nested [ternary operators](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Conditional_Operator). That is `(condition ? trueResult : falseResult)`. Here's how it looks in plain js:

```ts
const someValue =
    (firstCondition ? firstResult :
        (secondCondition ? secondResult :
            (thirdCondition ? thirdResult :
                elseFourthResult
            )
        )
    )
```

So the structure of the type definition is simply a big, ugly, ultra-nested, `if, else if, else if, ..., else` block. Given that, we can focus on each expression line by line.


Mapping Non-Built-In Types
==========================

The conditions on each line match against a generic type constraint. Let's unpack the first condition:

```ts
export type ForTesting<T> =
    (T extends Subject<infer C> ?
```

A `Subject` is a [generic](https://www.typescriptlang.org/docs/handbook/2/generics.html) container with the following methods:

 - A `subscribe()` method, to receive new values
 - A `next()` method, to set the current value

Being generic it can contain any type, for example, `Subject<number>`, `Subject<Date>`, `Subject<Car>`. We could add expressions for every type we want to support like so:

```ts
export type ForTesting<T> =
    (T extends Subject<Car> ? Subject<ForTesting<Car>> :
        (T extends Subject<Date> ? Subject<Date> :
            (T extends Subject<number> ? Subject<number> :
                T
            )
        )
    )
```

The problem is there are unlimited possibilities of types that could be contained in a `Subject`. This is why we have the `infer` keyword. It can used to capture a generic type parameter in a variable that can be used in the result expressions.

```ts
export type ForTesting<T> =
    (T extends Subject<infer C> ? Subject<ForTesting<C>> :
```

The type parameter of `Subject` is captured as the variable `C` and used in the result expression. In this case we recursively apply the `ForTesting` type to any contents of `Subject`.

If the type is not a subject, the evaluation proceeds to the second condition which looks very similar.
```ts
export type ForTesting<T> =
    /* ... */
        (T extends Observable<infer C> ? Subject<ForTesting<C>> :
```

In this case we infer the contents `C` again and recursively apply `ForTesting` to it, but we also change the `Observable` type to a `Subject` which is more useful for testing.

An `Observable` is also a container but only has the `subscribe` method. At time of writing I'm noticing that I can remove the first condition because all `Subject` types extend `Observable`, and so will be mapped correctly by this second condition.


Mapping Tuples and Arrays
=========================

The important thing to notice about the conditions is that specific types need to preceed general types. The next pair of conditions demonstrate the importance of this:

```ts
export type ForTesting<T> =
    /* ... */
            (T extends [unknown?, unknown?, unknown?, unknown?, unknown?] ? { [K in keyof T]: ForTesting<T[K]> } :
                (T extends Array<infer C> ? Array<ForTesting<C>> :
```

Consider the tuple `[number, string, Date]`. This can actually be matched by the latter `Array<infer C>` as `Array<number | string | Date>`. In doing so we've lost the positional type information of the tuple. Every item in the array can be any of the three types. In order to avoid this we first need to check for the more specific tuple types, and can use [Mapped Types](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html) to preserve specific positional types.

```ts
export type ForTesting<T> =
    /* ... */
            (T extends [unknown?, unknown?, unknown?, unknown?, unknown?] ? { [K in keyof T]: ForTesting<T[K]> } :
```

The condition is quite simple: Our type `T` must match a tuple of upto 5 `unknown` items (more could be added). `unknown` will match against any type. The result expression `{ [K in keyof T]: ForTesting<T[K]> }` extracts each key as `K` and recursively applies the `ForTesting` type to `T[K]`, the type at position `K` in our tuple `T`. By doing so we preserve the positional type information of the tuple.

If we don't have a tuple, but we do have an array, we simply apply the `ForTesting` to the array contents, much like we did for `Subject` and `Observable`.


Mapping Functions
=================

The next expression in the sequence deals with functions. If you understood the explanation of the `infer` keyword above it should be quite self-explanatory:

```ts
export type ForTesting<T> =
    /* ... */
                    (T extends (...args: infer A) => infer R ? SpyInstanceFn<A, R> :
```

The condition is matching against any function, achieved with arrow function syntax. The types of the arguments is being extracted as `A` and the return value as `R`. All matching functions are replaced with `SpyInstanceFn` which is the [vitest](https://vitest.dev/) spy interface. The `SpyInstanceFn` type accepts type information for the function arguments and return values.


Mapping Objects
===============

The final expression is the most general, matching against any object:

```ts
export type ForTesting<T> =
    /* ... */
                        (T extends { [K in keyof T]: unknown } ? { [K in keyof T]: ForTesting<T[K]> } :
                            T
```

As with the tuples expression above, it uses the mapped types syntax to preserve the relationship between specific keys and the types of their values. It matches any object and applies the `ForTesting` type to its members.

It's important that this comes last as it is the most general type. It would in fact match against every preceeding case. For example in the case of functions it would map them to objects with methods from the `Function` interface (`bind`, `apply`, `call`, etc), but lose the information that it's callable.

The only values it doesn't match against are the primitive types `string`, `number`, etc. We don't need to apply `ForTesting` to those, so we finally reach the recursion base case and simply map them to themselves, the input type `T`.


Final Thoughts
==============

This is very much a living type definition that will evolve with my application, even while writing this very post. Built-in types such as `Date` or `Regexp` would currently be matched by the generic object case and possibly some important type information so I'd need to add cases for those if and when I encounter them.

I don't foresee the `ForTesting` type being a widely useful utility type, but the concepts used in its definition can be applied in many circumstances. They are most useful when working with code that provides the framework for your application.

The type system of TypeScript is very broad and powerful, and half the challenge is knowing what is possible. I think the best way to learn is to extract patterns from your code as utility functions, and ensure they provide and maintain type safety without using `any` types everywhere and requiring consuming code to explicitly provide type information. Sometimes it's not possible, sometimes it just requires discovering another feature of the type system.