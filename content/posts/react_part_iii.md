---
title: "How to Not Build a React App (Part III)"
subtitle: "Design"
date: 2023-05-22T14:40:27-04:00
draft: true
---

[Part I]({{< ref "how-to-not-build-a-react-app" >}} "Part I")

The design will be personal, as this app has a target audience of one. I see three main pages:

1. The current track.
1. The queue.
1. The library. 

## The Current Track

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

