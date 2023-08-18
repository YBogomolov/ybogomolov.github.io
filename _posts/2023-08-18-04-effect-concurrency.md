---
layout: post
title:  "Intro To Effect, Part 4: Concurrency in Effect"
author: yuriy
categories: [ fp, typescript, effect ]
image: assets/images/04-effect-concurrency.png
---

In this article, we will delve into the world of fibers and their role in functional effect systems like ZIO or Effect. By exploring fibers and their various features, we aim to equip you with the ability to handle concurrency effectively in functional programming environments. We will discuss different concurrency combinators and demonstrate how they simplify the management of concurrent and asynchronous operations.

<!--more-->

Intro to Effect series:
1. [What is Effect?](https://ybogomolov.me/01-effect-intro)
2. [Handling Errors](https://ybogomolov.me/02-effect-handling-errors)
3. [Managing Dependencies](https://ybogomolov.me/03-effect-managing-dependencies)
4. Concurrency in Effect

Concurrency plays a vital role in many modern applications, enabling efficient utilisation of resources and improved responsiveness. To manage concurrency effectively, functional programming provides an abstraction called fibers, which simplifies handling concurrent and asynchronous operations. This article will introduce you to fibers and discuss their inherent features, as well as pave the way to explore various concurrency combinators. With fibers as the building blocks, you will gain a better understanding of how to model concurrency in your functional applications with ease. So, let's dive into the world of fibers and discover their potential in concurrent programming.

# What is a Fiber?

Effect uses [fiber-based concurrency](https://en.wikipedia.org/wiki/Fiber_(computer_science)) to execute its tasks.

Fibers, in the context of functional effect systems like ZIO or Effect, are lightweight, logical threads of execution for concurrent and asynchronous operations. They can be started, paused, interrupted, and resumed. Fibers are building blocks for modelling concurrency in applications and abstract away the complexities of managing threads and synchronization. Fibers allow you to run effects concurrently without having to deal with low-level thread management or race conditions. Instead, libraries like Effect manage the scheduling and resource allocation of fibers for you.

Here are a few key points about fibers:

1. *Lightweight*: Fibers consume fewer resources than native threads, allowing you to run thousands or even millions of them concurrently without significantly impacting performance. This is because fibers are managed by the effect system's scheduler and share a pool of native threads.
2. *Composable*: Fibers can be combined to create complex concurrency scenarios. You can easily run multiple fibers in parallel, sequentially, or in various combinations.
3. *Resource-safe*: Fibers make it easier to manage resources such as file handles or database connections in a safe and principled manner. These libraries typically provide resource management utilities to help ensure resources are properly acquired and released.
4. *Error handling*: Fibers allow for functional error handling. If an exception occurs within a fiber, the effect system captures the error and allows the developer to handle it appropriately.
5. *Interruption*: Fibers can be interrupted, which means their execution can be safely stopped if necessary. This is useful for implementing timeouts or cancelling background tasks.

In general, fibers are lightweight, composable concurrency primitives that simplify the management of concurrent and asynchronous operations in functional programs.

To learn more about how fiber-based interruption model works in Effect, see [their awesome documentation](https://effect.website/docs/interruption-model).

Fiber-based runtime present in Effect allows evaluating recursive Effect functions without blowing up the stack:

```ts
const add = (a: number, b: number): number => (b === 0 ? a : add(a + 1, b - 1));

console.log(add(1, 9999)); 
//> RangeError: Maximum call stack size exceeded

// Effect implementation:
const addEffect = (a: number, b: number): Effect.Effect<never, never, number> =>
  Effect.gen(function* (_) {
    if (b === 0) return a;
    return yield* _(addEffect(a + 1, b - 1));
  });

void Effect.runPromise(addEffect(1, 9999)); 
//> timestamp=2023-08-17T15:27:55.382Z level=INFO fiber=#0 message=10000
```

Of course, we could use techniques like trampolining to make our `add` function work, but fiber-based runtime has more benefits — it is transparent to the user, it provides more predictable performance and is more scalable in general, it allows for easier resource management and composition, and much more.

Now let's see how we can leverage this to run our effects concurrently.

# Running Effects Concurrently

By default, effects are executed _sequentially_, even if you use combinators like `Effect.all`. This may be a surprise, especially if putting an analogy with `Promise.all` or `Promise.allSettled`. To run them in a concurrent way, you need to specify the concurrency options explicitly. If you think a bit, this design choice makes perfect sense: Effect can be executed not only in a server environment but also in the browser, where resources may be severely limited. Let's look at some examples:

```ts
const tasks = Array.from({ length: 10 }, (_, i) => {
  const delay = i * 100;
  return Effect.log(`${i}, delay: ${delay} ms`).pipe(Effect.delay(`${delay} millis`));
}); // :: Effect<never, never, void>[]

const program = Effect.all(tasks); // :: Effect<never, never, void[]>

await Effect.runPromise(program);
```

This code will output its results sequentially:

```log
timestamp=2023-08-18T08:59:41.198Z level=INFO fiber=#0 message="0, delay: 0 ms"
timestamp=2023-08-18T08:59:41.301Z level=INFO fiber=#0 message="1, delay: 100 ms"
timestamp=2023-08-18T08:59:41.503Z level=INFO fiber=#0 message="2, delay: 200 ms"
timestamp=2023-08-18T08:59:41.805Z level=INFO fiber=#0 message="3, delay: 300 ms"
timestamp=2023-08-18T08:59:42.208Z level=INFO fiber=#0 message="4, delay: 400 ms"
timestamp=2023-08-18T08:59:42.712Z level=INFO fiber=#0 message="5, delay: 500 ms"
timestamp=2023-08-18T08:59:43.314Z level=INFO fiber=#0 message="6, delay: 600 ms"
timestamp=2023-08-18T08:59:44.019Z level=INFO fiber=#0 message="7, delay: 700 ms"
timestamp=2023-08-18T08:59:44.821Z level=INFO fiber=#0 message="8, delay: 800 ms"
timestamp=2023-08-18T08:59:45.725Z level=INFO fiber=#0 message="9, delay: 900 ms"
```

To talk about concurrency, we need to measure how long our effects are executing. Fortunately, Effect comes with batteries included, so profiling our effectful programs is a matter of using the `Effect.timed` combinator!

```ts
const program = Effect.all(tasks).pipe(
  Effect.timed, // <-- transforms result of type `A` into a tuple `[Duration, A]`
  Effect.tap(([duration]) => Effect.log(`Took ${formatHrtime(Duration.toHrTime(duration))}`))
);
```

where `formatHrtime` is a helper to convert `[number, number]` to a nicely formatted string:

```ts
const formatHrtime = ([seconds, nanoseconds]: readonly [number, number]): string => {
  const milliseconds = Math.floor(nanoseconds / 1e6);
  const microseconds = Math.floor(nanoseconds / 1e3) % 1e3;

  const msFormatted = milliseconds > 0 ? ` ${milliseconds}ms` : '';
  const microFormatted = microseconds > 0 ? ` ${microseconds}μs` : '';

  return `${seconds}s${msFormatted}${microFormatted}`;
};
```

Now the results of our execution look like this:

```log
timestamp=2023-08-18T08:59:41.198Z level=INFO fiber=#0 message="0, delay: 0 ms"
timestamp=2023-08-18T08:59:41.301Z level=INFO fiber=#0 message="1, delay: 100 ms"
timestamp=2023-08-18T08:59:41.503Z level=INFO fiber=#0 message="2, delay: 200 ms"
timestamp=2023-08-18T08:59:41.805Z level=INFO fiber=#0 message="3, delay: 300 ms"
timestamp=2023-08-18T08:59:42.208Z level=INFO fiber=#0 message="4, delay: 400 ms"
timestamp=2023-08-18T08:59:42.712Z level=INFO fiber=#0 message="5, delay: 500 ms"
timestamp=2023-08-18T08:59:43.314Z level=INFO fiber=#0 message="6, delay: 600 ms"
timestamp=2023-08-18T08:59:44.019Z level=INFO fiber=#0 message="7, delay: 700 ms"
timestamp=2023-08-18T08:59:44.821Z level=INFO fiber=#0 message="8, delay: 800 ms"
timestamp=2023-08-18T08:59:45.725Z level=INFO fiber=#0 message="9, delay: 900 ms"
timestamp=2023-08-18T08:59:45.726Z level=INFO fiber=#0 message="Took 4s 529ms 645μs"
```

To run those tasks concurrently, all we have to do is provide `options` to the second parameter of `Effect.all`:

```ts
const program = Effect.all(tasks, { concurrency: 5 }).pipe(
  //                              ^^^^^^^^^^^^^^^^^^ — that's all!
  Effect.timed,
  Effect.tap(([duration]) => Effect.log(`Took ${formatHrtime(Duration.toHrTime(duration))}`))
);
```

With `{ concurrency: 5 }` we instructed Effect to run our `tasks` across five fibers, distributing the load in a round-robin manner:

```log
timestamp=2023-08-18T09:01:44.362Z level=INFO fiber=#1 message="0, delay: 0 ms"
timestamp=2023-08-18T09:01:44.461Z level=INFO fiber=#2 message="1, delay: 100 ms"
timestamp=2023-08-18T09:01:44.561Z level=INFO fiber=#3 message="2, delay: 200 ms"
timestamp=2023-08-18T09:01:44.661Z level=INFO fiber=#4 message="3, delay: 300 ms"
timestamp=2023-08-18T09:01:44.761Z level=INFO fiber=#5 message="4, delay: 400 ms"
timestamp=2023-08-18T09:01:44.864Z level=INFO fiber=#1 message="5, delay: 500 ms"
timestamp=2023-08-18T09:01:45.064Z level=INFO fiber=#2 message="6, delay: 600 ms"
timestamp=2023-08-18T09:01:45.262Z level=INFO fiber=#3 message="7, delay: 700 ms"
timestamp=2023-08-18T09:01:45.463Z level=INFO fiber=#4 message="8, delay: 800 ms"
timestamp=2023-08-18T09:01:45.664Z level=INFO fiber=#5 message="9, delay: 900 ms"
timestamp=2023-08-18T09:01:45.669Z level=INFO fiber=#0 message="Took 1s 311ms 215μs"
```

See how `fiber#1` took the next task from the queue once it finished the first one? And total execution time is roughly the sum of the two longest tasks from both batches (400 ms and 900 ms), plus some overhead of the Effect runtime and logging.

We can also specify `{ concurrency: 'unbounded' }` to make Effect use as many fibers as it wants:

```ts
const program = Effect.all(tasks, { concurrency: 'unbounded' }).pipe(
  Effect.timed.
  Effect.tap(([duration]) => Effect.log(`Took ${formatHrtime(Duration.toHrTime(duration))}`))
);
```

The results will be delivered much quicker — at the cost of using more computational resources:

```log
timestamp=2023-08-18T09:04:18.453Z level=INFO fiber=#1 message="0, delay: 0 ms"
timestamp=2023-08-18T09:04:18.553Z level=INFO fiber=#2 message="1, delay: 100 ms"
timestamp=2023-08-18T09:04:18.653Z level=INFO fiber=#3 message="2, delay: 200 ms"
timestamp=2023-08-18T09:04:18.753Z level=INFO fiber=#4 message="3, delay: 300 ms"
timestamp=2023-08-18T09:04:18.853Z level=INFO fiber=#5 message="4, delay: 400 ms"
timestamp=2023-08-18T09:04:18.954Z level=INFO fiber=#6 message="5, delay: 500 ms"
timestamp=2023-08-18T09:04:19.054Z level=INFO fiber=#7 message="6, delay: 600 ms"
timestamp=2023-08-18T09:04:19.154Z level=INFO fiber=#8 message="7, delay: 700 ms"
timestamp=2023-08-18T09:04:19.254Z level=INFO fiber=#9 message="8, delay: 800 ms"
timestamp=2023-08-18T09:04:19.354Z level=INFO fiber=#10 message="9, delay: 900 ms"
timestamp=2023-08-18T09:04:19.359Z level=INFO fiber=#0 message="Took 0s 909ms 646μs"
```

Finally, the most interesting option — `{ concurrency: 'inherit' }`:

```ts
const program = Effect.all(tasks, { concurrency: 'inherit' }).pipe(
  Effect.timed,
  Effect.tap(([duration]) => Effect.log(`Took ${formatHrtime(Duration.toHrTime(duration))}`))
);
```

When specified just as that, it behaves no differently from `unbounded` concurrency. But it has a really nice feature: an effect that has `inherited` concurrency uses the setting of its parent executor! Let me show you an example:

```ts
const program = Effect.all(tasks, { concurrency: 'inherit' }).pipe(
  Effect.timed,
  Effect.tap(([duration]) => Effect.log(`Took ${formatHrtime(Duration.toHrTime(duration))}`))
);

const parent1 = Effect.log('Parent #1').pipe(
  Effect.flatMap(() => program),
  Effect.withConcurrency(5) // <-- observe the `Effect.withConcurrency` combinator
);

const parent2 = Effect.log('Parent #2').pipe(
  Effect.flatMap(() => program),
  Effect.withConcurrency(2) // <-- and this one will run `program` across only two fibers
);
```

Now, when we run the `parent1` and `parent2` effects, we will get different behaviour from our `program` tasks:

```log
timestamp=2023-08-18T09:09:25.830Z level=INFO fiber=#0 message="Parent #1"
timestamp=2023-08-18T09:09:25.834Z level=INFO fiber=#1 message="0, delay: 0 ms"
timestamp=2023-08-18T09:09:25.935Z level=INFO fiber=#2 message="1, delay: 100 ms"
timestamp=2023-08-18T09:09:26.034Z level=INFO fiber=#3 message="2, delay: 200 ms"
timestamp=2023-08-18T09:09:26.133Z level=INFO fiber=#4 message="3, delay: 300 ms"
timestamp=2023-08-18T09:09:26.235Z level=INFO fiber=#5 message="4, delay: 400 ms"
timestamp=2023-08-18T09:09:26.336Z level=INFO fiber=#1 message="5, delay: 500 ms"
timestamp=2023-08-18T09:09:26.537Z level=INFO fiber=#2 message="6, delay: 600 ms"
timestamp=2023-08-18T09:09:26.736Z level=INFO fiber=#3 message="7, delay: 700 ms"
timestamp=2023-08-18T09:09:26.941Z level=INFO fiber=#4 message="8, delay: 800 ms"
timestamp=2023-08-18T09:09:27.138Z level=INFO fiber=#5 message="9, delay: 900 ms"
timestamp=2023-08-18T09:09:27.144Z level=INFO fiber=#0 message="Took 1s 312ms 144μs"

timestamp=2023-08-18T09:10:47.273Z level=INFO fiber=#0 message="Parent #2"
timestamp=2023-08-18T09:10:47.277Z level=INFO fiber=#1 message="0, delay: 0 ms"
timestamp=2023-08-18T09:10:47.377Z level=INFO fiber=#2 message="1, delay: 100 ms"
timestamp=2023-08-18T09:10:47.478Z level=INFO fiber=#1 message="2, delay: 200 ms"
timestamp=2023-08-18T09:10:47.679Z level=INFO fiber=#2 message="3, delay: 300 ms"
timestamp=2023-08-18T09:10:47.880Z level=INFO fiber=#1 message="4, delay: 400 ms"
timestamp=2023-08-18T09:10:48.181Z level=INFO fiber=#2 message="5, delay: 500 ms"
timestamp=2023-08-18T09:10:48.483Z level=INFO fiber=#1 message="6, delay: 600 ms"
timestamp=2023-08-18T09:10:48.884Z level=INFO fiber=#2 message="7, delay: 700 ms"
timestamp=2023-08-18T09:10:49.286Z level=INFO fiber=#1 message="8, delay: 800 ms"
timestamp=2023-08-18T09:10:49.791Z level=INFO fiber=#2 message="9, delay: 900 ms"
timestamp=2023-08-18T09:10:49.796Z level=INFO fiber=#0 message="Took 2s 522ms 96μs"
```

> ❗️ **Pay attention**: do not forget to specify the concurrency options to the `Effect.all` combinator, or else you will be surprised by a not-so-ideal performance of your programs.

## Other Concurrency Combinators

### Zipping

We can `zip` two effects together and specify whether they should happen sequentially or in parallel:

```ts
const parent1 = Effect.log('Parent #1').pipe(
  Effect.flatMap(() => program),
  Effect.withConcurrency(5)
);

const parent2 = Effect.log('Parent #2').pipe(
  Effect.flatMap(() => program),
  Effect.withConcurrency(2)
);

const zipped = Effect.zip(parent1, parent2, { concurrent: true });

await Effect.runPromise(zipped);
```

But when we run this program, we will see a mess of interleaved logs:

```log
timestamp=2023-08-18T09:17:26.846Z level=INFO fiber=#1 message="Parent #1"
timestamp=2023-08-18T09:17:26.849Z level=INFO fiber=#2 message="Parent #2"
timestamp=2023-08-18T09:17:26.850Z level=INFO fiber=#3 message="0, delay: 0 ms"
timestamp=2023-08-18T09:17:26.851Z level=INFO fiber=#8 message="0, delay: 0 ms"
timestamp=2023-08-18T09:17:26.951Z level=INFO fiber=#4 message="1, delay: 100 ms"
timestamp=2023-08-18T09:17:26.951Z level=INFO fiber=#9 message="1, delay: 100 ms"
timestamp=2023-08-18T09:17:27.051Z level=INFO fiber=#5 message="2, delay: 200 ms"
timestamp=2023-08-18T09:17:27.051Z level=INFO fiber=#8 message="2, delay: 200 ms"
timestamp=2023-08-18T09:17:27.150Z level=INFO fiber=#6 message="3, delay: 300 ms"
timestamp=2023-08-18T09:17:27.250Z level=INFO fiber=#7 message="4, delay: 400 ms"
timestamp=2023-08-18T09:17:27.252Z level=INFO fiber=#9 message="3, delay: 300 ms"
timestamp=2023-08-18T09:17:27.351Z level=INFO fiber=#3 message="5, delay: 500 ms"
timestamp=2023-08-18T09:17:27.453Z level=INFO fiber=#8 message="4, delay: 400 ms"
timestamp=2023-08-18T09:17:27.553Z level=INFO fiber=#4 message="6, delay: 600 ms"
timestamp=2023-08-18T09:17:27.752Z level=INFO fiber=#5 message="7, delay: 700 ms"
timestamp=2023-08-18T09:17:27.753Z level=INFO fiber=#9 message="5, delay: 500 ms"
timestamp=2023-08-18T09:17:27.952Z level=INFO fiber=#6 message="8, delay: 800 ms"
timestamp=2023-08-18T09:17:28.053Z level=INFO fiber=#8 message="6, delay: 600 ms"
timestamp=2023-08-18T09:17:28.151Z level=INFO fiber=#7 message="9, delay: 900 ms"
timestamp=2023-08-18T09:17:28.156Z level=INFO fiber=#1 message="Took 1s 308ms 98μs"
timestamp=2023-08-18T09:17:28.454Z level=INFO fiber=#9 message="7, delay: 700 ms"
timestamp=2023-08-18T09:17:28.855Z level=INFO fiber=#8 message="8, delay: 800 ms"
timestamp=2023-08-18T09:17:29.356Z level=INFO fiber=#9 message="9, delay: 900 ms"
timestamp=2023-08-18T09:17:29.358Z level=INFO fiber=#2 message="Took 2s 508ms 889μs"
```

To help differentiate them, we can use a thing called a _log span_ — basically, a way of providing a label to each log line. Log spans have a lifecycle equal to the executing effect, which is a good thing in our case:

```ts
const parent1 = Effect.log('Parent #1').pipe(
  Effect.flatMap(() => program),
  Effect.withConcurrency(5),
  Effect.withLogSpan('parent1')
);

const parent2 = Effect.log('Parent #2').pipe(
  Effect.flatMap(() => program),
  Effect.withConcurrency(2),
  Effect.withLogSpan('parent2')
);

const zipped = Effect.zip(parent1, parent2, { concurrent: true });

await Effect.runPromise(zipped);
```

Now we will get a much better output:

```log
timestamp=2023-08-18T09:23:50.276Z level=INFO fiber=#1 message="Parent #1" parent1=0ms
timestamp=2023-08-18T09:23:50.278Z level=INFO fiber=#2 message="Parent #2" parent2=0ms
timestamp=2023-08-18T09:23:50.280Z level=INFO fiber=#3 message="0, delay: 0 ms" parent1=4ms
timestamp=2023-08-18T09:23:50.280Z level=INFO fiber=#8 message="0, delay: 0 ms" parent2=2ms
timestamp=2023-08-18T09:23:50.380Z level=INFO fiber=#4 message="1, delay: 100 ms" parent1=104ms
timestamp=2023-08-18T09:23:50.381Z level=INFO fiber=#9 message="1, delay: 100 ms" parent2=103ms
timestamp=2023-08-18T09:23:50.481Z level=INFO fiber=#5 message="2, delay: 200 ms" parent1=205ms
timestamp=2023-08-18T09:23:50.481Z level=INFO fiber=#8 message="2, delay: 200 ms" parent2=203ms
timestamp=2023-08-18T09:23:50.581Z level=INFO fiber=#6 message="3, delay: 300 ms" parent1=305ms
timestamp=2023-08-18T09:23:50.680Z level=INFO fiber=#7 message="4, delay: 400 ms" parent1=404ms
timestamp=2023-08-18T09:23:50.680Z level=INFO fiber=#9 message="3, delay: 300 ms" parent2=402ms
timestamp=2023-08-18T09:23:50.782Z level=INFO fiber=#3 message="5, delay: 500 ms" parent1=506ms
timestamp=2023-08-18T09:23:50.884Z level=INFO fiber=#8 message="4, delay: 400 ms" parent2=606ms
timestamp=2023-08-18T09:23:50.982Z level=INFO fiber=#4 message="6, delay: 600 ms" parent1=706ms
timestamp=2023-08-18T09:23:51.182Z level=INFO fiber=#9 message="5, delay: 500 ms" parent2=904ms
timestamp=2023-08-18T09:23:51.183Z level=INFO fiber=#5 message="7, delay: 700 ms" parent1=907ms
timestamp=2023-08-18T09:23:51.383Z level=INFO fiber=#6 message="8, delay: 800 ms" parent1=1107ms
timestamp=2023-08-18T09:23:51.485Z level=INFO fiber=#8 message="6, delay: 600 ms" parent2=1207ms
timestamp=2023-08-18T09:23:51.582Z level=INFO fiber=#7 message="9, delay: 900 ms" parent1=1306ms
timestamp=2023-08-18T09:23:51.586Z level=INFO fiber=#1 message="Took 1s 308ms 593μs" parent1=1310ms
timestamp=2023-08-18T09:23:51.883Z level=INFO fiber=#9 message="7, delay: 700 ms" parent2=1605ms
timestamp=2023-08-18T09:23:52.287Z level=INFO fiber=#8 message="8, delay: 800 ms" parent2=2009ms
timestamp=2023-08-18T09:23:52.785Z level=INFO fiber=#9 message="9, delay: 900 ms" parent2=2507ms
timestamp=2023-08-18T09:23:52.788Z level=INFO fiber=#2 message="Took 2s 509ms 783μs" parent2=2510ms
```

### Racing

We can run two (using `Effect.race` or `Effect.raceFirst`) or more (`Effect.raceAll`) effects to compete, with the result being delivered by the quickest, and the rest will be interrupted:

```ts
const tasks = Array.from({ length: 10 }, (_, i) => {
  const delay = Math.ceil(Math.random() * 100); // <-- randomised delay
  return Effect.log(`${i}, delay: ${delay} ms`).pipe(Effect.delay(`${delay} millis`));
});

const winner = Effect.raceAll(tasks);

await Effect.runPromise(winner);
```

Running this program will yield each time different results:

```log
> ts-node concurrency-parallelism.ts
timestamp=2023-08-18T09:27:43.978Z level=INFO fiber=#1 message="0, delay: 16 ms"

> ts-node concurrency-parallelism.ts
timestamp=2023-08-18T09:27:46.986Z level=INFO fiber=#7 message="6, delay: 2 ms"

> ts-node concurrency-parallelism.ts
timestamp=2023-08-18T09:27:49.442Z level=INFO fiber=#4 message="3, delay: 5 ms"
```

This could be very helpful if you want to write an effect with a capped execution. In this case, you'll want to use `Effect.raceFirst`, as the regular `race` will yield the result of the first _successful_ effect, and not just the quickest one:

```ts
const fetchData = Effect.succeed(42).pipe(
  Effect.delay(`${Math.random() * 1000} millis`),
  Effect.tap((res) => Effect.log(`Result: ${res}`)),
  Effect.withLogSpan('fetcher')
);
const failer = Effect.fail(new Error('timeout')).pipe(
  Effect.delay('500 millis'),
  Effect.catchAll((err) => Effect.logError(`Error: ${err.message}`)),
  Effect.withLogSpan('failer')
);

const result = Effect.raceFirst(fetchData, failer);

await Effect.runPromise(result);
```

This may yield results like these:

```log
> ts-node concurrency-parallelism.ts
timestamp=2023-08-18T11:01:08.576Z level=INFO fiber=#1 message="Result: 42" fetcher=382ms

> ts-node concurrency-parallelism.ts
timestamp=2023-08-18T10:59:51.197Z level=ERROR fiber=#2 message="Error: timeout" failer=503ms
```

### Iterating an Iterable

We also can utilise a very useful combinator — `Effect.forEach` to traverse an array of values, applying an effectful computation to each of them, all with the possibility to execute the traversal concurrently:

```ts
const tasks = Effect.forEach(
  Array.from({ length: 10 }, (_, i) => i),
  (i) => Effect.log(`Value: ${i}`).pipe(Effect.delay(`${Math.random() * 1000} millis`)),
  { concurrency: 3 } // <-- or 'inherit', or 'unbounded'
); // :: Effect<never, never, void[]>

await Effect.runPromise(tasks);
```

Running this code yields:

```log
timestamp=2023-08-18T11:04:23.715Z level=INFO fiber=#3 message="Value: 2"
timestamp=2023-08-18T11:04:24.048Z level=INFO fiber=#2 message="Value: 1"
timestamp=2023-08-18T11:04:24.263Z level=INFO fiber=#3 message="Value: 3"
timestamp=2023-08-18T11:04:24.412Z level=INFO fiber=#2 message="Value: 4"
timestamp=2023-08-18T11:04:24.596Z level=INFO fiber=#1 message="Value: 0"
timestamp=2023-08-18T11:04:24.598Z level=INFO fiber=#3 message="Value: 5"
timestamp=2023-08-18T11:04:24.716Z level=INFO fiber=#1 message="Value: 7"
timestamp=2023-08-18T11:04:25.044Z level=INFO fiber=#1 message="Value: 9"
timestamp=2023-08-18T11:04:25.146Z level=INFO fiber=#2 message="Value: 6"
timestamp=2023-08-18T11:04:25.513Z level=INFO fiber=#3 message="Value: 8"
```

### Merging

Another super-cool feature — the ability to execute an array of effectful tasks, merging (reducing) their values together with a `zero` element:

```ts
const tasks = Array.from({ length: 10 }, (_, i) =>
  Effect.succeed(i).pipe(
    Effect.tap((i) => Effect.log(`Value: ${i}`)),
    Effect.delay(`${Math.random() * 1000} millis`),
    Effect.withLogSpan('mapper')
  )
); // :: Effect<never, never, number>[]

const sumOfTen = Effect.mergeAll(
  tasks, // <-- tasks to execute and merge
  0, // <-- initial accumulator value
  (acc, element) => acc + element, // <-- reducer function
  { concurrency: 3 }
); // :: Effect<never, never, number>

await Effect.runPromise(
  sumOfTen.pipe(
    Effect.tap((sum) => Effect.log(`Sum: ${sum}`)),
    Effect.withLogSpan('reducer')
  )
);
```

This will print:

```log
timestamp=2023-08-18T11:24:26.116Z level=INFO fiber=#1 message="Value: 0" mapper=450ms reducer=451ms
timestamp=2023-08-18T11:24:26.216Z level=INFO fiber=#2 message="Value: 1" mapper=549ms reducer=551ms
timestamp=2023-08-18T11:24:26.320Z level=INFO fiber=#2 message="Value: 4" mapper=104ms reducer=655ms
timestamp=2023-08-18T11:24:26.416Z level=INFO fiber=#3 message="Value: 2" mapper=749ms reducer=751ms
timestamp=2023-08-18T11:24:26.566Z level=INFO fiber=#1 message="Value: 3" mapper=447ms reducer=901ms
timestamp=2023-08-18T11:24:26.583Z level=INFO fiber=#1 message="Value: 7" mapper=16ms reducer=918ms
timestamp=2023-08-18T11:24:26.647Z level=INFO fiber=#3 message="Value: 6" mapper=230ms reducer=982ms
timestamp=2023-08-18T11:24:26.764Z level=INFO fiber=#2 message="Value: 5" mapper=444ms reducer=1099ms
timestamp=2023-08-18T11:24:26.938Z level=INFO fiber=#1 message="Value: 8" mapper=355ms reducer=1273ms
timestamp=2023-08-18T11:24:27.090Z level=INFO fiber=#3 message="Value: 9" mapper=443ms reducer=1425ms
timestamp=2023-08-18T11:24:27.094Z level=INFO fiber=#0 message="Sum: 45" reducer=1429ms
```

> Observe how `mapper` logs time spent in each effect that was producing the individual values, and `reducer` accumulates time because it traverses the array of results.

If you haven't guessed already, Effect gives you a built-in map-reduce framework _out of the box_! How cool is that?

# Conclusion

As you have seen, fibers serve as a powerful and efficient means of handling concurrency in functional programming environments. By abstracting away the complexities of thread management and race conditions, they allow developers to focus on the logic and structure of their applications. With the aid of various concurrency combinators, you can easily manage concurrent execution and resource allocation, resulting in more scalable and robust functional programs.
