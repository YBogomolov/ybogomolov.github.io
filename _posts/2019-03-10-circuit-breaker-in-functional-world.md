---
layout: post
title:  "Typesafe Frontend Development"
author: yuriy
categories: [ fp, typescript, circuit breaker, architecture ]
image: assets/images/5.jpg
---

In this article  I would like to talk a bit about one of the most commonly used patterns in enterprise architecture — Circuit Breaker — and its implementation using purely functional approach.

<!--more-->

> N.B. The creation of the described package was inspired by the [Glue.CircuitBreaker](https://hackage.haskell.org/package/glue-core-0.4.2/docs/Glue-CircuitBreaker.html) module for Haskell. The module itself is available here: [github://circuit-breaker-monad](https://github.com/YBogomolov/circuit-breaker-monad).

# What is a Circuit Breaker?

## Context and problem definition

Your application is communicating with a third-party service over some kind of protocol. It could be an HTTP API, a socket API, or maybe something else, but it’s major characteristics are these:
- You don’t control the network’s reliability, so packets may be dropped at random.
- You don’t control the service’s reliability, so it may fail at an unexpected time.
- Service errors and network failures are temporary. The service and the environment self-recover themselves after a period of time.
- If the service doesn’t respond after a certain timeout, the request is considered as failed.

## Solution

**Circuit Breaker** is a design pattern which resembles a simple [finite state machine](https://en.wikipedia.org/wiki/Finite-state_machine) with the following states:

1. **Closed** — requests are transported directly to the service. If the request is failed, the internal error counter is increased. If the amount of errors over a certain time span is greater than the given threshold, the Circuit Breaker’s state is changed to **Open**, and the internal timer is started — Circuit Breaker gives an external service some time to recover. When the timer goes off, the state is changed to **Half-open**.
2. **Open** — requests are immediately finished with error.
3. **Half-open** — in this state the first request (I will call it a canary request from now on) is allowed to pass to the service. If a canary request succeeds, the error is considered to be solved and the circuit goes to the Closed state again.

This is quite simple description of a Circuit Breaker pattern. You may want to read [a blog post by Martin Fowler](https://martinfowler.com/bliki/CircuitBreaker.html) about Circuit Breaker’s implementation using traditional imperative languages.

## Challenges of functional implementation

When you read the description of this pattern, you may have noticed that Circuit Breaker is a stateful pattern, as well as it requires atomic changes of that state (because you want this thing to be as robust as possible, as it should handle all of your network requests).

In a functional world, we have an approach to implement such requirements: an **IORef** monad — a mutable variable inside the IO monad. Basically, it lets you have a piece of (pure) functional code which communicates with (impure) outside world and atomically modify a variable inside those computations. You may want to take a look at it’s Haskell implementation for more details.

## Implementation

Let’s dive into code! As my library of choice, I will use the awesome [fp-ts](https://github.com/gcanti/fp-ts), which already has an implementation of `IORef`, as well as `Task`, `Either`, and their combination (`TaskEither`) monads.

First of all, let’s define our basic logic.

1. The Circuit Breaker is a proxy wrapper around a network request, which we will model as a simple thunk of `Promise<T>`, leaving the implementation of the call to the user – whether he or she will use a `fetch`, a plain ol' `XMLHttpRequest` or `axios`, it doesn't matter.
2. As we model our stateful Circuit Breaker using a stateless functional approach, we will utilize a common abstraction technique by taking the state as a parameter to the main function.
3. The result of a Circuit Breaker call we will model as a tuple of the next state and lazy result of the call. We usually represent async interactions using a `Task`, but the request may fail, leading us to usage of `Either` monad as well. So the resulting call will be modeled using a [`TaskEither`](https://github.com/gcanti/fp-ts/blob/master/docs/modules/TaskEither.ts.md) monad.

Let’s write a representation of Circuit Breaker’s status as a sum type:

```typescript
class BreakerClosed {
  public readonly tag = 'Closed';
  constructor(public readonly errorCount: number) {}
}

class BreakerOpen {
  public readonly tag = 'Open';
  constructor(public readonly timeOpened: number) {}
}

type BreakerState = BreakerClosed | BreakerOpen;
```

> Note that I don’t define `BreakerHalfOpen` status. It is only needed for stateful implementations, where the breaker itself controls its state. But as we are going with 'inverted' functional approach, the "half-open" state can easily be inferred from the combination of current time and `BreakerOpen`'s payload, `timeOpened`.

Given these states, our main breaker service could be defined like this:

```typescript
const breakerService = (
  request: Lazy<Promise<T>>,
  ref: IORef<BreakerState> = new IORef(breakerClosed(0)),
): [IORef<BreakerState>, TaskEither<Error, T>] =>
  [ref, fromIO<Error, BreakerState>(ref.read).chain(
    (state: BreakerState) => {
      switch (status.tag) {
        case 'Closed':
          return callIfClosed(request, ref);
        case 'Open':
          return callIfOpen(request, ref);
      }
    },
  )];
```

Implementation of `callIfClosed` is pretty straightforward:

```typescript
const callIfClosed = (request: Lazy<Promise<T>>, ref: IORef<BreakerState>): TaskEither<Error, T> =>
  tryCatch(request, (reason) =>
    // the `request` has failed, so we need to increase number of errors in breaker's state and return an error to the user:
    incErrors(ref).map(() => (reason instanceof Error) ? reason : new Error(String(rea
```

Function `incErrors` deals with changing breaker's state from "closed" to "open" if errors count reaches threshold:

```typescript
const incErrors = (ref: IORef<BreakerState>): IO<void> => getCurrentTime().read.chain(
  (currentTime) => ref.read.chain(
    (state) => {
      switch (state.tag) {
        case 'Closed': {
          const errorCount = state.errorCount;
          if (errorCount >= MAX_BREAKER_FAILURES) {
            // We open a breaker for a constant timeout, giving an external service a chance to recover:
            return ref.write(breakerOpen(currentTime + (RESET_TIMEOUT * 1000)));
          } else {
            return ref.write(breakerClosed(errorCount + 1));
          }
        }
        case 'Open': {
          return io.of<void>(undefined); // do nothing, the breaker is already open ¯\_(ツ)_/¯
        }
      }
    },
  ),
);
```

The function `callIfOpen` is more interesting. Remember that we need to pass a request through if the timeout is finished!

```typescript
const callIfOpen = (request: Lazy<Promise<T>>, ref: IORef<BreakerState>): TaskEither<Error, T> =>
  fromIO<Error, boolean>(getCurrentTime().read.chain(
    (currentTime) => ref.read.chain(
      (state) => {
        switch (state.tag) {
          case 'Closed':
            // No way we can get here, so act as if breaker is still open:
            return ref.write(state).map(constFalse);
          case 'Open': {
            if (currentTime > state.openEndTime) {
              // Here we are lettig a request through via returning a `true` as `canaryRequest` result:
              return ref.write(breakerOpen(currentTime + (RESET_TIMEOUT * 1000))).map(constTrue);
            }
            // Nope, still open:
            return ref.write(state).map(constFalse);
          }
        }
      },
    ),
  )).chain(
    // Should we let a request through?
    (canaryRequest) => canaryRequest ? canaryCall(request, ref) : failingCall(),
  );

// A failing call as it is, plain and simple:
const failingCall = (): TaskEither<Error, T> => fromLeft(new Error(BREAKER_ERROR_DESCRIPTION));
```

And the last part of our puzzle, `canaryCall`, is really simple:

```typescript
const canaryCall = (request: Lazy<Promise<T>>, ref: IORef<BreakerState>): TaskEither<Error, T> =>
  // Make a normal request and set breaker's state to "closed" only if the request succedes:
  callIfClosed(request, ref).chain((result: T) => fromIO(ref.write(breakerClosed(0)).chain(() => io.of(result))));
```

# Conclusion

There you have it, a Circuit Breaker pattern in an (almost) purely functional approach! To make it more useful, it is better to provide configuration for `MAX_BREAKER_FAILURES`, `RESET_TIMEOUT` and `BREAKER_ERROR_DESCRIPTION` variables. I like to do this using a `Reader` monad. So a final implementation you can find here: [github://circuit-breaker-monad](https://github.com/YBogomolov/circuit-breaker-monad). Star, fork and try using in your projects! And if you have a suggestion or you have found an issue, please write me in the Issues section.
