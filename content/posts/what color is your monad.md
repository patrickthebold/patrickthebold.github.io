---
title: "What Color Is Your Monad"
date: 2022-02-13T00:06:30-05:00
draft: false
---

There's a lovely old blog post [_What Color is Your Function_](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/) (_WCIYF_).
If you haven't read that, please do so instead of reading this one.

Since you are here, you should understand the _colored function_ problem described in that post. I'd like to explore how that problem generalized beyond
async concerns and how it connects to the much bemoaned *monad*.

You don't really need to know what a monad is to start reading this post, but you *should* understand the colored function problem described in the linked post.

## Recap

To recap _WCIYF_: Red functions are the async ones. If a blue (synchronous) function needs
to be updated to call a red one, the blue one needs to be changed to red, which in turn, causes a _proliferation of redness_. Also, the red functions
are more painful to call. I'll mostly be concerned with the _proliferation of redness_, but we'll discuss the pain as well.

The author mentions Promises as a mitigation, but ultimately decides using threads (green or os) is a better solution.

## Thesis

Promises are an example of a monad. There _is_ pain in using them, but I'll
argue it is worth it: you should just bite the bullet and make everything red.
When everything is red, you don't have to think about color, and the problem
goes away. 

I'll give a few alternative definitions of "red function", we will
have roughly the same problem, and the solution will always be some monad.

## The Big Reveal

The big reveal in _WCIYF_ is that "Red functions are asynchronous ones". Consider instead, functions that throw exceptions.
Could we consider those "red"?  If you have a function that does not throw (blue) and you need to edit it to call a function that throws, then your function
will also become one that throws, _unless_ you are able to handle the exception. In practice, there will be times where you can handle the exception and times
that you can't, so the redness really only _proliferates_ when you can't handle the exception. Catching an exception is the analogue waiting on a promise,
and it's much more reasonable to have code that handles exceptions than code that waits on promises. Nevertheless, let's focus on the case where we are _not_ able
to handle the exceptions and so the redness does proliferate like the async case.

Side note: some languages like Java have _checked_ exceptions, in this case you really do need to update you method signatures and explicitly change your blue
function to red by declaring that the exception is thrown. This fits a bit better with the async case since there are actual (mechanical) code changes. 
In the unchecked case, all the pain falls on whoever is getting paged because of new exceptions being thrown at runtime.

## Either

Exceptions separate the happy path from the sad path, the typical from the atypical, the unexceptional from the exceptional. 
They come with their own control flow mechanism as an implementation detail[^exceptions].

As an alternative, many languages or libraries define a type `Either<SomeErrorType, DesiredType>`,
where you can specify that your function returns one of two things. You have to
choose your preferred error type, which doesn't matter for this article, so let's
just say we have chosen "`SomeErrorType`".

_WCIYF_ explains promises as _reifying_ the concept of passing a callback/errorback. Likewise, `Either<SomeErrorType, *>` reifies the concept of typical/atypical outcomes.
In both cases, your function now returns a _value_ that represents the control flow, and _values_ tend to be easier to reason about than behaviors.

## Promises Can Fail

Side note: In every language I've seen, a Promise/Future has a success and failure case built in. So it essentially has `Either` built into it. I _think_
this is because languages that have exceptions often have no way of guaranteeing that an exception _won't_ happen. This feels a lot like languages where anything
_could_ be null. I'd be interested in a language that forces the use of `Optional`, `Either` instead of allowing for `null`, `exceptions`. Maybe Haskell does this?!

## Sugar and Pain

_WCIYF_ sort of admits that promises are less painful than callbacks, but also pooh-poohs them as being a half solution. I just want to point out
that some languages have `for expressions` (e.g. Scala) or `do notation` (e.g. Haskell), which further lessens the pain[^async]. A rough example:
[^async]: javascript, of course, has `async/await` syntax. I'm curious if that is equivalent to for expressions/do notation and could (in theory) be used for monads other than `Promise`. 

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
`asyncGetUser` does. `.traverse` is a a combination of [`map`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map) and [`Promise.all`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all), and it does the only thing that makes sense for this code.

