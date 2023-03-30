---
title: "How to Not Build a React App (Part II)"
date: 2023-03-28T21:23:51-04:00
draft: true
---

[See Part I]({{< ref "how-to-not-build-a-react-app" >}} "About Us")

## The Way

Every non-trivial frontend application needs to deal with state and side effects, (and I'm sure a few other concerns). I'm proposing and experimenting with a particular pattern, but I want to emphasize
the particulars of this pattern are _not_ as important has having *some* established pattern. Even a small team of developers will inevitably choose several different patterns
if there is not guidance on how to approach these concerns. I propose:
1. State in one place: One immutable object that holds all application state. It should be normalized.
1. Pure UI: Single function from the state to the DOM to be rendered. In particular: No hooks, no class components.
1. Effects triggered on state: Instead of triggering a side effect when something happens (an 'action'), trigger them whenever the state requires.

This is not a novel suggestion. It's quite inspired by [Re-frame](https://day8.github.io/re-frame/application-state/#the-benefits) amongst others. We are immediately going to run into some problems:
1.  Some state, such as current url, the window size, the scroll position is going to live on the `window` object, so it won't be part of your state object.
1.  Pure UI means we need to rerun the function on every state change. We will likely hit performance issues.
1.  We will want to break up the single function for the UI into manageable pieces.

Also I haven't actually tried out proposal point 3 above so we'll likely run into something there as well. In any case, this is meant to be a learning experience!

## Redux

The first point is basically the same as "use [Redux](https://redux.js.org/understanding/thinking-in-redux/three-principles)". One core [idea](https://redux.js.org/tutorials/fundamentals/part-3-state-actions-reducers) is that a function of the form `(state, ...data) => newState` (a "reducer") is needed since the state is immutable. In Redux that `...data` is always an `Action`, and there is always one reducer.
This core idea I like, but I question some of the other bits of Redux:
1. Actions and action creators seems like an unnecessary level of indirection: Instead let have separate reducers per "action".
1. State, of course, must affect the view. Redux has the [`useSelector`](https://react-redux.js.org/api/hooks#useselector) hook, or [`connect`](https://react-redux.js.org/api/connect). There's also questions on [where](https://redux.js.org/faq/react-redux#should-i-only-connect-my-top-component-or-can-i-connect-multiple-components-in-my-tree) to do the connections to the state.
1. Everything about [effects](https://redux.js.org/usage/side-effects-approaches).
1. More generally, using a reducer to update immutable data is dead simple and the redux tutorial is somehow _huge_.

The Redux folks seem fairly sure of themselves about keeping [actions](https://redux.js.org/usage/reducing-boilerplate#actions), so we shall see how point 1 plays out.

## Not Redux

Let's start with:
```ts
// We define a reducer as a function of the form (state, ...data) => newState.
// A handler is a function of the form (...data) => void.
// Once an initial state is given we can turn reducers into handlers.
// We can then supply a stream of states to anyone who subscribes.

export const initState = <State>(initialState: State) => {
  const subscribers = new Set<(state: State) => void>();
  let state: State = initialState;
  const pushState = () => {
    subscribers.forEach((sub) => {
      sub(state);
    });
  };

  const createHandler =
    <D extends unknown[]>(reducer: (s: State, ...args: D) => State) =>
    (...data: D) => {
      state = reducer(state, ...data);
      pushState();
    };

  const subscribe = (sub: (state: State) => void): void => {
    subscribers.add(sub);
  };

  const unsubscribe = (sub: (state: State) => void): void => {
    subscribers.delete(sub);
  };
  return {
    createHandler,
    subscribe,
    unsubscribe,
  };
};

```