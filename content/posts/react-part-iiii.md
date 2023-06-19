---
title: "How to Not Build a React App (Part IIII)"
subtitle: "(Player) Handlers and Yak Shaving"
date: 2023-06-17T20:15:58-04:00
draft: false
---

[Part I]({{< ref "how-to-not-build-a-react-app" >}} "Part I")

## Orientation

In the [previous post]({{< ref "react_part_iii" >}} "Part II") I said there would be three main pages:

1. The player.
1. The queue.
1. The library. 

We only got to talking about the state for the player. This post should be simpler: I'm only going to implement the handler's needed. Recall, our "handlers" are roughly an action builder and a reducer, for those familiar with Redux. They handle events in the browser, and update the state.

Next post, I'll deal with the websocket, then the view for the player. I hope to get the player working before I move onto the queue and the library.

## Some Renamings

From a [domain](https://en.wikipedia.org/wiki/Domain-driven_design) perspective I'm going to [rename](https://github.com/patrickthebold/mpd-client/commit/4893338fa0f7e04f2143c422f3d7da625ae56379) `MpdCommand` to `UserIntent`. What that is capturing is what the user want's to do, and it's an implementation detail that they happen to map onto mpd commands. I'm avoiding the term "Action" since Redux uses that for something similar. I also hopefully made things more clear by using the term ["player"](https://github.com/patrickthebold/mpd-client/commit/8891f67720778a189c216bc229a9b7e3967a38e2).


## Connection States.

The way I set up the [state management](https://github.com/patrickthebold/mpd-client/blob/ac0ac08a61947190beb238274233869401c839a6/src/state-management.ts) I need an initial state before I can create a handler. I may go back and change this later, but I imagine I want some states involving the websocket connection. Something like "Disconnected", "Connecting", "Connected". Where the state we talked about in the last post is really only for when we are connected. Recall we have a queue for pending user intents, I think I will always have this queue available even when we are disconnected.
```ts
export type State = ConnectedState | DisconnectedState | ConnectingState;

export type ConnectedState = {
    websocket: 'connected';
    player: PlayerStatus;
    sentIntents: UserIntent[];
    pendingIntents: UserIntent[];
    responseData: string;
}

export type DisconnectedState = {
    websocket: 'disconnected';
    pendingIntents: UserIntent[];
}
export type ConnectingState = {
    websocket: 'connecting';
    pendingIntents: UserIntent[];
}
```
[24deb60](https://github.com/patrickthebold/mpd-client/blob/24deb6020005e7021150e87bd034864a448458c0/src/state.ts)

## Immutable

I also said I would try out Immutable.js. Those [changes](https://github.com/patrickthebold/mpd-client/commit/26115f490bad2b254fb491505e3cf991172b634d) are straightforward, but I will note that I am using records for parts of the state that are expected to change over time, and `Readonly` objects elsewhere.

## The handlers

Apologies for number of higher order functions, but there's a clear pattern where we want a handler that takes some data, produces a `UserIntent` and pushes that on the pending intents queue. This is:
```ts
type Handler<T extends unknown[]> = (...t: T) => void;
export const makeIntentHandler = <T extends unknown[]>(makeIntent: (...args: T) => UserIntent): Handler<T> => createHandler((s, ...args) => {
    const intent = makeIntent(...args)
    // We need the switch to make typescript happy
    switch (s.websocket) {
        case "connected": return s.update('pendingIntents', intents => intents.push(intent));
        case "disconnected": return s.update('pendingIntents', intents => intents.push(intent));
        case "connecting":return s.update('pendingIntents', intents => intents.push(intent));
    }
})
```

We can then make handlers for all the events we expect to handle from the UI:

```ts
export const pause = makeIntentHandler(() => ({type: 'pause'}));
export const stop = makeIntentHandler(() => ({type: 'stop'}))
export const nextTrack = makeIntentHandler(() => ({type: 'next_track'}))
export const previousTrack = makeIntentHandler(() => ({type: 'previous_track'}))
export const setVolume = makeIntentHandler((volume: number) => ({type: 'set_volume', volume}))
```
[3405d0a](https://github.com/patrickthebold/mpd-client/commit/3405d0acaa701710f4f5a01ba0604e6b0eae6a43)