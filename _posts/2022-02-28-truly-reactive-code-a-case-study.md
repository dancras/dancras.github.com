---
layout: post
title: Truly Reactive Code, A Case Study
date: 2022-02-28 19:30:00
---

The problem:

 - A `Messages` component for communicating with users.
 - User can dismiss the current message.
 - New message replaces the current message.
 - An option to require that a message must be dismissed.

This post is not about the problem. The code uses RxJS, but it's not about RxJS either. I'll be comparing my initial solution, which should be familiar to anyone who understands `reduce`, with a final solution that showcases the conciseness of reactive programming.

> TL;DR: Here's part of the final code that I'll be explaining. It meets all the requirements. If it seems intriguing, read on. Reactive programming is fun and a good way to explore new ways to solve problems without learning a whole new ecosystem.

{% highlight js %}
// Note: Variable names with $ suffix are Observable streams
const currentMessage$ = newMessage$.pipe(
    map(data => takeAUntilBThenC({
        a: data,
        b$: data.mustDismiss ? dismiss$ : merge(dismiss$, hot(newMessage$)),
        c: undefined
    })),
    concatAll(),
    startWith(undefined)
);
{% endhighlight %}


The Initial Solution
====================

I've been working with Observables and reactive streams for ~8 years now and even so I didn't arrive at the above solution immediately. Reaching for a more imperative solution is just so familiar and comfortable, but there was a specific moment that made me zoom out and consider the direction I was heading.

So my initial idea was that this could be solved with `reduce` or its similar equivalent `scan` in RxJS. I would convert my input streams into actions, and reduce those actions into a list of messages and the current message would be the first in the list.

{% highlight js %}
const updates$ = merge(
    messages$.pipe(map(data => ({ action: 'new', data }))),
    dismiss$.pipe(map(() => ({ action: 'dismiss' })))
);
{% endhighlight %}

Simple enough. Now we have an `updates$` stream of `new` actions and `dismiss` actions.

{% highlight js %}
$pendingMessages = updates$.pipe(
    scan((messages, update) => {
        if (update.action === 'dismiss') {
            // Handle dismiss action
        } else {
            // Handle new action
        }
    }, []),
);
{% endhighlight %}

I started off thinking it was trivial; simply slice off the first message for `dismiss`, and handle a couple of cases for `new` depending on the value of a `mustDismiss` property on the first message.

{% highlight js %}
if (update.action === 'dismiss') {
    return messages.slice(1);
} else {
    if (messages[0]?.mustDismiss) {
        return messages.concat([update.data]);
    } else {
        return [update.data];
    }
}
{% endhighlight %}

This worked for simple scenarios, but as my test cases expanded to include scenarios with sequences of messages, some with and without `mustDismiss`, it became clear that I would need more logic to prune the messages:

{% highlight js %}
if (update.action === 'dismiss') {
    let i = messages.slice(1).findIndex(message => message.mustDismiss);
    i = i === -1 ? Math.max(1, messages.length - 1) : i + 1;
    return messages.slice(i);
}
{% endhighlight %}

Suddenly it's starting to look "ugly" to me. I'm looking for the next message with `mustDismiss`, or skipping to the last message in the list if there isn't one. I'm sure I could extract a function and express this more clearly, but the imperative processing of messages and all the index juggling sent me exploring for an alternative.

I'd still call this "reactive" code, and I'd still call it "declarative" code, at least from the perspective of the `$pendingMessages` stream. Just some of the implementation details slip into imperative land and so it falls short of the "truly reactive" designation.


The Reactive Solution
=====================

The thing that had bugged me about the initial solution was revisiting messages that had already been processed. Aside from the "ugly" code, to understand how a message would behave I'd need to understand all the possible actions of the reducer, as any of them could affect my message. In this case it was just two, but it's not hard to imagine how this scales.

I wanted a solution where the behaviour of a message was declared upfront alongside it. The question to ask was "what is the behaviour of each message?"

{% highlight js %}
const message$ = newMessage$.pipe(
    map(message => takeAUntilBThenC({
        a: message,
        b: message.mustDismiss ? dismiss$ : dismissOrNewMessage$,
        c: undefined
    }))
);
{% endhighlight %}

I couldn't have said it better myself. A message is itself, until something happens, at which point it's discarded (becomes `undefined`). Messages with `mustDismiss` are only discarded by a `dismiss$` update, but the rest by either a `dismiss$` or a `newMessage$` update.

So now we have a Higher-Order Observable, that is, a stream of streams. The inner streams are our messages and their behaviour. They emit the message, and at some point later they emit undefined, and then they complete.

