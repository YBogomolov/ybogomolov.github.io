---
layout: post
title:  "#MonadicMonday compilation: June"
author: yuriy
categories: [ fp, typescript, monadic monday ]
image: assets/images/monadicmonday.jpg
---

In 2019 I started an activity in Twitter called #monadicmonday – each Monday I posted a thread about some FP stuff which is useful and is easy to start using right away. This is a compilation of the third month, June.

<!--more-->

# Episode 10: Reasons to choose FP

Welcome to tenth episode of #monadicmonday! I would like to make today's episode lite and codeless, and talk about reasons to chose FP, and how to show its viability to the customers.

When you make decisions as an engineer or architect, you have to consider a lot of things when chosing programming paradigm. It's not a big secret that functional approach is far less popular than imperative style, and when you choose it, you have to deal with consequences.

What these consequences might be? 

1. First of all, increased complexity of on-boarding for developers. Unless you hide all functional stuff somewhere under the hood, your developers have to learn what a functor, a monad, an applicative, etc. are and how they are used. And even that isn't a guarantee that your code base will follow best functional practices. It is still possible to write imperatively using monads&stuff, and think that you're "doing FP".
2. Second, if you bid on hiring already familiar with FP devs from the market, be ready to close your open positions for months, unless your primary language is (semi)pure functional one like Haskell/PureScript/etc. And if you expose the hardcore bits (say, like recursion schemes or freer monads) to the surface, chances to find quickly an individual who truly understands such topics are very little.
3. Third, amount of pratical materials about FP is far lesser than about any other approach. Yes, we have a lot of yet-another-monad-tutorial, but there's little to none materials about functional design patterns, mental models and problem decomposition examples. Nowadays the situation is slowly changing, and I hope to stand among those who finally change it for good.

But what benefits are we gaining when we make a decision to write in a functional paradigm?

1. Equational reasoning. When we program with pure functions, monads, algebraic data types, we gain tremendous power of equational reasoning – ability to substitute equals by equals in all contexts. This allows ut to check the correctness of our programs "in mind compiler", and be confident about our understanding of the code. I must admit, this requires additional efforts and in many cases support from the language as well.
2. Modularity and composability. This could be achieved in other styles, as well, – say, in traditional OOP. Low coupling & high cohesion require your code to be modular and isolated, which often leads to good composability. However, only in functional style these characteristics of code are a cornerstone to the whole approach.
3. Testability. Naturally follows from p.2. If your code is modular, it's very easy to mock parts of it and test them independently. Please check out fifth episode about Tagless Final and the example of testing the program in that style: https://twitter.com/YuriyBogomolov/status/1124244215620341760
4. Robustness. I mean robustness in mathematical sense – if your code follows mathematical laws like associativity, transitivity, idempotence, totality, it's far less likely that it contains errors. This doesn't mean you don't need tests or you're allowed to write sloppy, but when a compiler backs you up, it's a good feeling.
5. Proofs of correctness. I'm looking over a horison here, but using math apparatus of type theory and its successors (like HoTT) it is possible to build automatic proofs of correctness of the system you're building. Using dependently-typed languages like Coq and Agda it is possible not only to express the logic logic of your application, but also prove that it doesn't contain flaws and mistakes. Dep-types languages still have a long road ahead of them to be truly useful (and usabe) in production, but it's only the beginning.

So how might you communicate to a customer when you're defending your decision to use FP? First of all, you have to answer the simple question: what is the main pain this customer is facing? Is his/her code is tightly coupled and poorly testable? Is it not highly reusable? Does it have a lot of runtime errors like NPE or 'undefined is not a function'? Rank those pain points and build your statements around the solutions to those. Also, be fair to yourself and don't bring FP only for sake of brining FP :) Always consider the people you'll be working with, the overall situation with the codebase and the environment you'll be working in. Talking about FP and migration to Scala with a customer who uses JavaEE in all solutions for 20+ years is not a very bright idea. However, if you're building a frontend app, you can ops to chose TypeScript – and convince the customer that it's a viable decision by showing that is leads to lesser number of bugs per line, more development convenience and ease of transition for pure JS developers. Almost in any language it is possible to use bits of FP, and it's up to you to find the fine balance of FP and non-FP stuff in your solution.

Join the discussion! I would love to hear about your experience with "selling" FP to the customer, lessons learned and transition approaches.

# Episode 11: Kleisli arrows

Welcome to eleventh episode of #monadicmonday! Today we'll talk about Kleisli arrows, and I'll introduce my small project called "kleisli-ts".

Function composition is a cornerstone of functional programming. Having `f :: A -> B` and `g :: B -> C`, we compose `g . f` and obtain `h :: A -> C`. This works well when we are speaking about total pure functions. But when we interact with the real world, problems appear. Real World™ is asynchronous, impure and seems to be a total opposite of what we want it to be.

