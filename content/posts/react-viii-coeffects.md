---
title: "Coeffects and useSyncExternalStore"
subtitle: "How to Not Build a React App (Part VII)"
date: 2023-10-08T12:16:09-04:00
draft: false
---

Welcome! This is part of a somewhat lengthy (and slow) series of posts on building React applications. If you don't want to start back at  [Part I]({{< ref "how-to-not-build-a-react-app" >}} "Part I"), you should know I'm a firmly in the "Hooks are evil" camp, and React should be limited to just rendering your application state, which should be managed _outside_ of React. In fact, We are on part eight of the series and haven't even installed React yet.

This project is largely inspired by [Reframe](https://day8.github.io/re-frame/)[^reframe], and a big idea there is that we have a single [state](https://github.com/patrickthebold/mpd-client/blob/6f3248dcae9076a15051f2151d0cd0d34bea6e65/src/state.ts#L46) object that we are managing. 

[^reframe]: I've never _actually_ used reframe but I find their documentation inspiring.

However, as I said earlier, there's going to be some state like the current time, the current URL, the window size, and so forth that we won't keep in our state object. I have been tempted to try and sync that (external) state with some part of our state object: For example, React Router [recommends](https://v5.reactrouter.com/web/guides/deep-redux-integration) not to attempt to sync the URL with a Redux store. I think trying to sync _anything_ is a bit of a smell. 

Furthermore, I'd argue that an `input` DOM element is not much different. React [encourages](https://react.dev/learn/reacting-to-input-with-state#step-5-connect-the-event-handlers-to-set-state) you to try and sync the value in every input element with some state (The state is "`answer`" in the example linked).

So we have this general problem of wanting to put all of the state in a single object, but need to handle "external" state where we don't have any control over. The best we can hope for is to notice when it changes and act appropriately. Reframe introduces the term [coeffects](https://day8.github.io/re-frame/Coeffects/), and React 18 introduced a hook called [useSyncExternalStore](https://react.dev/reference/react/useSyncExternalStore)

## Our approach

Recall that we&mdash;more or less&mdash;have a stream of `State`s to which we can subscribe a `Consumer<State>`, which will render our application and do other [things]({{< ref "react-vi-effects" >}}). 

We have a bug where we need the current time, but don't want to sync the current time into our `State`. So we need to allow for `Consumer<[State, Date]>`.

We already have a concept of a [`ConsumerTransformer`](https://github.com/patrickthebold/mpd-client/pull/2/files#diff-c54113cf61ec99691748a3890bfbeb00e10efb3f0a76f03a0fd9ec49072e410a):
```ts
export type Consumer<T> = (t: T) => void;
export type ConsumerTransformer<T, S = T> = (consumer: Consumer<T>) => Consumer<S>;
```
First we need a straight forward [refactoring](https://github.com/patrickthebold/mpd-client/pull/5) that allows consumers to have multiple parameters, and consumer transformers can add in additional parameters. The code get's a bit busy but ends up as:
```ts
export type Consumer<T extends unknown[]> = (...t: T) => void;
export type ConsumerTransformer<
  T extends unknown[],
  S extends unknown[] = T
> = (consumer: Consumer<T>) => Consumer<S>;
```
## Coeffects from a Producer.

Since the current time is easily available as `new Date()`, getting the time isn't really the problem, the problem is triggering out consumers when the time changes. Specifically, we decided to trigger effects based off of the state, and we want to reconnect a failed websocket connection after a few seconds, so we'd like to produce the current state and time at least once a second. (side note: now that I think about it, I am syncing the websocket state with my application state, I think I can refactor that out to be another coeffect.)

In any case it seems there might be a general pattern where we can easily produce a value: `Producer<T> = () => T`, and want to get a `ConsumerTransformer` that removes `T` as a parameter from the consumer. "Remove" might seem backwards, but if you have something that can produce a `T` you can transform something that needs `X` and `T` into something that just needs `X`. 
The code is [here](https://github.com/patrickthebold/mpd-client/pull/6/files).
```ts
export const makeCoeffect = <T extends unknown[], S>(
  producer: Producer<S>
): {
  transformer: ConsumerTransformer<[...T, S], T>;
  trigger: Callback;
} => {
  const consumers: Array<Consumer<[...T, S]>> = [];
  let currentT: T;
  const trigger: Callback = () => {
    consumers.forEach((c) => {
      c(...currentT, producer());
    });
  };
  const transformer: ConsumerTransformer<[...T, S], T> = (consumer) => {
    consumers.push(consumer);
    return (...t: T) => {
      consumer(...t, producer());
    };
  };
  return {
    transformer,
    trigger,
  };
};
```

Note that we need to hold on to all of the consumers so we can supply a value whenever the `trigger` is called.

We use this with the current time like [so](https://github.com/patrickthebold/mpd-client/pull/7/files#diff-bb5f0f23dee8124ba0edbf16dc48e91a8b4c9b16713a8702940d004f98937430)

```ts
export const time = <T extends unknown[]>(
  millis: number
): ConsumerTransformer<[...T, Date], T> => {
  const { transformer, trigger } = makeCoeffect<T, Date>(() => new Date());
  setInterval(trigger, millis);
  return transformer;
};
```

## Expanding to handlers/reducers.

We also have a spot where the current time is needed in the handlers/reducers[^reducer], so we introducer an analogous concept of `ReducerTransformer`, the handlers do not need an extra trigger but it still seems to make since to create the `ReducerTransformer` along with something that makes the trigger and `ConsumerTransformer`.

[^reducer]: We pass a reducer to a `createHandler` to make a handler.

There's a bit of refactoring [here](https://github.com/patrickthebold/mpd-client/pull/8/files) including some rework of the generics of `makeConfigurable`, but it's the main idea is captured above.