Now instead, suppose we have `syncGetUser` which returns `Either<SomeErrorType, User>`. The code is the same, except the types: `Either<SomeErrorType, T>` replaces `Promise<T>` .
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

I'll start with a concrete example: For some context, suppose you have a single page app with micro-services all making http calls sending json to and from the browser and between services.
It's nice to have a per-request "correlation id" that is included in all the log statements. In order for this to work, the correct id needs to be available
whenever you log something. It also needs to be available whenever you make and http call to another service. So let's say "red functions" are functions that need to know the correlation id. Given that
we need the id for logging and http calls we expect the redness to quickly proliferate almost everywhere. 

The naive[^1] thing to do is to add `correlationId` as a parameter everywhere:
 ```scala
def getUser(userId, correlationId) = {
  log.info("getting user", correlationId)
  ...
}
```

Recall however, that we were able to solve our other problems be returning a value that better represents our needs. So, in this case,
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
by replacing `F<*>` with `Promise<*>`, `Either<SomeErrorType, *>`, or `CorrelationId => *`. (The `*` being the spot where you have to substitute the type you _actually_ care about).

We can even combine things, so `F` could be `CorrelationId => Promise<*>` if `getUser` needed a `CorrelationId` _and_ we wanted it to be async.

### A Note on Generics
Many languages have generic types. Take `Array<Int>`, I'll refer to `Array` as the _outside_ and `Int` as the _inside_. Often a language will
let you make the inside a variable `Array<T>` is an array, but I don't know or care what is inside it. It's less common to be able to put a variable on the _outside_.
`F<Int>` is a bit odd, you don't really have any idea what it is, there's only a suggestion that is has something to do with `Int`. Nevertheless, that is precisely
the thing that changes across our three examples[^hkt].
[^hkt]: Languages that do support that feature would call it "Higher kinded types".

### Generalizing our Example.
In our example we needed a `CorrelationId`, but the same pattern can be used for other things you need, but don't really want to explicitly pass. For example, 
database connections, user session ids, user authorization info. Any time you want something available but don't really want to pass it around.

## One Last Example 

A bug comes in. A number is wrong in the UI. The backend that _my_ team is responsible for really just calls other APIs that other teams are responsible for.
The first thing we'd want to look at
is the request and response we made to these external APIs (after all, we want
to pass the blame as quickly as possible if it's an upstream error). Now, for
various reasons we didn't want to log all the requests and responses.  So,
for every external API call, we returned the full request/response in addition
to whatever we wanted to. At least, in QA environments. 

Consider the following return type:

```scala
def getUser(userId): [DiagnosticData[], User]
```

Here, `DiagnosticData` is the request/response or anything else we want to also return to help diagnose issues. It's an array because we need to allow for multiple things coming back.
Then the json coming back to the browser would always be an object and it would contain one extra field with the `diagnosticData` serialized in the response.

The hope would be that anyone doing QA could copy the json from the dev tools in the browser and any bug ticket would include all of the 
API calls we made. Often that's enough to figure out which team needs to take action.

In this case `F` is `[DiagnosticData[], *]` and it _proliferates_ precisely because we need to bubble up the extra data to the controller.

## The Value of Monads

There's no denying that the blue functions are nicer. The best signature for `getUser` is
```scala
def getUser(userId): User
```
But that is not always practical: You often have other _cross cutting_ concerns: resource usage, error handling, code cleanliness, monitoring.
So if, instead, you return:
```scala
def getUser<F>(userId): F[User]
```
You are giving yourself a lot of flexibility to not exactly return a `User`. We saw different examples where we did not exactly return a `User`. You can even vary `F` based on context, only return extra data outside of production, or be synchronous[^3] in unit tests but async when actually running.

## Are you Serious?

No, not really. I mean, I think it was fun to think about colored functions and monads, but most languages don't support these features that make this practical. Also, it's going to be hard to get your whole team (including future teammates) on board with coding like this. So _don't_ actually do this.


[^exceptions]: I think the control flow aspect mostly makes code a bit less verbose at a fairly large conceptual expense.
[^1]: There are other ways to avoid passing the parameter explicitly, but this is just for the sake of example.

[^2]: It needs to be a monad so we can use the `for` expression.

[^3]: There is an identity mondad `let Id<T> := T`
