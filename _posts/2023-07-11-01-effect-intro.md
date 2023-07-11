---
layout: post
title:  "Intro To Effect, Part 1: What Is Effect?"
author: yuriy
categories: [ fp, typescript, effect ]
image: assets/images/01-effect-intro.png
---

Recently, [Effect](https://effect.website) has gained incredible traction in the functional programming community. In this series of articles, I give an overview of Effect and its ecosystem. In the first article of the series, we take a look at what `Effect<R, E, A>` is, how to create, and how to compose effectful programs.

<!--more-->

Recently, [Effect](https://effect.website) has gained incredible traction in the functional programming community. While initially being a 1:1 port of [ZIO](https://zio.dev), it fully embraced the power of TypeScript and evolved in its own, unique way. With this post, I start a series of articles that will guide you, my reader, through all peculiarities of Effect.

I expect my reader to be familiar with the basics of functional programming and the `fp-ts` library, as many concepts (like `pipe`ing functions or using type classes) are the same or very similar in Effect.

Buckle up, as we won't stop.

# What is Effect?

When we look at the official documentation, we see this explanation:
> The `Effect<Requirements, Error, Value>` data type represents an **immutable** value that **lazily** describes a workflow or job.
> It encapsulates the logic of a program that requires a collection of contextual data `Context<Requirements>` to execute. This program can either fail, producing an error of type `Error`, or succeed, yielding a value of type `Value`.

From my perspective, in the beginning, it is easier to reason about Effect programs using the `ReaderTaskEither` type:

```ts
type Effect<R, E, A> 
  = ReaderTaskEither<R, E, A>
  = Reader<R, TaskEither<E, A>>
  = (context: R) => () => Promise<Either<E, A>>
```

But in reality, it's not that simple. The thing is, just like ZIO, Effect encapsulates not only *asynchronous* but also *synchronous* expressions, and this makes its mental model a bit complex for a beginner. To start working with Effect, my suggestion might be a little bit controversial: forget about the sync part, and focus on understanding the Effect as a model of asynchronous computation. Just like with Promises: when you started working with them in your program, there's no going back — it's all `Promise` all the way down. Then, when you're firmly comfortable with that mental model, enhance it with an understanding that Effect expressions can also represent synchronous computations.

What uniquely differentiates Effect from ZIO is that Effect is *covariant* in its context type `R`, while ZIO is *contravariant*. This is a deliberate decision by the Effect team which works best for TypeScript because an intersection of conflicting types is reduced to `never`, and that hurts the development experience. 

So, the simplest mental model for Effect would be:

```ts
type Effect<R, E, A> = (context: R) => Promise<E ⋃ A>;
```

I intentionally introduced the`⋃` symbol instead of TypeScript's union `|` to represent that it is a *disjoint union*, i.e. you always can differentiate between its constituents.

**Recap**: an Effect expression is a synchronous or asynchronous computation that:
- may succeed with a *result* of type `A`,
- or may fail with an *error* of type `E`,
- and requires a *computational context* of type `R` to execute.

The whole “computational context” is a bit complex, but we will deal with it in the further articles of the series, so stay tuned!

Down below, Effect is an implementation of the eDSL pattern: the biggest part of its API is dedicated to constructing expressions that represent various aspects of modern programs — async, async, concurrent, suspended executions, access to a dependency, etc. — and only a small part actually *runs* the code.

For those who attended my workshop “[Building eDSLs in functional TypeScript](https://www.youtube.com/watch?v=hTnxaB52awA)”, Effect most closely resembles the Freer monad pattern: Effect expressions are just POJOs that merely *represent* programs, which are *interpreted* by the Effect runtime. This powerful design pattern allows Effect to achieve incredible performance, resource efficiency, and excellent compositional properties.

# Creation of Effect expressions

There is a wide spectrum of ways you can create an Effect expression:
- with `Effect.succeed`:
  ```ts
  const program = Effect.succeed(42); // :: Effect<never, never, number>;  
  ```
- with `Effect.fail`:
  ```ts
  const program = Effect.fail('oops' as const); // :: Effect<never, 'oops', never>;
  ```
- from lazy functions:
  ```ts
  const program1 = Effect.sync(() => {
    console.log('Howdy!');
    return 42;
  }); // :: Effect<never, never, number>
  
  const program2 = Effect.try(() => fs.readFileSync('file.txt', 'utf8')); // :: Effect<never, unknown, string>
  ```
- from lazy Promises:
  ```ts
  const program1 = Effect.promise(() => Promise.resolve(42)); // :: Effect<never, never, number>
  
  const program2 = Effect.tryPromise(() => fs.promises.readFile('file.txt', 'utf8')); // :: Effect<never, unknown, string>
  ```
  As you've probably guessed, `promise` requires Promises to never reject, and `tryPromise` deals with possible rejections.

As for NodeJS-style callbacks, the official documentation suggests using `Effect.async` each time, but I prefer having a small helper that converts any nodeback into an Effect-returning function with correctly inferred types:

```ts
export function effectify<L, R>(
  f: (cb: (e: L | null | undefined, r?: R) => void) => void
): () => Effect.Effect<never, L, R>;
export function effectify<A, L, R>(
  f: (a: A, cb: (e: L | null | undefined, r?: R) => void) => void
): (a: A) => Effect.Effect<never, L, R>;
// and other overloads for different numbers of parameters, omitted for brevity
export function effectify<L, R>(f: Function): () => Effect.Effect<never, L, R> {
  return function () {
    const args = Array.prototype.slice.call(arguments);
    return Effect.async<never, L, R>((resume) => {
      const resolver = (e: L, r: R) => (e != null ? resume(Effect.fail(e)) : resume(Effect.succeed(r)));
      f.apply(null, args.concat(resolver));
    });
  };
}
```

An example of usage:

```ts
const readFile = effectify(fs.readFile); 
// :: (a: fs.PathOrFileDescriptor) => Effect.Effect<never, NodeJS.ErrnoException, Buffer>
```

# Composing Effect expressions

As the Effect type implements an interface of Functor, Monad, Bifunctor, Alternative, and many, many other type classes, it has a host of different ways of composing effectful values:
- using `map` (Functor interface)
  ```ts
  const program1 = pipe(
    Effect.succeed(21),
    Effect.map(n => n * 2)
  ); // :: Effect<never, never, number>
  ```
- using `flatMap` (Monad interface):
  ```ts
  const program2 = pipe(
    Effect.succeed(21),
    Effect.flatMap((n) => (Math.random() > 0.5 ? Effect.succeed(n * 2) : Effect.fail('no luck' as const))),
    Effect.flatMap((n) => readFile(`file${n}.txt`)),
    Effect.map((buf) => buf.toString('utf8'))
  ); // :: Effect<never, NodeJS.ErrnoException | "no luck", string>
  ```
- using `orElse` (Alt interface):
  ```ts
  const program3 = pipe(
    program2,
    Effect.orElse(() => Effect.succeed('dummy file content'))
  )
  ```

Also, using `flatMap` you can combine Effect expressions with Option and Either:
```ts
const doubleOption = (n: number) => (Math.random() > 0.5 ? Option.some(n * 2) : Option.none());

const program1 = pipe(Effect.succeed(21), Effect.flatMap(doubleOption));

const doubleEither = (n: number) => (Math.random() > 0.5 ? Either.right(n * 2) : Either.left('no luck' as const));

const program2 = pipe(Effect.succeed(21), Effect.flatMap(doubleEither));
```

IMO, this is pretty neat and really contributes to a good development experience. But hold on, the next thing is even better! The Effect project embraced generators as a way of doing monadic comprehensions, akin to `do` notation in Haskell or `for` comprehensions in Scala:

```ts
// We start with calling `Effect.gen` that needs a generator function.
// This function receives a parameter, conventionally called `_` (you may also encounter `$`),
// which is used to "lift" effectful values into a generator:
const program = Effect.gen(function* (_) {
  // We use `yield* _( <effect expression here> )` to do monadic comprehensions.
  // I will show how to work properly with random numbers in Effect later on, but as of now,
  // let's stick to this naïve implementation:
  const n = yield* _(Effect.succeed(Math.floor(Math.random() * 100)));
  
  const doubleN = yield* _(doubleOption(n));

  // You can even use your normal control flow statements here:
  if (doubleN % 2 === 0) {
    yield* _(Effect.log(`${doubleN} is even`));
  } else {
    yield* _(Effect.log(`${doubleN} is odd`));
  }

  // Finally, the return value of this function will be in the result type of the Effect
  return doubleN;
}); // :: Effect<never, NoSuchElementException, number>

```

At first, this seems to be rather complicated — `_`, `yield*`, what's going on?! — but if you squint, you'll see a striking familiarity with the `async/await` syntax for Promises. Let's rewrite the program in an imaginary TypeScript that supports async/await for Effect expressions on the top level:

```ts
const program = async () => {
  const n = await Effect.succeed(Math.floor(Math.random() * 100)));
  const doubleN = await doubleOption(n);

  if (doubleN % 2 === 0) {
    await Effect.log(`${doubleN} is even`);
  } else {
    await Effect.log(`${doubleN} is odd`);
  }

  return doubleN;
}; // :: Effect<never, NoSuchElementException, number>
```

And, of course, as an `fp-ts`-inspired library, Effect supports an old way of doing monadic comprehensions in TypeScript called `Do-syntax` that you can use in a `pipe`. Here's an equivalent program using `Do` syntax:

```ts
const program = pipe(
  Effect.Do,
  Effect.bind('n', () => Effect.succeed(Math.floor(Math.random() * 100))),
  Effect.bind('doubleN', ({ n }) => doubleOption(n)),
  Effect.bind('', ({ doubleN }) => {
    if (doubleN % 2 === 0) {
      return Effect.log(`${doubleN} is even`);
    }
    return Effect.log(`${doubleN} is odd`);
  }),
  Effect.map((ctx) => ctx.doubleN)
); // :: Effect<never, NoSuchElementException, number
```

So as you can see, you have quite a few options for working with Effect expressions — chose the one most convenient for you. But as with all choices, I highly recommend sticking to a single one to keep your code base maintainable. IMO, the most powerful option would be `Effect.gen`, as it allows one to use the convenient control flow statements.

# Interpreting Effect programs

What is really important is that Effect expressions are always just pure immutable values. They have no execution model attached to them. To actually execute an Effect program, you need to pass it to the Effect runtime:

1. For running asynchronous effects, you can use `runPromise` or `runPromiseExit` to interpret your effectful programs
    ```ts
    const program1 = Effect.succeed(42);
    const program2 = Effect.fail(new Error('oops'));
    
    await Effect.runPromise(program1); // => 42
    await Effect.runPromise(program2); // rejects with Error('oops')
    
    await Effect.runPromiseExit(program1); // => Exit.succeed(42)
    await Effect.runPromiseExit(program2); // => Exit.fail(Error('oops'))
    ```
2. For running synchronous effects, you can use `runSync` and `runSyncExit` functions. The catch is that they will **throw an error** if any of your programs contain asynchronous effects in their chains. That's why I do not recommend relying on these methods unless you have a good mental model of Effect and know that the program you're going to run contains only synchronous expressions.

> As a rule of thumb: stick to `runPromise`/`runPromiseExit`, and you'll be safe.

---

This concludes the first article of this series. What do you think of Effect? Have you tried it, or, maybe, even delivered a production project using it? Leave a comment below to help spread the word about Effect 👇