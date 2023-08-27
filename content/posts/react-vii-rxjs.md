---
title: "How to Not Build a React App (Part VII)"
subtitle: "Wherein it feels like we are starting to reimplement RxJS"
date: 2023-08-25T12:16:09-04:00
draft: false
---

Back to: [Part I]({{< ref "how-to-not-build-a-react-app" >}} "Part I")

## Motivation

There's a couple of (perhaps minor) things we can clean up from the [PR on effects](https://github.com/patrickthebold/mpd-client/pull/1).
Take a closer [look](https://github.com/patrickthebold/mpd-client/pull/1/files#diff-780c9fb210c75680e4c1425fcb05fe5956d2a75cc2058d5b4f328f44e042af64) at how effects are triggered:
```ts
subscribe((state) => {
  let newState = state;
  for (const effect of effects) {
    newState = effect(newState);
  }
  if (newState !== state) {
    setState(newState);
  }
});
```
The logic for most effects is: "Is the UI in a state where something needs to happen? If so, do it and update the state so we know the effect is in progress." Most of the time they should do nothing and return the same state.
The condition at the end (`if (newState !== state)`) is a bit awkward. If you think of `subscribe` as providing a stream of `State`s we never will want to provide the exact same state twice. It seems nicer (to me anyway) to somehow
modify `subscribe` so that it won't produce the same value twice in a row. The code we end up with with look more like:
```ts
subscribe
    .with(skipDuplicates())
    ((state => {...}))
```

The other issue is a bit more subtle. When we call `setState`, that new state re-enters the code above, and all the effects run again. On one hand, this is expected as that block of code should get every state change. However, there is some danger of the stack overflowing because those effects run synchronously.
I'd like to have `subscribe` produce at most one value 'per tick'. (This will also be useful when we have a UI to render: We don't really want to render state changes that happen faster than the browser can render the UI.)

## Comparison with RxJS

