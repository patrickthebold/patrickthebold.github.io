---
title: "How to Not Build a React App (Part VI)"
subtitle: "Trigger effects based on the state not on events!"
date: 2023-08-20T12:16:09-04:00
draft: false
---

Now we get a chance  to demonstrate the "effects based on state" idea. Our state is a discriminated union of:
```ts
type State = ConnectedState | DisconnectedState | ConnectingState;
```
so the (simplified) logic is: "When we are disconnected, we attempt a Websocket connection". Attempting the connection is the "effect". The key point is we don't do this based on any event, we do it based on the state.

We define:
```ts
type Effect = (state: State) => State
```
We are not going down the pure functional programming route of capturing effects in the type system with [monads]({{< ref "what color is your monad" >}} "Monads"), but it's implied that all side effect will happen inside those functions.
They need to return a new state because we need to track that the effect is in progress, so they don't rerun.

We connect our effects to the state like [so](https://github.com/patrickthebold/mpd-client/pull/1/files#diff-780c9fb210c75680e4c1425fcb05fe5956d2a75cc2058d5b4f328f44e042af64R15-R23):
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
So every effect sees almost every state, as the state changes.
Strictly speaking the order could matter and the position of an `effect` in the `effects` array determines what states the effect sees. I don't anticipate that this will be an issue, but I acknowledge the danger.
## Minor points
  -  If a connection fails we want to wait before we retry. But since effects only run on state changes I force a state "change" every second like so:

```ts
const tick = createHandler((state) => state);
setInterval(tick, 1000);
```
  This actually doesn't work very well and we will fix it when when introduce `coeffects`.

  - Our `Disconnected` state goes through two stages (and there are two effects) when the connection fails: At first, the old websocket is in the state and we remove the event handlers and clear the websocket.
    Next, we create a new connection. [code](https://github.com/patrickthebold/mpd-client/pull/1/files#diff-a9f393297cb1ae3eab8a110bcb9bb34262e01f6b0cbdb63f276c0e793012008aR44-R72)
```ts

export const removeWsListeners: Effect = (state) => {
  if (state.websocketStatus === "disconnected" && state.ws) {
    state.ws.removeEventListener("close", setDisconnected);
    state.ws.removeEventListener("open", setConnected);
    state.ws.removeEventListener("error", setDisconnected);
    state.remove("ws");
  }
  return state;
};

export const ensureConnection: Effect = (state) => {
  if (
    state.websocketStatus === "disconnected" &&
    state.ws === undefined &&
    (state.failureAt === undefined ||
      state.failureAt.getTime() < Date.now() - RECONNECT_MS)
  ) {
    const ws = new WebSocket(WS_URL);
    ws.addEventListener("close", setDisconnected);
    ws.addEventListener("open", setConnected);
    ws.addEventListener("error", setDisconnected);
    return makeConnectingState({
      ...state.toObject(),
      websocketStatus: "connecting",
      ws,
    });
  }
  return state;
};
```

  - Finally I just want to emphasize that the websocket provides several [events](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket#events), and like most events, we attach [handlers](https://github.com/patrickthebold/mpd-client/pull/1/files#diff-a9f393297cb1ae3eab8a110bcb9bb34262e01f6b0cbdb63f276c0e793012008aR23-R42) to them.
  And we handle the actual messages in a future [PR](https://github.com/patrickthebold/mpd-client/pull/4/files)