---
layout: post
title: Recursively widen an interface for testing with TypeScript, Part One
date: 2022-03-14 20:45:00
---

This two-part post will dissect a custom [Utility Type](https://www.typescriptlang.org/docs/handbook/utility-types.html) that I've been refining over several projects. In doing so I'll touch on the [Interface Segregation Principle](https://en.wikipedia.org/wiki/Interface_segregation_principle) of SOLID design, consider a negative impact it can have on unit testing, and breakdown Recursive Types and Generic Conditional Types in TypeScript.

> TL;DR: Here is the Type Definition I'll be unpacking. It's not intended to be a drop-in solution to any specific problem, but it may be a useful starting point for any recursive type mapping. In part one I'll explore what it does and how it made my unit tests simpler and less coupled to the implementation. In [Part Two](/2022/03/28/recursively-widen-an-interface-for-testing-with-typescript-part-two.html) I'll dig into the TypeScript.

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


What does it do?
================

Put simply, it accepts a type as a parameter, and it will return a new type with the following modifications:

 * Recursively widen any `Observable<T>` to `Subject<T>`
 * Recursively replace any `Function` with a [vitest](https://vitest.dev/) spy of the same interface

For those unfamiliar with [RxJS](https://rxjs.dev/), seeing how `Observable` and `Subject` relate to promise might help understand them. The only important thing to understand for the post going forward is that a `Subject` has a "wider" interface than an `Observable`, meaning it exposes more functions or properties.

|Values|Read-Only|ForTesting|
|:--:|:--------:|:------:|
|One |Promise   |Deferred|
|Many|Observable|Subject |


ForTesting in practice
======================

Here is a partial definition of a `SudokuApp` type as exposed to components and features:

```ts
export type SudokuApp = {
    updates$: Observable<SudokuAppUpdate>
    game$: Observable<SudokuGame>
    startGame(): void
}
```

And here is the actual implementation, considered internal to the module:

```ts
export default class DefaultApp {
    // ...

    constructor() {
        this.updates$ = new Subject();
        this.game$ = new BehaviourSubject();
    }

    startGame() {
        // ...
    }
```

You many notice that the properties are exposed as `Observable` but implemented as types of `Subject`. This is by design, following the aforementioned Interface Segregation Principle. By exposing the narrower interface, I'm being clear with consumers of `SudokuApp` about how I intend for it to be used, and I can be confident that the internal expectations of my `DefaultApp` implementation aren't being violated by some consuming code pushing values in unexpectedly.

However this narrower interface is a little annoying to work with when testing. I have an undo feature that listens to the `updates$` stream. I could use something like [jest-mock-extended](https://github.com/marchaos/jest-mock-extended) to automatically create a mock interface, but the problem with that is best shown in code:

```ts
import { mock } from 'jest-mock-extended';

test('setupUndo() returns a function that undoes most recent update', () => {
    const mockApp = mock<SudokuApp>();
    
    const undo = setupUndo(mockApp, /* ... */);

    const updateHandler = mockApp.updates$.subscribe.mock.calls[0][0];

    updateHandler({ /* SudokuAppUpdate */ });

    undo();

    // expect ...
});
```

In order to get values into the observable I need to access the `updateHandler` from the `updates$.subscribe` spy that was created by the `mock` function. So it's possible, but it doesn't look great, and worse than that it is coupling my test to implementation details. I don't actually care about the specifics of the subscribe calls, just that when `updates$` emits a value, the functionality works as expected.

By using the `ForTesting` type, I can make my `SudokuApp` type test-friendly:

```ts
function createMockSudokuApp(): ForTesting<SudokuApp> {
    return {
        updates$: new Subject(),
        game$: new BehaviourSubject(),
        startGame: vi.fn()
    };
}

test('setupUndo() returns a function that undoes most recent update', () => {
    const mockApp = createMockSudokuApp();

    const undo = setupUndo(mockApp, /* ... */);

    mockApp.updates$.next({ /* SudokuAppUpdate */ });

    undo();
});
```

Now I can just use the wider Subject interface to update observables directly, while maintaining perfect type-safety and editor support. As `ForTesting` is applied recursively, it will also be applied to the `SudokuGame` interface on the `game$` property. I typically extract creating these mock instances to exported factory functions for reuse.

In [Part Two](/2022/03/28/recursively-widen-an-interface-for-testing-with-typescript-part-two.html) I explore the implementation of `ForTesting`, and explain how it uses Recursive Types and Generic Conditional Types in TypeScript.