[RxJS](https://rxjs.dev/) is (was?) a popular library for reactive programming. Their [`observer`](https://rxjs.dev/guide/observer) is similar to our [`Consumer`](https://github.com/patrickthebold/mpd-client/pull/1/files#diff-c54113cf61ec99691748a3890bfbeb00e10efb3f0a76f03a0fd9ec49072e410aR1).
```ts
type Consumer<T> = (t: T) => void;
```
`observer` has three methods: `next`, `error`, and `complete`. 
```ts
const observer = {
  next: (state: State) => console.log('Observer got a next value: ' + state),
  error: err => console.error('Observer got an error: ' + err),
  complete: () => console.log('Observer got a complete notification'),
};
```

`next` is precisely `Consumer<State>`, and if we were using RxJS that's the method that would get the stream of all the states. We don't need `complete` because our stream of `State`s is infinite; it continually produces values as long as the UI exists.
We might have errors but those will be modeled _inside_ of our `State`.  For example, our `DisconnectedState` optionally has a [time of failure](https://github.com/patrickthebold/mpd-client/pull/1/files#diff-f205dae8a870fe4c43305ee8ddd660e51710768dcbf9998ce57e9f7dea83f9c2R60). Finally, since we just have the `next` method there's no need to even have a method and we just use `Consumer<State>` directly.

### Operators

RxJS has pipeable [operators](https://rxjs.dev/guide/operators), and it looks like we end up in more or less the same place. But back to our project:

## ConsumerTransformers

This is perhaps a subtle point: Above, I said I wanted to transform `subscribe`, and we pass a `Consumer<State>` to `subscribe` as an argument. 

We will instead begin by [transforming](https://github.com/patrickthebold/mpd-client/pull/2) `Consumer`s:

```ts
type ConsumerTransformer<T, S = T> = (
  mapper: Consumer<T>
) => Consumer<S>;
```

Once we have that type it's easy to implement the transformations I need:
```ts
const skipDuplicates =
  <T>(): ConsumerTransformer<T> =>
  (consumer) => {
    let lastValue: T;
    return (t: T) => {
      if (t !== lastValue) {
        lastValue = t;
        consumer(t);
      }
    };
  };
```
Note that `skipDuplicates` is a function that returns a `ConsumerTransformer<T>`, this is just to deal with the generic `<T>`. (You can't have a value that is generic).

I split the next transformation in two: We don't want to produce value "too fast", for the effects I'm going to use [`queueMicrotask`](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide), but for the rendering of the UI I will use [`requestAnimationFrame`](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame).
I'll be honest that I don't quite understand the intricacies of these two functions, but it seems like a good place to start.

```ts
type Callback = () => void;
type Scheduler = Consumer<Callback>;

const makeThrottler =
  <T>(scheduleWork: Scheduler): ConsumerTransformer<T> =>
  (consumer) => {
    let workScheduled = false;
    let latestValue: T;
    return (t: T) => {
      latestValue = t;
      if (!workScheduled) {
        scheduleWork(() => {
          consumer(latestValue);
          workScheduled = false;
        });
      }
    };
  };

export const perTick = <T>(): ConsumerTransformer<T> =>
  makeThrottler(queueMicrotask);
export const perAnimationFrame = <T>(): ConsumerTransformer<T> =>
  makeThrottler(requestAnimationFrame);
```

## Futzing with `subscribe`

Now we come back to the subtle point: I'd rather transform `subscribe` directly than transform the "consumers". Of course the code above transforms consumers.
It took me (at least) 3 tries to get this right. 

### Ugly Attempt 1
This is in the [PR](https://github.com/patrickthebold/mpd-client/pull/2/files#r1271669216). The code looks like:
```ts
const subscribeEffects = pull(
  compose(skipDuplicates<State>(), perTick<State>())
)(subscribe);

subscribeEffects((state) => {
  setState(effects.reduce((newState, effect) => effect(newState), state));
});
```

On one hand, `pull` and `compose` are fairly natural things you'd want to do with functions. Most people are familiar with function [composition](https://en.wikipedia.org/wiki/Function_composition), [pullbacks](https://en.wikipedia.org/wiki/Pullback#Precomposition) are probably less so.
On the other hand, the code is quite ugly:
```ts
export const pull =
<A, B>(f: (a: A) => B) =>
  <C>(g: (b: B) => C) =>
  (a: A) =>
    g(f(a));
export const compose =
  <A, B, C>(g: (b: B) => C, f: (a: A) => B) =>
  (a: A) =>
    g(f(a));
```

That said, the idea behind a pullback (or "precomposition" if you prefer), is important for the problem at hand. Anytime you know how to transform the argument to a function, you can instead transform the function itself. This is precisely what we want.

### Better Attempt 2

Javascript allows functions to have properties. We basically move `pull` from a stand alone function to a property we can attach to `subscribe`. [PR](https://github.com/patrickthebold/mpd-client/pull/3/files)

First we define a `ConfigurableFunction`:
```ts
type ConfigurableFunction<A, B> = {
  with: <C>(transformer: (c: C) => A) => ConfigurableFunction<C, B>;
  (a: A): B;
};
```

I'm not sure that's the best name, but `with` precomposes `this` with the `transformer` function passed in.

The way it's used looks better:
```ts
const subscribeEffects = subscribe.with(skipDuplicates()).with(perTick());
```

We, of course, need to add the `with` property to our functions, and also remember to attach it to the function it returns (so we can chain multiple `.with` calls):
```ts
export const makeConfigurable = <A, B>(
  f: (a: A) => B
): ConfigurableFunction<A, B> => Object.assign(f, { with: withProperty });

function withProperty<A, B, C>(
  this: (a: A) => B,
  transformer: (c: C) => A
): ConfigurableFunction<C, B> {
  return makeConfigurable((c: C) => this(transformer(c)));
}
```

Attempt 3 won't be much different; I need to futz with where the generics show up. We will see that in the next article on coeffects.
