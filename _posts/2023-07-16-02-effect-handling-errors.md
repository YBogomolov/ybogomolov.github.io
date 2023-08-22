---
layout: post
title:  "Intro To Effect, Part 2: Handling Errors"
author: yuriy
categories: [ fp, typescript, effect ]
image: assets/images/02-effect-handling-errors.png
---

One of the most important aspects of programming is finding a reliable way of dealing with errors, making them visible, and making them actionable. In conventional TypeScript, we don't have such means except, probably, `@throws` JSDoc annotation. Effect relieves this pain by bringing errors to the surface of type and giving you a lot of convenient instruments for handling them.

<!--more-->

Intro to Effect series:
1. [What is Effect?](https://ybogomolov.me/01-effect-intro)
2. Handling Errors
3. [Managing Dependencies](https://ybogomolov.me/03-effect-managing-dependencies)
4. [Concurrency in Effect](https://ybogomolov.me/04-effect-concurrency)
5. [Software Transactional Memory in Effect](https://ybogomolov.me/05-effect-stm)

---

The biggest problem I have with modern TypeScript is that errors are completely transparent to the programmer. Whether a function throws, returns a rejected Promise, or just calls `process.exit()` — you'll never know. This leads to a whole bunch of problems, starting with unreliable behaviour in the runtime and ending with a maintenance nightmare.

If you recall one of my previous articles, "[Tasks as an alternative to Promises](https://ybogomolov.me/04-tasks)", there I introduced `TaskEither` as a type-safe way of bringing errors to the surface and making the programmer not to forget handling them. Effect goes in the same direction — its second type parameter, `E` , shows which possible errors could an effectful expression return, and equips you with a whole bunch of helper functions to efficiently deal with them.

# Defects and Failures

There are two possible types of failures we as developers have to deal with:
- *defects* — an unexpected error that we cannot recover from, such as running out of memory, or catching a `SIGKILL` signal;
- *failure* — an error condition that represents a flaw in the program's logic, such as division by 0 or parsing a malformed JSON string.

Effect deals only with the latter, as tracking all defects is rather an impossible task. So let's add to our mental model of Effect that its `E` channel is used to track only *failures*. However, there are methods of partial recovery from defects present in Effect runtime.

# Reporting Failures

When working with Effect, an error is just a value, so no stack machine interruption happens when you return an `Effect.fail` expression. This allows errors to be accumulated and composed in a flexible manner. You should become comfortable with *returning* errors rather than *throwing* them.

For example, let's write a program that simulates getting user name and age over network, validates them, and prints to console:

```ts
import { Effect, Random, Duration } from 'effect';
import { NegativeAgeError, UnderageError, NameError } from './errors';

const validateAge = (age: number): Effect.Effect<never, NegativeAgeError | UnderageError, number> => {
  if (age < 0) {
    return Effect.fail(new NegativeAgeError(age));
  }

  if (age < 18) {
    return Effect.fail(new UnderageError(age));
  }

  return Effect.succeed(age);
};

const fetchAge = Effect.gen(function* (_) {
  const age = yield* _(Effect.succeed(-10).pipe(Effect.delay(Duration.seconds(2))));

  return yield* _(validateAge(age));
}); // :: Effect<never, NegativeAgeError | UnderageError, number>

const fetchName = Effect.gen(function* (_) {
  const shouldSucceed = yield* _(Random.nextBoolean);

  if (shouldSucceed) {
    return yield* _(Effect.succeed('John').pipe(Effect.delay(Duration.seconds(1))));
  }

  return yield* _(Effect.fail(new NameError('Missing name')).pipe(Effect.delay(Duration.seconds(1))));
});

const userDetails = Effect.all([fetchAge, fetchName]).pipe(Effect.map(([age, name]) => ({ age, name })));
// :: Effect<never, NegativeAgeError | UnderageError | NameError, { age: number; name: string }>
```

Here we compose two effects that can fail, and the resulting `userDetails` effect accumulates the errors — its error type is the union of the input effect's errors.

# Handling Failures

Effect provides a rich set of combinators to handle failures flexibly — you can manually match on Effect state, provide fallbacks, catch all or some failures, or just simply terminate the execution of your program.

## Matching On `Effect<R, E, A>`

As you probably remember from [the previous article](https://ybogomolov.me/01-effect-intro), `Effect<R, E, A>` could be understood as a function returning an `Either<E, A>`, which is a [discriminated union](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#discriminated-unions). As it is usually done with discriminated unions, you can match on them, dealing with all cases separately.  Effect provides two useful functions that allow pure and effectful pattern matching — `match` and `matchEffect`:

```ts
declare const userDetails: Effect<never, NegativeAgeError | UnderageError | NameError, { age: number; name: string }>;

const handled = userDetails.pipe(
  Effect.match({
    onFailure: () => ({ age: 0, name: 'Anonymous' }),
    onSuccess: x => x,
  })
); // :: Effect<never, never, { age: number, name: string }>

// equivalent to:
const handled = userDetails.pipe(
  Effect.matchEffect({
    onFailure: () => Effect.succeed({ age: 0, name: 'Anonymous' }),
    onSuccess: Effect.succeed,
  })
); // :: Effect<never, never, { age: number, name: string }>
```

## Falling Back

Effect provides `orElse`, `orElseFail`, and `orElseSucceed` functions that allow you to provide a lazy fallback values or Effect expressions that will be evaluated only if the previous effect is a failure:

```ts
const handled = userDetails.pipe(
  Effect.orElse(() => Effect.succeed({ age: 0, name: 'Anonymous' }))
); // :: Effect<never, never, { age: number, name: string }>

// equivalent to:
const handled = userDetails.pipe(
  Effect.orElseSucceed(() => ({ age: 0, name: 'Anonymous' }))
); // :: Effect<never, never, { age: number, name: string }>
```

Please notice that after calling `orElseSucceed` the failure type in Effect signature is inferred as `never`, showing that the resulting Effect expression will never be evaluated as a failure — which means that we handled all failure situations! Meanwhile, if you call `orElseFail`, you will get a type of failure that this function returns:
	
```ts
const notHandled = userDetails.pipe(
  Effect.orElseFail(() => new TypeError('Not handled'))
); // :: Effect<never, TypeError, { age: number, name: string }>
//                     ^^^^^^^^^ — a new type of failure!	
```

## Catching Specific Failures

Before we dive into catching failures with Effect, I want to stress that you need to turn your errors into easily discriminated ones. The required by Effect way is adding `_tag` field to all your errors:

```ts
class NegativeAgeError extends Error {
  readonly _tag = 'NegativeAgeError';
  constructor(age: number) {
    super(`Age cannot be negative: ${age}`);
    this.name = this.constructor.name;
  }
}

class UnderageError extends Error {
  readonly _tag = 'UnderageError';
  constructor(age: number) {
    super(`Age cannot be under 18: ${age}`);
    this.name = this.constructor.name;
  }
}

class NameError extends Error {
  readonly _tag = 'NameError';
  constructor(message: string) {
    super(message);
    this.name = this.constructor.name;
  }
}
```

Effect has a few very useful functions — `catch`, `catchAll`, `catchTag`, and their variations — that allow you to handle all or some of the errors, narrowing down the Effect signature:

```ts
const handled = userDetails.pipe(
  Effect.catchAll(() => Effect.succeed({ age: 0, name: 'Anonymous' }))
); // :: Effect<never, never, { age: number, name: string }>

const partiallyHandled = userDetails.pipe(
  Effect.catch('_tag', {
    failure: 'NegativeAgeError',
    onFailure: () => Effect.succeed({ age: 0, name: 'Anonymous' }),
  })
);

// Uses `_tag` field to discriminate on errors:
const partiallyHandled = pipe(
  userDetails,
  Effect.catchTag('NegativeAgeError', () => Effect.succeed({ age: 0, name: 'Anonymous' })),
  Effect.catchTag('UnderageError', () => Effect.succeed({ age: 0, name: 'Anonymous' }))
); // :: Effect<never, NameError, { age: number, name: string }>
```

There's also `catchSome` function that allows you to handle errors manually, allowing also *widening* the type of failure:

```ts
const partiallyHandled = userDetails.pipe(
  Effect.catchSome(err => {
    if (err instanceof NameError) {
      return Option.none<Effect.Effect<never, NameError, { age: number; name: string }>>();
    }

    return Option.some(
      Effect.if(Random.nextBoolean, {
        onTrue: Effect.succeed({ age: 0, name: 'Anonymous' }),
        onFalse: Effect.fail(new RangeError('Not handled')),
      })
    );
  })
); // Effect<never, NegativeAgeError | UnderageError | NameError | RangeError, { age: number; name: string; }>
//                                                                 ^^^^^^^^^^ — added!
```

## Dying

But what if you cannot recover from a failure or just don't want to, but still need to do something about failures? In this case, you can call the `orDie` or `orDieWith` functions to make the Effect runtime throw an error and terminate execution:

```ts
const unsafeHandled = userDetails.pipe(Effect.orDie); // :: Effect<never, never, { age: number, name: string }>

const unsafeHandled = userDetails.pipe(
  // or you can provide your own error transformer:
  Effect.orDieWith(err => new Error(err.name))
); // :: Effect<never, never, { age: number, name: string }>

// achieving the same as above manually:
const unsafeHandled = userDetails.pipe(
  Effect.orElse(() => Effect.die(new Error('Not handled')))
); // :: Effect<never, never, { age: number, name: string }>
```

## Retrying

What if when a failure happens you really just want to retry the action? Well, Effect got you covered! It has built-in combinators — `retry`/`retryN`, `retryOrElse`, `retryUntil`, `retryWhile` — that allow you to provide an action and a retry policy. Let's see them in action:

```ts
const retried = Effect.retry(userDetails, Schedule.forever); // will run until succeeds

const retriedN = Effect.retryN(userDetails, 10); // will retry 10 times and then fail

const retriedUntil = Effect.retryUntil(
  userDetails,
  err => err._tag === 'NameError'
); // will retry until encounters a NameError, and then fail

// equivalent to:
const retriedWhile = Effect.retryWhile(
  userDetails,
  err => err._tag === 'NegativeAgeError' || err._tag === 'UnderageError'
);

const retriedUntilEffect = Effect.retryUntilEffect(userDetails, err =>
  // accepts an effectful predicate that returns a condition whether the error should be retried
  err._tag === 'NegativeAgeError' ? Effect.succeed(false) : Effect.succeed(true)
);
```

I want to say a few words about the retry policy. Effect provides a `Schedule` module that contains a lot of combinators and constructors that allow you to create any imaginable retry policy you want. Do you want it to run forever?

```ts
const program = Effect.retry(action, Schedule.forever);
```

Do you want it to back off at exponentially-increasing times, capped at 10 repetitions?

```ts
const program = Effect.retry(
  action,
  Schedule.exponential(Duration.seconds(5), 2).pipe(Schedule.intersect(Schedule.recurs(10)))
);
```

Or, maybe, you want your delays to be a Fibonacci sequence?

```ts
const program = Effect.retry(userDetails, Schedule.fibonacci(Duration.seconds(5)));
```

And, of course, different schedules could be combined using `Schedule.union`, `Schedule.intersection`, or `Schedule.sequence`. I urge you to explore this module, as it is incredibly awesome!

# Conclusion

From my perspective, the key benefit of using Effect is modelling errors explicitly via custom types and handling them in a total, type-safe manner. Overall, Effect's error-handling model leads to safer code by forcing developers to handle failures in a comprehensive manner. The functional nature avoids mutation bugs, making it well-suited for mission-critical applications.