So how do we consolidate this stream of streams into our `currentMessage$` stream? We need to understand the [concatAll operator](https://rxjs.dev/api/operators/concatAll).

{% highlight js %}
const currentMessage$ = message$.pipe(
    concatAll(),
    startWith(undefined)
);
{% endhighlight %}

Without going into too much detail, the `concatAll` operator will ensure that all values from inner streams emit in order, and moves on to the next inner stream when the previous completes.

This flow diagram visualises the journey of messages with `mustDismiss`.

![Timeline of concatAll operator](/media/2022-02-27/concat-messages.png)

The final detail is a subtle one that takes us back to the behaviour of each message. The issue is with the "until B" part in particular:

{% highlight js %}
takeAUntilBThenC({
    a: message,
    b: message.mustDismiss ? dismiss$ : dismissOrNewMessage$,
    c: undefined
})
{% endhighlight %}

If we queue up a bunch of messages that discard themselves when `dismiss$` emits, then a single dismiss will empty the whole queue, rather than just the first item. Fortunately this isn't how `concatAll` works. It will only set up subscriptions for the first inner stream, until that completes and it moves on to the next. All done then...

Except that for `dismissOrNewMessage$`, or rather the `OrNewMessage$` part, we want the opposite behaviour. A new message should discard any previous message even when it isn't currently active; we just want to skip it. To achieve this we need to keep that stream "hot", keep its subscriptions active.

{% highlight js %}
const dismissOrNewMessage$ = merge(dismiss$, hot(newMessage$));

function hot(source$) {
    const replay$ = source$.pipe(
        first(),
        shareReplay(1)
    );

    const sub = replay$.subscribe(() => sub.unsubscribe());
    return $replay;
}
{% endhighlight %}

Our `hot` function configures our stream to immediately replay any previous value, and subscribes to that stream until it has a previous value to replay. That way, when `concatAll` switches to an inner stream that should be discarded by a later message, `dismissOrNewMessage$` will have a value waiting to emit immediately, discarding it.

We'll also take a quick look at the `takeAUntilBThenC` function so we've covered everything.

{% highlight js %}
function takeAUntilBThenC({ a, b$, c }) {
    return concat(NEVER.pipe(startWith(a), takeUntil(b$)), of(c));
}
{% endhighlight %}

`concat` works just like `concatAll` but outside of a `pipe` chain. In this case we are `concat`ing two streams: `NEVER.pipe(startWith(a), takeUntil(b$))` and `of(c)`.

The latter will simply emit `c`. As it is part of a `concat` it will do so when the former completes.

`NEVER` is a stream that never emits and never completes. We use `startWith(a)` to have it emit `a` immediately, and `takeUntil(b$)` to force it to complete when `b$` emits. So emit `a` until `b$`, then switch to the stream containing only `c`.

Here's the complete code in TypeScript as committed:

{% highlight ts %}
export type MessageData = {
    text: string[],
    mustDismiss?: boolean
}

export function createMessagesModel() {
    const newMessage$ = new Subject<MessageData>();
    const dismiss$ = new Subject<void>();

    const currentMessage$ = newMessage$.pipe(
        map(data => takeAUntilBThenC({
            a: data,
            b$: data.mustDismiss ? dismiss$ : merge(dismiss$, hot(newMessage$)),
            c: undefined
        })),
        concatAll(),
        startWith(undefined),
        shareReplay(1),
        prewarm()
    );

    return {
        newMessage$,
        currentMessage$,
        dismiss$
    };
}

function takeAUntilBThenC<T>(a: T, b$: Observable<unknown>, c: T) {
    return concat(NEVER.pipe(startWith(a), takeUntil(b$)), of(c));
}

function hot<T>(source$: Observable<T>) {
    return source$.pipe(
        first(),
        shareReplay(1),
        prewarm()
    );
}

function prewarm<T>() {
    return (source$: Observable<T>) => {
        const sub = source$.pipe(observeOn(asyncScheduler)).subscribe(() => sub.unsubscribe());
        return source$;
    };
}
{% endhighlight %}


Final Thoughts
==============

Finding the reactive solution was very satisfying; no loops, no indexes, and code that really shouts what it does, particularly for those familiar with reactive terminology. It wasn't quite the smooth ride presented here though. RxJS makes precise control over observable streams possible, but in doing so has quite a learning curve.

Figuring out which operator to use, even the subtle differences like `map(), concatAll()` vs `concatMap()`, then which streams should be hot or cold, which streams need replays. `tap()` debugging to figure out where the communication breakdown is. A relatively trivial feature became cognitively quite expensive.

I'd highly recommend for any programmer to explore reactive programming. It presents a big challenge to the way most of us learn to code. Solutions can be quite unusual and come together like some kind of puzzle game.