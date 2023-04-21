---
title: "How to Not Build a React App (Part II)"
subtitle: "State Management"
date: 2023-03-28T21:23:51-04:00
draft: false
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
    sub(state);
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
[3f56358](https://github.com/patrickthebold/mpd-client/blob/3f56358a5c1888f953802422cd51476e31c35aeb/src/state.ts)

This enforces that only a "handler" can update the state, and ensures that every subscriber sees all state changes (since the time of the subscription).
[Note: we won't use `unsubscribe` or have more than one subscriber so there's a bit of over-engineering here.]

To compare to Redux, we'll write a separate reducer for every action, and the handler that we get back from `createHandler` is a combination of the action creator and `dispatch`. One clear downside, 
of this approach is that with Redux the actions are serializable. Actions are basically reified handlers.

## Immutable State

[immer.js](https://immerjs.github.io/immer/) seems like the more popular choice, but I'm tempted to go with [immutable.js](https://immutable-js.com/). I've only used immutable.js with javascript, and I'm hopeful
it will be nicer to use in typescript. Things I like about immutable.js:
1. More rich methods built in: `groupBy`, `flatMap`, `partition`, etc. You could use [lodash](https://lodash.com/) with immer.js, to get something similar.
1. `Map` and `Set` have a better definition of equals than the javascript versions. (Sets don't allow duplicates according to the definition of 'equals'.)
1. 'Deep' methods. e.g. [`updateIn`](https://immutable-js.com/docs/v4.3.0/Map/#updateIn()).

Things I don't like amount immutable.js:
1. Accessing members is done with `.get('key')` instead of `.key`. (This seems not to be true for [`Records`](https://immutable-js.com/docs/v4.3.0/Record/).)
1. No canonical way to serialize to Json. (Last I checked there were some small libraries, but I'd expect it to be built into the project.)

I'm going to start with immutable.js, but be prepared to back out if I get too annoyed with it.
```shell
$ npm i immutable
```

## Our State

To make some progress on our MPD client we need a think on our state and what user actions we will expose in the UI.
Normally I'd go back and forth between the state and the view, but I'm going to see how much work I can do on just the state and write the view later. If only to emphasize the separation.

First we will explore the

### MPD API
Things will be easier if there's not too much of a mismatch between our models and theirs. Let's explore the MPD API directly to understand how it works. I'll  `telnet` directly with mpd rather than our websocket bridge. 

#### idle
First we notice there is an [`idle`](https://mpd.readthedocs.io/en/latest/protocol.html#querying-mpd-s-status) command. Let's listen while I add a song an play a track.


```
$ telnet $MPD_HOST 6600
Trying 192.168.1.7...
Connected to 192.168.1.7.
Escape character is '^]'.
OK MPD 0.23.5
> idle
changed: playlist
OK
> idle
changed: player
changed: mixer
OK

```
Note that it leaves the `idle` state once it reports an event. In the above I sent the `idle` command twice. (My input I prefixed with a `>` so you can see what I sent, but that was not actually sent.)

It seems the intended use is to always stay in the `idle` state unless you need to send a command, where then you'd send `noidle` to leave the idle state.

#### playlistinfo
```
> playlistinfo                                                              
file: Pink Floyd/Animals/01 Pigs on the Wing, Part 1.mp3                    
Last-Modified: 2019-01-04T04:51:49Z                                     
Format: 44100:24:2                                                                                 
Artist: Pink Floyd                                                     
AlbumArtist: Pink Floyd                                                                   
ArtistSort: Pink Floyd                                                          
Title: Pigs on the Wing, Part 1                                    
Album: Animals                                                      
Track: 1                                                                                                                                                                                                             
Date: 2000-04-25                                              
OriginalDate: 1977-01-23                                       
Genre: Rock                                                                   
Composer: Roger Waters                                                            
Disc: 1                                                         
Label: Capitol Records                                                                                                                                                                                               
AlbumArtistSort: Pink Floyd                                 
MUSICBRAINZ_ALBUMID: 1ae20bd2-e90a-3e20-b50a-9319691be8e3         
MUSICBRAINZ_ARTISTID: 83d91898-7763-47d7-b03b-b92132375c47    
MUSICBRAINZ_ALBUMARTISTID: 83d91898-7763-47d7-b03b-b92132375c47           
MUSICBRAINZ_RELEASETRACKID: f58d2312-7283-3cdd-82bf-8d7732383d1c         
MUSICBRAINZ_TRACKID: aca2620e-eee7-416c-bb3b-b881b7d68780                                      
Time: 85                                                             
duration: 85.420                                                          
Pos: 0                                                                           
Id: 1                                                                        
file: Pink Floyd/Animals/02 Dogs.mp3                                            
Last-Modified: 2019-01-04T04:51:51Z                             
Format: 44100:24:2                                             
Artist: Pink Floyd                                              
AlbumArtist: Pink Floyd                                   
ArtistSort: Pink Floyd                                         
Title: Dogs                                                                                                                                                                                                          
Album: Animals                                                  
Track: 2                                                        
Date: 2000-04-25                                                
OriginalDate: 1977-01-23                                        
Genre: Rock                                                     
Composer: Roger Waters, David Gilmour                                                                                                                                                                                
Disc: 1                                                                                               
Label: Capitol Records                                                                                                                                                                                               
AlbumArtistSort: Pink Floyd                                                                                                                                                                                          
MUSICBRAINZ_ALBUMID: 1ae20bd2-e90a-3e20-b50a-9319691be8e3                                                                                                                                                            
MUSICBRAINZ_ARTISTID: 83d91898-7763-47d7-b03b-b92132375c47            
MUSICBRAINZ_ALBUMARTISTID: 83d91898-7763-47d7-b03b-b92132375c47                                                                                                                                                      
MUSICBRAINZ_RELEASETRACKID: 69ee84e6-c9e9-3deb-b0db-06c65b9db488                                                                                                                                                     
MUSICBRAINZ_TRACKID: 143d01f4-a464-45e8-9143-50bfea4abf9a                    
Time: 1028                                                                 
duration: 1028.204                                                                                    
Pos: 1                                                                             
Id: 2                                           
```

`playlistinfo` gives the current queue. Notice there does not seem to be a delimited between tracks and rather a new track seems to start with a `file` attribute. `file` seems like the best way to globally identify a track.
There is also an `Id` field sent this is an id of a playlist entry, which is different from a track id since the same track can be in the playlist twice. `Id` is recommended over `Pos` to interact with the queue, and you can see
many commands come in pairs with `<command>id` being the newer replacement commands.

#### status and currentsong
These give some info about the current state.
```
> status
volume: 100
repeat: 0
random: 0
single: 0
consume: 0
partition: default
playlist: 3
playlistlength: 62
mixrampdb: 0
state: pause
song: 27
songid: 28
time: 229:262
elapsed: 229.139
bitrate: 192
duration: 261.616
audio: 44100:24:2
nextsong: 28
nextsongid: 29
OK
> currentsong
file: R.E.M_/Automatic for the People/06 Sweetness Follows.mp3
Last-Modified: 2019-01-04T05:00:16Z
Format: 44100:24:2
Artist: R.E.M.
AlbumArtist: R.E.M.
ArtistSort: R.E.M.
Title: Sweetness Follows
Album: Automatic for the People
Track: 6
Date: 1992-10-06
OriginalDate: 1992-10-06
Disc: 1
Label: Warner Bros. Records
AlbumArtistSort: R.E.M.
MUSICBRAINZ_ALBUMID: 3ebe514d-d197-3984-81ee-4d09231be775
MUSICBRAINZ_ARTISTID: ea4dfa26-f633-4da6-a52a-f49ea4897b58
MUSICBRAINZ_ALBUMARTISTID: ea4dfa26-f633-4da6-a52a-f49ea4897b58
MUSICBRAINZ_RELEASETRACKID: 79e4dabb-2ba3-3256-a142-831a20660c5b
MUSICBRAINZ_TRACKID: 594a6de2-1aa5-4624-bda9-3fe455d1d2e2
Time: 262
duration: 261.616
Pos: 27
Id: 28
OK
```

#### search
```
> search "(any == 'pink')"
file: P!nk/Raise Your Glass/01 Raise Your Glass.mp3
Last-Modified: 2019-01-04T04:51:37Z
Format: 44100:24:2
Artist: P!nk
AlbumArtist: P!nk
ArtistSort: Pink
Title: Raise Your Glass
Album: Raise Your Glass
Track: 1
Date: 2010-10-06
OriginalDate: 2010-10-06
Genre: Pop
Composer: P!nk
Disc: 1
Label: LaFace Records
AlbumArtistSort: Pink
MUSICBRAINZ_ALBUMID: 54d6d31e-19a8-46fb-b9c6-f2632f2ad473
MUSICBRAINZ_ARTISTID: f4d5cc07-3bc9-4836-9b15-88a08359bc63
MUSICBRAINZ_ALBUMARTISTID: f4d5cc07-3bc9-4836-9b15-88a08359bc63
MUSICBRAINZ_RELEASETRACKID: e183fa67-001e-38b2-8dad-e7b0da1c32f1
MUSICBRAINZ_TRACKID: 5ba0505e-e2b4-4f09-9796-a3750282f093
Time: 203
duration: 203.441
OK
```
There's lots of search options, specifically using `contains` and `starts_with` instead of `==`, and searching for specific tags.

#### Wrap up

That should give the basics for building an app, there's lots more commands available in the API, but it we understand the above we can find anything else we need
when we need it. Next post we will do some up front design and figure out what MPD commands go with the features we want to implement.