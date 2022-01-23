---
title: "What Color Is Your Monad"
date: 2022-01-17T10:06:30-05:00
draft: true
---

There's a lovely old blog post [_What Color is Your Function_](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/) (_WCIYF_).
If you haven't read that, please do so instead of reading this one.

I'd like to give a very rough "pro monad" argument in the same vein as that post.

You don't really need to know what a monad is for this post, but you should understand the colored function problem described in the linked post.

## Recap

To recap _WCIYF_: Red functions are the async ones, they are more painful to call. If a blue (synchronous) function needs
to be updated to call a red one, the blue one needs to be changed to red, which in turn, causes a _proliferation of redness_.

The author mentions Promises as a mitigation, but ultimately decides using threads (green or os) is a better solution.

## Thesis

Promises are an example of a monad. There _is_ pain in using them, but I'll arguing it is worth it; you should just bite the bullet and make
everything red. I'll give a few alternative proposals for "red function" where we have roughly the same problem, and the solution will be
different monads.

## The Big Reveal

The big reveal in _WCIYF_ is that "Red functions are asynchronous ones". Consider instead, functions that throw exceptions.
Could we consider those "red"?  If you have a function that does not throw (blue) and you need to edit it to call a function that throws, then your function
will also become one that throws, _unless_ you are able to handle the exception. In practice, there will be times where you can handle the exception and times
that you can't, so the redness really only _proliferates_ when you can't handle the exception. Catching an exception is the analogue waiting on a promise,
but it's much more reasonable to have code that handles exceptions than code that waits on promises. Nevertheless, let's focus on the case where we are _not_ able
to handle the exceptions and so the redness does proliferate like the async case.

Side note: some languages like Java have _checked_ exceptions, in this case you really do need to update you method signatures and explicitly change your blue
function to red by declaring that the exception is thrown. This fits a bit better with the async case since there are actual (mechanical) code changes. 
In the unchecked case, all the pain comes when exceptions start coming
from places they didn't use to and you need to hunt down why your application doesn't work.

## Either

Exceptions are both a control flow mechanism and a way of separating the typical from the atypical. Let's ignore
the control flow aspect of exceptionsi[^exceptions]. 

This leads us to needing a way to separate out the typical and atypical return values. Many languages have some way of returning
`Either<SomeErrorType, DesiredType>`, where you can specify that your function returns one of two things. You have to choose your preferred error type;
it doesn't matter for this article, so let's just say we have chosen "`SomeErrorType`".

Taking a function that takes a callback/errorback and changing it to a function that returns a promise, is quite similar to take a function that throws and 
exception to a function that returns `Either<SomeErrorType, *>`. In both cases, we decide it's better to return a _value_ that better represents our needs.

## Promises Can Fail

Side note: In every language I've seen, a Promise/Future has a success and failure case built in. So it sort of has `Either` built into it. I _think_
this is because languages that have exceptions often have no way of guaranteeing that an exception _won't_ happen. This feels a lot like languages where anything
_could_ be null. I'd be interested in a language that forces the use of `Optional`, `Either` instead of allowing for `null`, `exceptions`.

## Sugar

_WCIYF_ sort of admits promises are less painful than callbacks, but also pooh-poohs them as being a half solution. I just want to point out
that some languages have `for expressions` (e.g. Scala) or `do notation` (e.g. Haskell), which helps close the gap between blue and red a bit more. As a rough example:

```scala
def getFriends(userId) = 
  for {
    user <- asyncGetUser(userId)
    friends <- user.friendIds.traverse(asyncGetUser)
  } yield friends
```

That's some made up code to asynchronously get a user from their id and then get all their friends by id. Allow me to repeat the code with the
types specified (typescript style) after all the values:
```scala
def getFriends(userId): Promise<User[]> = 
  for {
    user: User <- asyncGetUser(userId): Promise<User>
    friends: User[] <- user.friendIds.traverse(asyncGetUser): Promise<User[]>
  } yield friends
```
Notice how on the left of the `<-` we always have `T` whenever the right is `Promise<T>`. Also notice the _proliferatation_ of the `Promise`, `getFriends` returns a `Promise` because
`asyncGetUser` does. Also `.traverse` is a basically a combination of [`map`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map) and [`Promise.all`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)

Now instead, suppose we have `syncGetUser` which returns `Either<SomeErrorType, User>`. The code is the same except the types, where `Promise<T>` is replaced with `Either<SomeErrorType, T>`.
```scala
def getFriends(userId): Either<SomeErrorType, User[]> = 
  for {
    user: User <- syncGetUser(userId): Either<SomeErrorType, User>
    friends: User[] <- user.friendIds.traverse(asyncGetUser): Either<SomeErrorType, User[]>
  } yield friends
```


Compare that with the synchronous (blue) case where `getUser` returns `User`.
```scala
def getFriends(userId) = {
  user = getUser(userId)
  user.friendIds.map(getUser)
}
```


## Another Example

