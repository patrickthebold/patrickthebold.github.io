---
title: "What Color Is Your Monad"
date: 2021-12-25T14:06:30-05:00
draft: true
---

There's a lovely old blog post [What Color is Your Function](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/).
If you haven't read that please do so instead of reading this one.

I'd like to give a very rough "pro monad" argument in the same vein as that post.

You don't really need to know what a monad is for this post, but you should understand the colored function problem described in the linked post.

## Recap

Let's recap _What Color is Your Function_. Red functions are the async ones, they are more painful to call. If a blue (synchronous) function needs
to be updated to call a red one, the blue one essentially needs to be changed to red, which in turn, causes a proliferation of 'redness'.

The author mentions Promises (aka Futures) as a mitigation, but ultimately decides using threads (green or os) is better.

## My Point

Promises are an example of a monad, but there are other examples. There _is_ pain in using them, but I'd argue it is worth it.

## The Big Reveal

The big reveal in _What Color is Your Function_ declares that "Red functions are asynchronous ones". Let's instead, say "Red functions are ones
that sometimes throw exceptions that you don't know how to handle". I'm being a bit wiggly with the "don't know how to handle" part, but let's think 
about this and see how well it fits.

First, the _proliferation_ of redness. If you have a new blue function that does not throw and requirements change and you need to call a red
function that _does_ throw, your blue function automatically becomes red. This is in part because I said you don't know how to handle the exception. 
In practice, there will be some spots in your code where you can handle the exception and other's where you can't, but please allow me some wiggle room.

Note, some languages like Java have _checked_ exceptions, in this case you really do need to update you method signatures and explicitly change your blue
function red by declaring that the exception is thrown. This fits a bit better with the async case since there are actual (mechanical) code changes. 
In the unchecked case you don't really need to change the code, so there's no pain in the coding part. All the pain comes when exceptions start coming
from places they didn't use to and you need to hunt down why your application doesn't work.

## Either

Exceptions are both a control flow mechanism and a way of separating out typical and atypical things that happen during a function call. Let's ignore
the control flow aspect of exceptions. (Personally, I think it mostly makes code a bit less verbose at a fairly large conceptual expense, but I suppose
it's a matter of taste.) This leads us to needing a way to separate out the typical and atypical return values. Many languages have some way of returning
`Either[SomeErrorType, DesiredType]`, where you can specify that your function returns one of two things. You have to choose your preferred `SomeErrorType`.
This could be the existing type your language has for exceptions or whatever else you want. It doesn't matter so much for this article, so let's just say 
we have chosen `SomeErrorType`.

Taking a function that takes a callback/errorback and changing it to a function that returns a promise, is quite similar to take a function that throws and 
execption to a function that returns `Either<SomeErrorType, *>`. In both cases, the return type is a value that represents reality—either an async operation, or 
the fact that there might be an error. 

## Promises Can Fail

This is a side note: In every language I've seen, a Promise/Future has a success and failure case built in. So it sort of has `Either` built into it. I _think_
this is because languages that have exceptions often have no way of guaranteeing that an exception _won't_ happen. This feels a lot like languages where anything
_could_ be null. I'd be interested in a language that forces the use of `Optional`, `Either` instead of allowing for `null`, `exceptions`.

## Sugar

_What Color is Your Function_ sort of admits promises are less painful than callbacks, but also pooh-poohs them as being a half solution. I just want to point out
that some languages have `for expressions` (Scala) or `do notation` (Haskell), which helps close the gap between blue and red a bit more. As a rough example:

```
def getFriends(userId) = 
  for {
    user <- asyncGetUser(userId)
    friends <- user.friendIds.traverse(asyncGetUser)
  } yield friends
```

That's some made up code to asynchronously get a user from their id and then get all their friends by id. Allow me to repeat the code with the
types specified (typescript style) after all the values:
```
def getFriends(userId): Promise<User[]> = 
  for {
    user: User <- asyncGetUser(userId): Promise<User>
    friends: User[] <- user.friendIds.traverse(asyncGetUser): Promise<User[]>
  } yield friends
```
Notice how on the left of the `<-` we always have `T` whenever the right is `Promise<T>`. Also notice the _proliferatation_ of the `Promise`: `getFriends` returns a `Promise` because
`asyncGetUser` does. Also `.traverse` is a basically a combination of [`map`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map) and [`Promise.all`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)

Now instead, suppose we have `syncGetUser` which returns `Either<SomeErrorType, User>`. The code is the same except the types, where `Promise<T>` is replaced with `Either<SomeErrorType, T>`.
```
def getFriends(userId): Either<SomeErrorType, User[]> = 
  for {
    user: User <- syncGetUser(userId): Either<SomeErrorType, User>
    friends: User[] <- user.friendIds.traverse(asyncGetUser): Either<SomeErrorType, User[]>
  } yield friends
```


In any case, this doesn't look too bad compared to the blue case:
```
def getFriends(userId) = {
  user = getUser(userId)
  user.friendIds.map(getUser)
}
```


## More Monads



