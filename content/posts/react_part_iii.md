---
title: "How to Not Build a React App (Part III)"
subtitle: "Design"
date: 2023-05-22T14:40:27-04:00
draft: false
---

Back to: [Part I]({{< ref "how-to-not-build-a-react-app" >}} "Part I")

The design will be personal, as this app has a target audience of one. I see three main pages:

1. The player.
1. The queue.
1. The library. 

In this post I will focus on the Player, and furthermore, modeling the state needed for the player.

## The Player

Here we need information about the current track:
* Title
* Artist
* Album
* Track number
* Track position
* Track length

As well as some basic controls:
* Play/Pause/Stop
* Previous/Next Track
* Volume

Album art would be good, but I need to think about text vs [binary](https://github.com/websockets/ws/issues/2085#issuecomment-1266926356) data coming over the websocket, and test out how MPD deals with album art for my library.
Refer to [Part II]({{< ref "react_part_ii" >}} "Part II") for the commands `currentsong` which gives all the info for the first part; `status` has information about the volume and if the track is paused. [Controlling Playback](https://mpd.readthedocs.io/en/latest/protocol.html#controlling-playback)
has `pause`, `stop`, `next`, `previous`. While [Playback Options](https://mpd.readthedocs.io/en/latest/protocol.html#playback-options) has `getvol` and `setvol`.

### State I: The view state.

We mostly [repeat](https://github.com/patrickthebold/mpd-client/blob/ac0ac08a61947190beb238274233869401c839a6/src/state.ts) the above in typescript, but I will note that the track position is special because it only applies to the current song. In light of that, I group that value with the general status of the player.
It should be clear that this is enough data to render the component described above

```ts
export type Song = {
    file: SongId;
    title?: string;
    artist?: string;
    album?: string;
    trackNo?: string;
    duration?: number;
}

export type PlayStatus = {
    state: 'pause' | 'stop' | 'play';
    volume: number;
    trackPosition: number;
    currentTrack: SongId
}
```

### State II: Side Effects and Server Communication.

#### Background Tangent
Our app of course will need to talk to the mpd server. Right now I plan on doing things differently than most popular frameworks. I think I have a good motivation, but we shall see if it works out well. First, let's discuss some of the existing options.

[Redux Saga](https://redux-saga.js.org/) has this example:
```ts
class UserComponent extends React.Component {
  ...
  onSomeButtonClicked() {
    const { userId, dispatch } = this.props
    dispatch({type: 'USER_FETCH_REQUESTED', payload: {userId}})
  }
  ...
}
```

Then there is some redux [middleware](https://redux.js.org/understanding/history-and-design/middleware) that (more or less) will do the side effects based on the actions dispatched. (By side effect I mean something like making an http request, or sending a message on a websocket.)

There's also [TanStack Query](https://tanstack.com/query/latest/docs/react/overview) which has this example:
```ts
function Example() {
  const { isLoading, error, data } = useQuery({
    queryKey: ['repoData'],
    queryFn: () =>
      fetch('https://api.github.com/repos/tannerlinsley/react-query').then(
        (res) => res.json(),
      ),
  })

  if (isLoading) return 'Loading...'

  if (error) return 'An error has occurred: ' + error.message

  return (
    <div>
      <h1>{data.name}</h1>
      <p>{data.description}</p>
      <strong>👀 {data.subscribers_count}</strong>{' '}
      <strong>✨ {data.stargazers_count}</strong>{' '}
      <strong>🍴 {data.forks_count}</strong>
    </div>
  )
}
``` 
Here, there is a hook that does the side effect directly in the component.

#### The Gist of Our Approach

We will do something more in line with Redux Saga, but instead of the side effects being driven off of the actions the side effects will happen based on the _state_.
Let's do a quick review of our [state management](https://github.com/patrickthebold/mpd-client/blob/ac0ac08a61947190beb238274233869401c839a6/src/state-management.ts) and how it relates to Redux. If you follow that link you see we have a `createHandler`. A handler will take the place of both an action creator and a reducer. We won't dispatch actions, but instead invoke the handlers directly. (The arguments to the handler will be the `payload` that is typically part of a Redux action.) Instead of "action" I'll use the term "MpdCommand" for the side effects we need to send to the server. We will store the commands in our state and there will be some analogue of middleware that watches the state and performs side effects based on the state.

"Handlers" are for events in the browser and the effect will be to update the state object, "commands" or "MpdCommands" represent messages we need to send (or have sent) over the websocket, which ultimately ends up at the Mpd server.

I'll give a detailed example soon, but first let me explain why I want to take this approach: If you look at the [protocol](https://mpd.readthedocs.io/en/latest/protocol.html#requests) there is no documentation on sending a request while one is in progress. There is a [command list](https://mpd.readthedocs.io/en/latest/protocol.html#command-lists) option, but you need to know all the commands up front. I did not experiment with what happens when you send a request while one is in progress, and frankly because it is not documented I'd rather not try. This means I cannot always immediately send a command: Say the user adds a bunch of songs to the queue and while that is happening they press "pause", I have to wait until the song additions finish before sending the "pause" command. I need to hold a queue of pending commands somewhere. Since I've already decided in keeping one large object with all of the state that seems like the place to put it. Thus, the user clicking "pause" in the browser will be a browser event, it will be handled by a "handler" which will update the state by adding the "pause" `MpdCommand` to a queue in the state. Something else will watch the state and actually send the messages on the websocket at the appropriate time.

I also find this approach aesthetically appealing. I'm promoting this idea of having all[^most] state in a single object with ui/dom a function of the state. Every time the state changes our function runs and produces the desired state. This makes our programming _declarative_, and React handles the actual side effects needed to bring the browser's dom in-line with what we declared. I can have another function that depends on the state that computes what messages need to be sent on the websocket. The rough parallel is that updating the browsers DOM and sending messages on the websocket are both side effects. There's no "React for websocket events" nor "React for http calls", so things will be a bit different, but that's kind of what I am aiming for.

[^most]: most, because probably can't get things like the url or window size, in your state.

#### The Finer Points of State

Consider the state in part I above. For example, if the song is paused, and what the position in the track is. There is the true state on the server. We can query the true state and know what we were last told, we can also infer what the state must be from our commands and the responses. For example, suppose based on our last query the state is "playing" and we send the "pause" command and receive a successful response, "OK", we can then infer the current state to be "paused". Also note that we will need two queues of commands, one for commands[^bulk] in progress on the server, and another for commands we need to send. We will need some kind of reducer to determine the new state based on a successful command. In any case, it should be clear that it's impossible to know the true state of the server, but based on querying and successful commands we can track, "what we believe the server state to be". This is what we will store in our state object. Since the two command queues are also in our state object we can compute (in our declarative view function) what the state will be if the in-progress commands complete successfully as well as what the new state will be if all of the outstanding commands are sent and complete successfully. I imagine this will all be quite quick, but it's worth thinking about the best UX for the differences between those three states. 

Imagine a song is playing, or at least we believe it is playing based on the state we are tracking. The user clicks pause, but we have not sent the message yet. We could show the pause symbol in the UI, but blinking slowly. Then once we send the command, we could optimistically show it as paused (no blinking), but maybe have a progress line or spinner somewhere. Then once it completes successfully we go back to rendering the state as we best believe it to be.

[^bulk]: This is plural because MPD supports [command lists](https://mpd.readthedocs.io/en/latest/protocol.html#command-lists)

#### Code

[Explicitly](https://github.com/patrickthebold/mpd-client/blob/6aabd920ee7a24e106f07aa9ff9ea3a1985601be/src/state.ts)
```ts
export type State = {
    status: PlayStatus;
    sentCommands: MpdCommand[];
    desiredCommands: MpdCommand[];
    responseData: string;
}

// This is not literally what get's sent on the websocket, just a representation of
// something we need to tell the server.
export type MpdCommand = {type: 'pause'} | {type: 'stop'} | {type: 'next_track'} | {type: 'previous_track'} | {type: 'set_volume', volume: number}
```
You see the status which is what we believe to be true, and the two queues. Finally there is the `responseData` we need to track the server's reply to the `sentCommands`. Since we will have something watching the state, it will be able to see if `responseData` ends in "OK" to know that the sent commands are finished (and do error handling). But we haven't gotten there quite yet.

Next: [Part IIII (Handlers)]({{< ref "react-part-iiii" >}} "Part IIII")
