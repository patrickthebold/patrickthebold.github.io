---
title: "How to Not Build a React App"
date: 2023-03-23T20:23:36-04:00
draft: true
---

I've been meaning to write a post on "How I wish people made React apps" for a while now. Today I got annoyed by some HN [discussion](https://news.ycombinator.com/item?id=35270877) on React hooks.
So that pushed me over the edge and figured I'd write something. This will be a multi-part series and part of the fun is discovering where we will end up. I've really only been involved in one production 
React app, so I'll solicit feedback as we go along. Oh, and sharpen you razors&mdash;we will be shaving a [yak](https://projects.csail.mit.edu/gsb/old-archive/gsb-archive/gsb2000-02-11.html).

My introduction to React through the Clojurescript eco-system. [Quiescent](https://github.com/levand/quiescent#rationale)[^dumdom], for example, does very little. One might say it is just the 'V' in 'MVC'. Others
might say that is lets you write a pure function from your state to "HTML/DOM" (`ReactElement`s really). Of course, you still need to figure out how to do all the other things that your app needs to do, but this separation is quite nice:
All of your features can be implemented by modeling your state, figuring out how to update it, and (separately) what DOM you want for the given state.
[^dumdom]: See [Dumdom](https://github.com/cjohansen/dumdom) for a modern replacement.

All of this sounds quite abstract so my primary goal is to build a real application. It just so happens that I've been unhappy with all the [MPD](https://code.visualstudio.com/Docs/languages/markdown) clients I can find. So let's make one of those!
A secondary goal will be a general primer on building apps. My plan is to be complete but terse. My tertiary goal is to use as few libraries as possible&mdash;to reinvent the wheel so to speak&mdash;in hopes that we can learn something about design.

## MPD
I've a Raspberry Pi (1B I think) with [arch linux](https://bbs.archlinux.org/viewtopic.php?id=259639) hooked up to my stereo. I use MPD to play music (mp3s mostly) from a networked disk. First thing is to skim over their [API](https://mpd.readthedocs.io/en/latest/protocol.html).
Now, I was hoping we could do everything in the web-browser but MPD's API is over TCP. One option would be to run a node.js server on the raspberry pi that translates a websocket connection from the browser to the TCP connection to the MPD. Something like ![Architecture](/architecture.svg)

First I need to figure out the latest version of node.js that is running on the pi[^arch] 
```shell
[patrick@pi ~]$ node
Welcome to Node.js v17.3.0.
Type ".help" for more information.
> 
```
[^arch]: Arch linux [stopped supporting](https://archlinuxarm.org/forum/viewtopic.php?f=3&t=15721) ARMv6 in Feb. 2022. I promised yak shaving but I'm not reinstalling my os.

## MPD Bridge
First we set up a basic node project.
```shell
$ mkdir mpd-bridge
$ cd mpd-bridge/
$ git init
$ npm init
<Answer the questions>
```
(Follow along! [0e4f4c6](https://github.com/patrickthebold/mpd-bridge/tree/0e4f4c6df2978999e3716ebce1413bcfd9f3f144))

There are several websocket libraries but [ws](https://github.com/websockets/ws) seems popular.
```shell
$ npm i ws
```

At this point I want to test out the basic [example](https://github.com/websockets/ws#simple-server) ([a3966b0](https://github.com/patrickthebold/mpd-bridge/blob/a3966b070417c53f02a9c5ab83736c303f335a47/index.mjs)), but I need a client to connect. So let's also set up the...

## Frontend

Let's give [vite](https://vitejs.dev/guide/) a try, since that seems the simplest way to get a dev server to proxy to our node backend.
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

The last command will start the dev server, you can navigate to [http://localhost:5173/](http://localhost:5173/), and test out the client [example](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket#examples) in the browser's dev-tools console.

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

Interestingly, I don't seem to need to do anything with CORS even though the node server is on a different port.