I'll start with a concrete example: For some context, suppose you have a single page app with micro-services all making http calls sending json to and from the browser and the different services.
It's nice to have a per-request "correlation id" that is included in all the log statements. In order for this to work, the correct id needs to be available
whenever you log something. It also needs to be available whenever you make and http call to another service. So let's say "red functions" are functions that need to know the correlation id. Given that
we need the id for logging and http calls we expect the redness to quickly proliferate almost everywhere. 
```scala
def getUser(userId, correlationId) = {
  log.info("getting user", correlationId)
  ...
}
```

The naive[^1] thing to do is to add `correlationId` as a parameter everywhere. Recall however, that we were able to solve our other problems be returning a value that better represents our needs. So, in this case,
we need to return a value that represents "the need for a correlation id". For this, we can use a function, i.e.
```scala
CorrelationId => T
```
With this change the signature of `getUser` is:
```scala
def getUser(userId): CorrelationId => User
```

We can repeat the same example above without any changes:
```scala
def getFriends(userId): CorrelationId => User[] = 
  for {
    user: User <- syncGetUser(userId): CorrelationId => User
    friends: User[] <- user.friendIds.traverse(asyncGetUser): CorrelationId => User[]
  } yield friends
```

Note that this code doesn't deal with the correlation Id. (It also didn't deal with asynchronicity, or errors). 

### A Pattern Emerges?
Now that we've written essentially the same code three times, let's pull out the different part as a variable:
```scala
def getFriends<F: Monad>(userId): F<User[]> = 
  for {
    user: User <- syncGetUser(userId): F<User>
    friends: User[] <- user.friendIds.traverse(asyncGetUser): F<User[]>
  } yield friends
```
Here my method is parameterized by `F` with the restriction that `F` is a monad[^2] (`F: Monad`). You can get all the previous examples
by replacing `F<*>` with `Promise<*>`, `Either<SomeErrorType, *>`, or `CorrelationId => *`. (The `*` being the spot where you have to substitute).

We can even combine things, so `F` could be `CorrelationId => Promise<*>` if `getUser` needed a `CorrelationId` _and_ we wanted it to be async.

### A Note on Generics
Many languages have generic types. You often see `Array<Int>`. I'll refer to `Array` as the _outside_ and `Int` as the _inside_. Often a language will
let you make the inside a variable `Array<T>` is an array, but I don't care what is inside it. It's less common to be able to put a variable on the _outside_.
`F<Int>` is a bit odd, you don't really have any idea what it is, there's only a suggestion that is has something to do with `Int`. Nevertheless, that is precisely
the thing that changes across our three examples.

### Generalizing our Example.
In our example we needed a `CorrelationId`, but the same pattern can be used for other things you need, but don't really want to explicitly pass. For example, 
database connections, user session ids, user authorization info. 

## One Last Example
Again we will be concrete, with the same context. Often the actual data we showed on the page was from some external apis provided by another team at some other company. Say, during QA some
number is wrong on the page. The first thing we'd want to look at is the request and response we made to these APIs. For various reasons we didn't want to log all the requests and responses.
As a solution, every time we called an external API we returned the full request/response in addition to whatever we wanted to.

Sticking with our example we have:
```scala
def getUser(userId): [DiagnosticData[], User]
```

Here, `DiagnosticData` is the request/response or anything else we want to also return to help diagnose issues. It's an array because we need to allow for multiple things coming back.
Then the json coming back to the browser would always be an object and it would contain one extra field with the `diagnosticData` serialized in the response.

This would only be turned on in QA environments. The hope would be that anyone doing QA could copy the json from the dev tools in the browser and any bug ticket would include all of the 
API calls we made. Hopefully, we wouldn't even need to look at the code, but instead could determine quickly if either the request or the response was not behaving as intended.

I won't repeat myself too much, but in this case `F` is `[DiagnosticData[], *]` and it _proliferates_ precisely because we need to bubble up the extra data to the controller.

## The Value of Monads

There's no denying that the blue functions are nicer. The best signature for `getUser` is
```scala
def getUser(userId): User
```
But, if that is your signature you are committing to always being able to immediately return a `User` with no errors and no other parameters.
If, instead, you return:
```scala
def getUser<F>(userId): F[User]
```
You are giving yourself a lot of flexibility to not literally return a `User`. By varying `F` you can return a `User` at some point in the future, return an error instead, require additional parameters
or return a `User` and some other data. You can even vary `F` based on context, only return extra data outside of production, or be synchronous[^3] error in unit tests but async when actually running.

## Are you Serious?

No, not really. I mean, I think I think it was fun to think about colored functions and monads, but most languages don't support these kind of features that it's practical to write code like this. And even if
you are, it's going to be hard to get a whole team (including future teammates) on board with coding like this, so _don't_ actually do this.


[^exceptions]: I think the control flow aspect mostly makes code a bit less verbose at a fairly large conceptual expense.
[^1]: There are other ways to avoid passing the parameter explicitly, but this is just for the sake of example.

[^2]: It needs to be a monad so we can use the `for` expression.

[^3]: There is an identity mondad `let Id<T> := T`
