---
title: "How to Not Build a React App (Part I)"
date: 2023-03-28T20:23:36-04:00
draft: false
---

I've been meaning to write a post on "How I wish people made React apps" for a while now. Today I got annoyed by some HN [discussion](https://news.ycombinator.com/item?id=35270877) on React hooks.
So that pushed me over the edge and figured I'd write something. This will be a multi-part series and part of the fun is discovering where we will end up. I've really only been involved in one production 
React app, so I'll solicit feedback as we go along.

My introduction to React through the Clojurescript eco-system. Please take a minute to review [Quiescent](https://github.com/levand/quiescent#rationale)[^dumdom]. I think you will see it is quite simple. One might say it is just the 'V' in 'MVC'. Others
might say that is lets you write a single (pure) function from your state to "HTML/DOM" (`ReactElement`s really). 

Of course, you still need to figure out how to do all the other things that your app needs to do, but this separation is quite nice:
All of your features can be implemented by modeling your state, figuring out how to update it, and then (separately) figuring out what DOM you want for any given state.
[^dumdom]: See [Dumdom](https://github.com/cjohansen/dumdom) for a modern replacement.

I'd like to explore this idea more by building a real application. It just so happens that I've been unhappy with all the [MPD](https://code.visualstudio.com/Docs/languages/markdown) clients I can find. So let's make one of those!

A secondary goal will be a general primer on building (web) apps. My plan is to be complete but terse. A tertiary goal is to use as few libraries as possible&mdash;to reinvent the wheel, so to speak&mdash;in hopes that we can learn more about building applications.

## MPD
I've a Raspberry Pi (1B I think) with [arch linux](https://bbs.archlinux.org/viewtopic.php?id=259639) hooked up to my stereo. I use MPD to play music (mp3s mostly) from a networked disk. You can skim over their [API](https://mpd.readthedocs.io/en/latest/protocol.html).
I was hoping we could do everything in the web-browser, but MPD's API is over (raw) TCP. One option is to run a node.js server on the raspberry pi that bridges a websocket connection from the browser to a TCP connection to the MPD. Something like:

![Architecture](/architecture.svg)

## MPD Bridge
First we set up a basic node project for the backend.
```shell
$ mkdir mpd-bridge
$ cd mpd-bridge/
$ git init
$ npm init
<Answer the questions>
```
(Follow along! [0e4f4c6](https://github.com/patrickthebold/mpd-bridge/blob/0e4f4c6df2978999e3716ebce1413bcfd9f3f144/package.json))

There are several websocket libraries; [ws](https://github.com/websockets/ws) seems popular.
```shell
$ npm i ws
```

Next  I'd like to test out the basic [example](https://github.com/websockets/ws#simple-server) ([a3966b0](https://github.com/patrickthebold/mpd-bridge/blob/a3966b070417c53f02a9c5ab83736c303f335a47/index.mjs)), but I need a client for that. So let's also set up the...

## Frontend

There are several tools to bootstrap a new frontend project. Let's give [vite](https://vitejs.dev/guide/) a try.
```shell
$ npm create vite@latest 
✔ Project name: … mpd-client
✔ Select a framework: › Vanilla
✔ Select a variant: › TypeScript

Scaffolding project in /home/patrick/code/mpd-client...

Done. Now run:

  cd mpd-client
  npm install
  npm run dev

```
(and do as it says [b6d166b](https://github.com/patrickthebold/mpd-client/tree/b6d166b050008dd51c37203887a9b22d16fb90ce))

The last command will start the dev server, you can navigate to [http://localhost:5173/](http://localhost:5173/), and test out this client [example](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket#examples) in the browser's dev-tools console.

```js
// Create WebSocket connection.
const socket = new WebSocket("ws://localhost:8080");

// Connection opened
socket.addEventListener("open", (event) => {
  socket.send("Hello Server!");
});

// Listen for messages
socket.addEventListener("message", (event) => {
  console.log("Message from server ", event.data);
});

```
If you have started the node server example `$ node index.mjs` you'll see logs in both the backend and frontend consoles.
Note that we don't need to do anything with CORS since we are using a [websocket](https://blog.securityevaluators.com/websockets-not-bound-by-cors-does-this-mean-2e7819374acc).

## Bridging the sockets

I'm hoping to keep the backend app small so the frontend can serve as a larger example. After consulting the docs for the [websocket](https://github.com/websockets/ws#use-the-nodejs-streams-api) and the [tcp connection](https://nodejs.org/docs/latest-v17.x/api/net.html#netconnectpath-connectlistener),
I ended up with [cf45eee](https://github.com/patrickthebold/mpd-bridge/blob/cf45eee7a5221276fca7bd37f79b8c8d3d36f275/index.mjs)
```js
import { WebSocketServer, createWebSocketStream } from 'ws';
import * as net from 'net';

// We Allow configuration through environmental variables.
// If a MPD_UNIX_SOCKET is set it will override the host and port combination.
const MPD_HOST = process.env.MPD_HOST ?? 'localhost';
const MPD_PORT = parseInt(process.env.MPD_PORT ?? '6600');
const MPD_UNIX_SOCKET = process.env.MPD_UNIX_SOCKET;


const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws) => {
  const webSocket = createWebSocketStream(ws, { encoding: 'utf8' });

  webSocket.on('error', console.error);
  const mpdSocket = MPD_UNIX_SOCKET ? 
    net.connect(MPD_UNIX_SOCKET) :  
    net.connect(MPD_PORT, MPD_HOST);

  webSocket.pipe(mpdSocket);
  mpdSocket.pipe(webSocket);
});
```
I happen to run my raspberry pi on a static ip so I can do 

```shell
$ MPD_HOST='192.168.1.7' node index.mjs
``` 
to start the backend.
As before, we can test out the connection from the browser's console:

```js
const socket = new WebSocket("ws://localhost:8080");

socket.addEventListener("message", (event) => {
    event.data.text().then(console.log)
});
socket.addEventListener("open", (event) => {
  socket.send('status\n')
});
------
> OK MPD 0.23.5
------
> volume: 100
> repeat: 0
> random: 0
> single: 0
> consume: 0
> playlist: 1
> playlistlength: 39
> xfade: 0
> state: pause
> song: 29
> songid: 1
> nextsong: 30
> nextsongid: 2
> time: 1:257
> elapsed: 1.855
> bitrate: 0
> OK
```

Note that `data` is a [`Blob`](https://developer.mozilla.org/en-US/docs/Web/API/Blob/text) which is why we have to call `.text()`. Also it's a bit unclear to me how things are buffered, or how the data will be split up across websocket messages. It would be nice if `OK` was always at the end of a websocket message, but I'm not sure we can count on that.

Next: [Part II (State Management)]({{< ref "react_part_ii" >}} "Part II")