The usual approach involves monads in all their glory – IO, Task, Reader, with an apex in form of bifunctorial ZIO (see https://twitter.com/YuriyBogomolov/status/1125403292530507776). Probably, you know the main problem with monads – they don't compose, at least in generic way: https://www.slideshare.net/pjschwarz/monads-do-not-compose

However, there's another way of writing effectful functions that compose well, and this way involves Kleisli arrows.

Kleisli arrows are named after Swiss mathematicial Heinrich Kleisli, who worked in a field of category theory and homotopy theory. These arrows are functions in form `f :: A -> F B`, having `F` as a context of our computation. In TypeScript we can write them down as a type with three parameters, using fp-ts notation for HKTs:

```ts
// should be interpreted as: (a: A) => F<B>
type Kleisli<F extends URIS, A, B> = (a: A) => Type<F, B>; 
```

Having functions of such form, we can compose them if we have a monad for the `F` kind (or even narrower: if we have a Chain<F>):

```ts
const composeK = <F extends URIS>(F: Chain1<F>) =>
  <A, B, C>(f: Kleisli<F, A, B>, g: Kleisli<F, B, C>): Kleisli<F, A, C> => 
    (a: A) => F.chain(f(a), g);
```

We can defined some useful combinators, such as `zipWith` for zipping two Kleisli arrows, or `ifThenElse` to organize the conditional flow:

```ts
const zipWith = <F extends URIS>(M: Chain1<F>) =>
  <A, B, C, D>(l: Kleisli<F, A, B>, r: Kleisli<F, A, C>) =>
    (f: (t: [B, C]) => D): Kleisli<F, A, D> =>
      (a) => M.chain(l(a), (b) => M.map(r(a), (c) => f([b, c])));

const ifThenElse = <F extends URIS>(M: Chain1<F>) =>
  <A, B>(cond: Kleisli<F, A, boolean>) =>
    (then: Kleisli<F, A, B>) => (else_: Kleisli<F, A, B>): Kleisli<F, A, B> =>
      (a) => M.chain(cond(a), (t) => t ? then(a) : else_(a));
```

This may look like a tongue-in-cheek replacement – after all, we are still using monadic chaining for passing the values around, – but in reality this approach allows you to think about your code in a different way, and what's more important, Kleisli arrows allow some optimizations to be done, mitigating allocation overhead of monadic wrappers.

I would like to give credits to awesome talk at LambdaConf'18 by @jdegoes about KleisliIO: https://www.youtube.com/watch?v=L8AEj6IRNEE. You can find benchmarks and some implementation details there. I went a bit further, and with John's approval ported KleisliIO to TypeScript with some minor enhancements like Bifunctor instance: https://github.com/YBogomolov/kleisli-ts

I should admit that after some time of using bifunctor IO<E, A> (IOEither, TaskEither in fp-ts, ZIO in Scala) I find it's expressiveness much bigger than plain ol' IO<A>. That's why `kleisli-ts` uses it as a main typeclass.

Let's look at the example. If you recall fifth and sixth episodes, we wrote a random generation app using Tagless Final and ZIO/TIO. Now we can write it using one more approach – with KleisliIO.

We start by obtaining an instance of KleisliIO API for the given monad, in this case – TaskEither:

```ts
import { taskEither, tryCatch, URI } from 'fp-ts/lib/TaskEither';
import { getInstancesFor, KleisliIO } from 'kleisli-ts/lib';

const K = getInstancesFor(taskEither);
```

The distinctive part here is that we describe our computation by composing functions as whole, not by looking at their input and output separately:

```ts
// Note usage of `ifThenElse` combinator here:
export const parse: KleisliIO<URI, Error, string, number> =
  K.ifThenElse<Error, string, number>
    (K.liftK((s: string) => {
      const i = +s;
      return isNaN(i) || i % 1 !== 0;
    }))
    (K.identity<Error, string>().chain((s) => K.fail(new Error(s + ' is not a number'))))
    (K.liftK(Number));

// We usually lift impure functions into Kleisli arrows via `liftK` or `impure`/`impureVoid`:
export const generateRandom: KleisliIO<URI, never, number, number> =
  K.impureVoid((lowerBound) => Math.floor(Math.random() * lowerBound + 1));

export const print: KleisliIO<URI, never, string, void> =
  K.impureVoid((message: string) => console.log(message));
```

As we have to deal with async I/O here, we can utilize convenient `pure` method, which works with pure monadic computations:

```ts
import { createInterface } from 'readline';

export const read: KleisliIO<URI, Error, void, string> =
  K.pure(
    () => tryCatch(() => new Promise<string>((resolve) => {
      const rl = createInterface({
        input: process.stdin,
        output: process.stdout,
      });
      rl.question('> ', (answer) => {
        rl.close();
        resolve(answer);
      });
    }), (e) => new Error(String(e))),
  );
```

Now interesting part: we can use chaining combinators like `andThen` or `chain` to stitch our functions together. Compare with combining pure total functions with `compose`!

```ts
export const getUpperStr: KleisliIO<URI, Error, void, string> =
  K.of<Error, void, string>('Enter random upper bound:')
    .andThen(print)
    .andThen(read);

export const checkContinue: KleisliIO<URI, Error, void, boolean> =
  K.of<Error, void, string>('Do you want to continue?')
    .andThen(print)
    .andThen(read)
    .chain((answer) => {
      switch (answer.toLowerCase()) {
        case 'y':
          return K.of(true);
        case 'n':
          return K.of(false);
        default:
          return checkContinue;
      }
    });
```

And finally we can write and run our main program:

```ts
import { unsafeRunTE } from 'kleisli-ts/lib/unsafe';

export const main: KleisliIO<URI, Error, void, void> =
  getUpperStr
    .andThen(parse)
    .andThen(generateRandom)
    .chain((rnd) => K.of<Error, void, string>(`Your random is: ${rnd}`).andThen(print))
    .andThen(checkContinue)
    .chain((answer) => answer ? main : K.of<Error, void, string>('Good-bye').andThen(print));

unsafeRunTE(main.run());
```

This concludes my introduction of Kleisli arrows.

That's all, folks. Hope you liked this episode! And stay tuned for July's compilation and next episodes ;)  
As usual, all code examples are available at https://github.com/YBogomolov/monadic-mondays
