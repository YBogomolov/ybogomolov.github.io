---
layout: post
title:  "#MonadicMonday compilation: May"
author: yuriy
categories: [ fp, typescript, monadic monday ]
image: assets/images/monadicmonday.jpg
---

In 2019 I started an activity in Twitter called #monadicmonday – each Monday I posted a thread about some FP stuff which is useful and is easy to start using right away. This is a compilation of the second month, May.

<!--more-->

# Episode 5: Tagless Final

Welcome to the fifth episode of #monadicmonday! Today we'll look at the famous Tagless Final pattern and its implementation in TypeScript.

If you haven't heard of Tagless Final style (also called "Finally Tagless Interpreters"), here's a brief recap.

Originally this pattern was introduced by Oleg Kiselyov et al. in his paper "Finally Tagless, Partially Evaluated: Tagless Staged Interpreters for Simpler Typed Languages": http://okmij.org/ftp/tagless-final/JFP.pdf

TF style is a general method of embedding domain-specific languages (DSL) in a typed functional language. It effectively separates eDSL statements from their interpreters, allowing building a robust extensible applications.

One of the most notable features of TF – it allows building different interpreters for the same eSDL, so you can run your production code on optimizing cache-enabled interpreter, while your tests could be ran using simple Identity interpreter.

Also note that one of functional Scala influencers @jdegoes deprecate TF style in favor of slightly different approach using horizontal effect rotation: https://skillsmatter.com/skillscasts/13247-scala-matters

IMO, TF style is a valid choice for many functional languages like TypeScript. So let's build a small program in this style!

Our program will do the simple thing: it will as a user about random upper bound and return a random number in range [0, bound).

In TypeScript we don't have proper support for higher-kinded types, so we need to use some hacks. So we start with defining an "HKT" synonym which will adhere to Functor and Monad interfaces:

```ts
import { Type, URIS } from 'fp-ts/lib/HKT';

export interface ProgramSyntax<F extends URIS, A> {
  map: <B>(f: (a: A) => B) => _<F, B>;
  chain: <B>(f: (a: A) => _<F, B>) => _<F, B>;
}

export type _<F extends URIS, A> = Type<F, A> & ProgramSyntax<F, A>;
```

Next we'll define a few algebras – sets of operations of type `A => F A`, which will be building blocks of our eDSL – i.e., expressions we can use to build the final program.

```ts
export interface Program<F extends URIS> {
  // Exit the program with some message or code
  terminate: <A>(a: A) => _<F, A>;
}

export interface Console<F extends URIS> {
  // Print a line on some kind of console:
  print: (message: string) => _<F, void>;
  // Read a line from console:
  read: _<F, string>;
}

export interface Random<F extends URIS> {
  // Get next number in [0, upper) bounds:
  nextInt: (upper: number) => _<F, number>;
}

export type Main<F extends URIS> = Program<F> & Console<F> & Random<F>;
```

Now, when we have these expressive powers, we can express our program. I think of the following logic flow:

1. App asks for a upper bound
2. App reads user input
3. If the input is not a number:
   1. App prints a warning
   2. App asks if the user wants to continue:
      1. If 'y' -> start over
      2. If 'n' -> print 'Good-bye' and terminate
4. If the input is a number:
   1. App generates a random int
   2. App prints this number
   3. App asks if the user wants to continue:
      1. If 'y' -> start over
      2. If 'n' -> print 'Good-bye' and terminate

I would like to split the program's logic in small functions to show the core idea of TF style: you can "summon" only those parts of functionality you really use!

```ts
const generateRandom = <F extends URIS>(F: Program<F> & Random<F>) =>
  (upper: number): _<F, number> => F.nextInt(upper);

const getUpperStr = <F extends URIS>(F: Program<F> & Console<F>): _<F, string> =>
  F.print('Enter random upper bound:')
    .chain(() => F.read);

const checkContinue = <F extends URIS>(F: Program<F> & Console<F>): _<F, boolean> =>
  F.print(`Do you want to continue?`)
    .chain(() => F.read)
    .chain((answer) => {
      switch (answer.toLowerCase()) {
        case 'y':
          return F.terminate(true);
        case 'n':
          return F.terminate(false);
        default:
          return checkContinue(F);
      }
    });
```

You may have noticed some "burden" of passing `F` instance over and over to various functions. This is a price we have to pay in TypeScript. In other languages like Scala such instances are passed implicitly, resulting in a much cleaner code.

With these building blocks we can finally express our main program:

```ts
const main = <F extends URIS>(F: Main<F>): _<F, void> =>
  getUpperStr(F).chain(
    (upper) => parse(upper).foldL(
      () => F.print(`"${upper}" is not an integer`),
      (upperN) => generateRandom(F)(upperN).chain(
        (rand) => F.print(`Your random is: ${rand}`),
      ),
    ).chain(() => checkContinue(F).chain(
      (answer) => answer ?
        main(F) :
        F.print('Good-bye').chain(
          () => F.terminate(undefined),
        ),
    )),
  );

// Helper: a pure function to parse a string to number
const parse = (s: string): Option<number> => {
  const i = Number(s);
  return isNaN(i) || i % 1 !== 0 ? none : some(i);
};
```

However, if you try to run this code, you'll get… nothing. Why? Because our `main` is just an expression – it's data needed to be interpreted! And here come the second half of TF style: interpreters.

As we are building a console application, let's use `Task` typeclass for expressing asynchronous operations. Here are instances of out algebras using Tasks:

```ts
import { log } from 'fp-ts/lib/Console';
import { randomInt } from 'fp-ts/lib/Random';
import { fromIO, Task, task, URI as TaskURI } from 'fp-ts/lib/Task';
import { createInterface } from 'readline';

const programTask: Program<TaskURI> = {
  terminate: task.of,
};

const randomTask: Random<TaskURI> = {
  nextInt: (upper) => fromIO(randomInt(0, upper)),
};

const consoleTask: Console<TaskURI> = {
  read: new Task(
    () => new Promise((resolve) => {
      const rl = createInterface({
        input: process.stdin,
        output: process.stdout,
      });
      rl.question('> ', (answer) => {
        rl.close();
        resolve(answer);
      });
    }),
  ),
  print: (message: string): Task<void> => fromIO(log(message)),
};
```

Finally, we can obtain our `mainTask` and run it:

```ts
const mainTask = main({
  ...programTask,
  ...randomTask,
  ...consoleTask,
});

mainTask.run();
```

Here's an example of program session:
```
 yuriy@MBP  ts-node ./src/monday/tagless.ts
Enter random upper bound:
> aaaa
"aaaa" is not an integer
Do you want to continue?
> y
Enter random upper bound:
> 23
Your random is: 4
Do you want to continue?
> n
Good-bye
```

However, our journey is not over yet. We can define a set of interpreters for `Identity` type which just do nothing and use hard-coded values – but allow us to debug the logic of our program on a fixed set of values:

```ts
import { identity, URI as IdURI } from 'fp-ts/lib/Identity';
const programId: Program<IdURI> = {
  terminate: identity.of,
};

const randomId: Random<IdURI> = {
  nextInt: (upper) => identity.of(upper),
};

const consoleId: Console<IdURI> = {
  print: (_message) => identity.of(undefined),
  read: identity.of('42'),
};

console.log(
  generateRandom({ ...programId, ...randomId })(
    +getUpperStr({ ...programId, ...consoleId }).fold((x) => x),
  ).toString(),
); // => new Identity(42)
```

Or we can use a `State` monad with a tuple of inputs, outputs and randoms – and use it for creating fully synchronous tests! Or we can, for example, define yet another set of algebra instances which "render" output as HTML and read input from dialog windows. Or we can introduce some kind of network interaction – say, log our data not onto terminal, but put it in LogStash. Or in Kafka. Or in any database.

This concludes my short intro to Tagless Final.

# Episode 6: ZIO in TypeScript

Welcome to the sixth episode of #monadicmonday! Today I will answer to the question from one of my followers: “Could you share an example of ZIO in Typescript?”.

For those of you who are not familiar what a ZIO is, let me explain it briefly.
ZIO is a trifunctor (i.e., it takes 3 type parameters and can map over them), and it encapsulates the following idea:
`ZIO[-R, +E, +A]` describes a computation which can read from an enviroment typed by `R`, can fail with error of type `E` or succeed with a value of type `A`.

Concuptually, ZIO is identical to a `Reader[R, Task[Either[E, A]]]`, but it bakes all effects in a single high-performant container – opposed to the way of using monadic transformers, familiar to most Haskell developers. You may read a bit more about such design decision in @jdegoes blog post: http://degoes.net/articles/rotating-effects

You may wonder why ZIO incorporates a `Reader` and not a `State`, and how to implement a mutable environment using it. I sugges reading great article by @snoyberg: https://www.fpcomplete.com/blog/2017/06/readert-design-pattern. It is written in Haskell, but let's be honest: Haskell is nowadays a "de-facto" FP language, so you should be familiar with it simple syntax.
In short, Reader lets you write concurrent programs and guarantees runtime immutability of a shared resource, which is a cornerstone of concurrent programming, while State/Writer have no such guarantees. 

Let's switch to a more familiar to my readers TypeScript syntax and try to replicate ZIO in it. I will call my execise `TIO` (T stands for TypeScript, obviously) to avoid name collisions.

We start with a typeclass definition:

```ts
declare module 'fp-ts/lib/HKT' {
  interface URI2HKT3<U, L, A> {
    // We need to have our R, E, A parameters 
    // called U, L, A in order for this HKT pattern to work.
    // It is required that all interface augmentations 
    // have the same parameters:
    TIO: TIO<U, L, A>;
  }
}

export const URI = 'TIO';
export type URI = typeof URI;
```

Next, we'll have to define our base class, `TIO`. In order for it to work fast, I should have written the implementation of `Reader<R, Task<Either<E, A>>>` manually. This is quite a tedious work, so I'll still use monad transformers to achieve the same interface:

```ts
import { Either } from 'fp-ts/lib/Either';
import * as readerT from 'fp-ts/lib/ReaderT';
import * as taskEither from 'fp-ts/lib/TaskEither';

import TaskEither = taskEither.TaskEither;

const readerTTaskEither = readerT.getReaderT2v(taskEither.taskEither);

export class TIO<R, E, A> {
  readonly _R!: R;
  readonly _E!: E;
  readonly _A!: A;
  readonly _URI!: URI;

  constructor(readonly value: (e: R) => TaskEither<E, A>) { }

  run(e: R): Promise<Either<E, A>> {
    return this.value(e).run();
  }

  chain<R1 extends R, E1, B>(f: (a: A) => TIO<R1, E1, B>): TIO<R1, E1, B>;
  chain<E1, B>(f: (a: A) => TIO<R, E1, B>): TIO<R, E1, B>;
  chain<B>(f: (a: A) => TIO<R, E, B>): TIO<R, E, B> {
    return new TIO(readerTTaskEither.chain(this.value, (a) => f(a).value));
  }
  
  // Some methods are skipped for readability
}

export const of = <R, E, A>(value: (e: R) => TaskEither<E, A>) => new TIO(value);
export const fromIO = <R, E, A>(a: io.IO<A>) => new TIO<R, E, A>(() => taskEither.fromIO(a));

export const tio = {
  of,
};
```

Finally, here are some synonims ZIO defines at its core:

```ts
export type UIO<A> = TIO<any, never, A>;
export type Task<A> = TIO<any, Error, A>;
export type IO<E, A> = TIO<any, E, A>;
```

Now we can express the same program I've used as a demo in the previous installment of MonadicMonday – a program which asks the user for upper bound for random generation, prints the resulting random number and asks if the user wants to continue. In order to show the power of ZIO as a pattern, I will change the logic by the slightliest bit: RNG will read the _lower_ bound from the environment instead of using hard-coded "1":

```ts
export interface MainEnvironment {
  lowerBound: number;
}

export const parse = (s: string): Option<number> => {
  const i = Number(s);
  return isNaN(i) || i % 1 !== 0 ? none : some(i);
};

// Take a note: `ask` just returns it's environment, 
// so we can `chain` it and gain access to the runtime value:
export const generateRandom = (upper: number): TIO<MainEnvironment, never, number> =>
  ask().chain(({ lowerBound }) => fromIO(randomInt(lowerBound, upper)));
```

Today the expressiveness of this program will be greater, as we don't have to carry around first parameter which determines runtime implementation of our program's interpreter. We still keep one of the main benefits of FP-styled program – algorythm as data – but we lose the ability to test the effects with an ease of Tagless Final approach.

```ts
import { ask, fromIO, Task, TIO, tio } from './tio';

export const terminate = tio.of;
export const print = (message: string): Task<void> => fromIO(log(message));
export const getUpperStr: Task<string> = print('Enter random upper bound:').chain(() => read);
export const checkContinue: Task<boolean> = print(`Do you want to continue?`)
  .chain(() => read)
  .chain((answer) => {
    switch (answer.toLowerCase()) {
      case 'y':
        return terminate(true);
      case 'n':
        return terminate(false);
      default:
        return checkContinue;
    }
  });

export const main: TIO<MainEnvironment, never, void> = getUpperStr.chain(
  (upper) => parse(upper).foldL(
    () => print(`"${upper}" is not an integer`),
    (upperN) => generateRandom(upperN).chain(
      (rand) => print(`Your random is: ${rand}`),
    ),
  ).chain(() => checkContinue.chain(
    (answer) => answer ? main : print('Good-bye').chain(() => terminate(undefined)),
  )),
);
```

We can run our program by supplying it with a correct environment:

```ts
import { main } from './main';

main.run({ lowerBound: 33 });
```

Scala's ZIO can be tested: http://degoes.net/articles/testable-zio. But in my opinion, testing programs in TF style is much simpler. In either approach you need to pass the mock object, but with TF you pass it (and carry around) to each method, and with ZIO you need to write your programs with explicit access to the environmental effects via `ZIO.access`/`ZIO.accessM`.

In this episode I won't be showing the testing approach to the ZIO style. John @jdegoes described this very well, so please go check out his article.

As a summary, let's briefly compare the TF and ZIO:

*Tagless Final*
Pros:
+ Explicit abstracting over effects;
+ Higher performance compared to MTL/Free monads style;
+ Good composability;
+ Ease of testing;
+ Is a pattern, rather a final implementation – so it is possible to write in TF style in many languages;
Cons:
- Very steep learning curve;
- Every bit of program needs to be in TF style;
- Effect stack my be VERY big in real apps;

*ZIO*
Pros:
+ High performance (see John's blog for details);
+ More approachable to the new developers (more shallow learning curve);
+ Quite good composability;
Cons:
- Harder to test (effects must be baked in the Reader's environment);
- Still is more an end-to-end implementation rather than a pattern, exclusive for Scala;

I guess it's better to compare TF with a ReaderTaskEither pattern (achievable via MTL style). There's a lot of talks and articles around this topics, so I won't be duplicating them here. Instead I would like to show you a good talk comparing TF vs. MTL vs. Bifunctor IO: https://www.youtube.com/watch?v=QM86Ab3lL20

As usual, you can find the code example for this episode on my GitHub: https://github.com/YBogomolov/monadic-mondays

# Episode 7: Functional optics

Welcome to the seventh episode of #monadicmonday! Today we'll talk a bit about functional optics: lenses, prisms, folds and traversals.

History of lenses started in 2009 with "The Essence of the Iterator Pattern" paper, which started a whole lot of cascading publications, leading to current `lens` package for Haskell and various implementations in different languages. I will use an awesome `monocle-ts` package from @GiulioCanti for this episode, but you may use any optics library you find suitable. The concepts are the same.

You can find more historical references at https://github.com/ekmett/lens/wiki/History-of-Lenses

It's a common practice that we work with immutable data structures when writing in functional style. It gives us incredible guarantees about our code, but also comes with a drawback: it's hard to update (rebuild) the deeply-nested data structure. So optics to the resque!

Functional optics gives you a way of working with a complicated immutable data structures (deeply nested, recursive, etc) without explicit need to maintain the structure by hand. Current generation of optics libraries heavily rely on profunctor encoding, but we will be using a much simpler encoding.

Consider an example in imperative style:

```ts
// We have an organizational data structure like this:
interface Organization {
  title: string;
  employees: Array< {
    name: string;
    age: number;
    car?: {
      model: string;
      plateNum: string;
    };
    partner?: {
      name: string;
      age: number;
    };
    projects: Array<{
      title: string;
      code: string;
      start: Date;
      end: Date;
    }>
  }>
}

// Say, you're changing internal project codes from lowercase to uppercase + prefix. 
// How would you do this imperatively?
function changeCodes(o: Organization): Organization {
  const newOrg: Organization = {
    ...o,
    employees: o.employees.map((emp) => ({
      ...emp,
      // Note: we need to preserve absense of `projects`, 
      // if it was absent in the first place:
      ...(emp.projects ? {
        projects: emp.projects.map((p) => ({
          ...p,
          code: 'MY-' + p.code.toLocaleUpperCase(),
        })),
      } : {}),
    })),
  };

  return newOrg;
}
```

I should say that this way is error-prone (for example, it's easy to make a mistake and forcefully create `projects` field for all employees – even if they hadn't had it), tedious and not composable. However, there's a way to solve this problem with ease.

> N.B. Bonus points are going to those who remembered the third episode and said that you can use a recursion scheme named "catamorphism" here (or simply `fold`). But today we take a look through optics at this problem.

The intuition behind optics comes straight from physics: each optic entity allows you to "focus" on some part(s) of your data structure and provide interface to interact with them.

Let's start with an `Iso`. "Iso" stands for "isomorphism", and is generally could be thought of as a pair of functions to "get" and "rebuild" the values (reverse the "get"). Please note that in general isomorphisms between any two types are not guaranteed:

```ts
interface Iso<S, A> {
  get: (s: S) => A;
  reverseGet: (a: A) => S;
}
```

One example I can think of is a pair of two "1/x" functions. Given an `y` value, "get" applies `x -> 1/x` and leaves you with `1/y`, and "reverseGet" rebuilds `y` from `1/y` applying `x -> 1/x` function again.
The other example may be a pair of "toLocaleLowerCase" and "toLocaleUpperCase" functions in JS – but if and only if we agree that an input to Iso must already be in uppercase. Law for `Iso` is the following:

```
reverseGet . get ≅ id
```

`Iso` as it is may seem to be pretty useless, because in general from any `A` you cannot obtain any `B`. But it gives a raise to the next optics – `Lens`.

`Lens<S, A>` is a pair of functions to "get" and "set" values in the structure. Please note that unlike `Iso`, `Lens` implies the structure itself is already defined, so you cannot "set" a deeply nested level if intermediate levels are missing. Lenses are great for working with product types (tuples & objects).

```ts
interface Lens<S, A> {
  get: (s: S) => A;
  set: (a: A) => (s: S) => S;
}
```

Consider an example:

```ts
interface Foo {
  bar: string;
  baz: {
    quux: number;
    fizz: {
      buzz: boolean;
    };
  };
}

const aFoo: Foo = { bar: 'bar', baz: { quux: 1, fizz: { buzz: true } } };

const buzzLens = Lens.fromPath<Foo>()(['baz', 'fizz', 'buzz']);

buzzLens.get(aFoo); // => true
buzzLens.set(false)(aFoo); // => { bar: 'bar', baz: { quux: 1, fizz: { buzz: false } } }
```

If you need to work with fields that may be missing, you'll need to use a `Prism` or `Optional`. They both are used for working with sum types: arrays, Options, Eithers, etc, and they are defined using a pair of functions:

```ts
interface Prism<S, A> {
  getOption: (s: S) => Option<A>;
  reverseGet: (a: A) => S;
}

interface Optional<S, A> {
  getOption: (s: S) => Option<A>;
  set: (a: A) => (s: S) => S;
}
```

If you are coming from an imperative programming background, you can think of Lens, Optional & Prism as of ways of overloading property accessors.

Another useful type of optics – `Fold`. It allows, given a Monoid instance for the data type `M`, to fold (reduce) its value into type `M`:

```ts
interface Fold<S, A> {
  foldMap: <M>(M: Monoid<M>) => 
    (f: (a: A) => M) => (s: S) => M;
}
```

`Traversal` is an optics which can traverse (what a surprise!) data structure and perform the following operations:

```ts
interface Traversal<S, A> {
  modifyF: <F>(F: Applicative<F>): (f: (a: A) => HKT<F, A>) => (s: S) => HKT<F, S>;
  modify(f: (a: A) => A): (s: S) => S;
  set(a: A): (s: S) => S;
  // `filter` allows narrowing a Traversal:
  filter(predicate: Predicate<A>): Traversal<S, A>;
}
```

Consider an example:

```ts
import { Tree, tree } from 'fp-ts/lib/Tree';

interface User {
  name: string;
  age: number;
}

const hierarchy = new Tree<User>({ name: 'Boss', age: 42 }, [
  new Tree({ name: 'Manager1', age: 35 }, [
    tree.of({ name: 'Emp1', age: 45 }),
    tree.of({ name: 'Emp2', age: 64 }),
  ]),
  new Tree({ name: 'Manager2', age: 52 }, [
    tree.of({ name: 'Emp3', age: 27 }),
    tree.of({ name: 'Emp4', age: 32 }),
    tree.of({ name: 'Emp5', age: 39 }),
  ]),
]);

const hierarchyTraversal = fromTraversable(tree)<User>();
console.dir(hierarchyTraversal.modify(
  (u) => ({ ...u, name: u.name.toLocaleUpperCase() }),
)(hierarchy), { depth: null }); // =>
// Tree {
//   value: { name: 'BOSS', age: 42 },
//   forest:
//    [ Tree {
//        value: { name: 'MANAGER1', age: 35 },
//        forest:
//         [ Tree { value: { name: 'EMP1', age: 45 }, forest: [] },
//           Tree { value: { name: 'EMP2', age: 64 }, forest: [] } ] },
//      Tree {
//        value: { name: 'MANAGER2', age: 52 },
//        forest:
//         [ Tree { value: { name: 'EMP3', age: 27 }, forest: [] },
//           Tree { value: { name: 'EMP4', age: 32 }, forest: [] },
//           Tree { value: { name: 'EMP5', age: 39 }, forest: [] } ] } ] }
```

The best thing about optics is that it is easily composed with each other. You can start with a Lens, compose it with a Prism to zoom deeper, then compose with a Traversal to walk the inner structure, and so on.

Let's take a look at another example: given an organizational structure, calculate average age of its members.

We start with or data type definition and example dataset:
```ts
import { Tree, tree } from 'fp-ts/lib/Tree';

interface User {
  name: string;
  age: number;
}

const hierarchy = new Tree<User>({ name: 'Boss', age: 42 }, [
  new Tree({ name: 'Manager1', age: 35 }, [
    tree.of({ name: 'Emp1', age: 45 }),
    tree.of({ name: 'Emp2', age: 64 }),
  ]),
  new Tree({ name: 'Manager2', age: 52 }, [
    tree.of({ name: 'Emp3', age: 27 }),
    tree.of({ name: 'Emp4', age: 32 }),
    tree.of({ name: 'Emp5', age: 39 }),
  ]),
]);
```

Now we can write the solution:

```ts
import { identity } from 'fp-ts/lib/function';
import { getTupleMonoid, monoidSum } from 'fp-ts/lib/Monoid';
import { fromFoldable, Lens } from 'monocle-ts';

const ageTupleLens = new Lens<User, [number, number]>(
  (u) => [u.age, 1],
  ([age]) => (u) => ({ ...u, age }),
);

const hierarchyFold = fromFoldable(tree)<User>().composeLens(ageTupleLens);

const tupleMonoid = getTupleMonoid(monoidSum, monoidSum);

// Note that we get an answer in a single traverse of the tree:
const [agesSum, agesCount] = hierarchyFold.foldMap(tupleMonoid)(identity)(hierarchy);
console.log(agesSum / agesCount); // => 42
```

If you want to dive into categorical view on the optics, you should read excellent article by @BartoszMilewski: https://bartoszmilewski.com/2017/07/07/profunctor-optics-the-categorical-view/
However, it's not an entry-level work, so be prepared for complicated categorical topics like profunctor, adjoints and Tambara modules.

# Episode 8: Free Monads

Welcome to the eighth episode of #monadicmonday! Today we'll talk about the topic you've requested: free monads.

Free monads is a way of describing eDSL and building expressions trees & interpreters for them. This may sound a lot like Tagless Final we've discussed in ep.5, but free monads have their own twist.

First of all, why this name – "free" monad? "Free" means "free of evaluation", like in "freedom" and not in "free beer". 
A free monad is a construction which allows you to build a monad from any functor. What's more important for us programmers, it also allows us to run it in a stack-safe way.

Categorically speaking, free monad is a left adjoint to a forgetful functor. A forgetful functor takes a monad and "forgets" its `of` and `chain` parts, keeping only `map`. It's left adjoint has its arrows reversed, so free monad is a construction which:
– take a functor;
- adds the pointed part (`of`);
- adds the monadic behavior (`chain`).

Using free monads we represend our computations as AST, with some expressions being terminal commands, and the others having sub-expressions as `next` part.

As usual, I will illustrate today's topic with a small example. Let's write a simple program which reads 2 lines from a text file and puts them onto console.

Programs written using free monads are just descriptions of computations, not the computations themselves. Our programs are values, and we need to interpret them to get things actually done.

Our domain model will be just `File` with a handle and `isOpen` attribute:

```ts
import fs from 'fs';

// Data structure we'll be working with:
interface File {
  handle: fs.promises.FileHandle;
  isOpen: boolean;
}
```

We start with a HKT `OpsF<A>`, which will be our free construction. It allows us to build our expression tree for our domain:

```ts
// Some boilerplate for higher-kinded types:
const OpsFURI = 'Ops';
type OpsFURI = typeof OpsFURI;

declare module 'fp-ts/lib/HKT' {
  interface URI2HKT<A> {
    Ops: OpsF<A>;
  }
}

// Our eDSL – building blocks of our small application:
type OpsF<A> = OpenFile<A> | ReadLine<A> | Log<A> | CloseFile<A>;

class OpenFile<A> {
  readonly _tag: 'OpenFile' = 'OpenFile';
  readonly _A!: A;
  readonly _URI!: OpsFURI;
  constructor(readonly name: string, readonly next: (file: File) => A) { }
}

class ReadLine<A> {
  readonly _tag: 'ReadLine' = 'ReadLine';
  readonly _A!: A;
  readonly _URI!: OpsFURI;
  constructor(readonly file: File, readonly next: (a: [string, File]) => A) { }
}

class Log<A> {
  readonly _tag: 'Log' = 'Log';
  readonly _A!: A;
  readonly _URI!: OpsFURI;
  constructor(readonly message: string, readonly next: () => A) { }
}

class CloseFile<A> {
  readonly _tag: 'CloseFile' = 'CloseFile';
  readonly _A!: A;
  readonly _URI!: OpsFURI;
  constructor(readonly file: File, readonly next: (file: File) => A) { }
}
```

Next, we want to have a way to use those building blocks, preferably enhanced with monadic interface. So we need to "lift" our pure data of type `OpsF` into a free monad world. Luckily, we have a function just for that – `liftF`:

```ts
import { liftF } from 'fp-ts/lib/Free';

// Helpers for building eDSL expressions.
// Note that in general we have a possibility to fiddle with values
// as they are getting returned in `next` part:
const open = (name: string) => liftF(new OpenFile(name, identity));
const readLine = (file: File) => liftF(new ReadLine(file, identity));
const log = (message: string) => liftF(new Log(message, () => void 0));
const closeFile = (file: File) => liftF(new CloseFile(file, identity));
```

Finally, we can build the program which expresses the logic of our domain:

```ts
// This is our program – just a value, nothing REALLY happens here,
// we just declare our intentions:
const program = open('./src/episode-8/file.csv')
  .chain((file) => log(`File is open: ${file.isOpen}`).chain(() => readLine(file)))
  .chain(([line, file]) => log(line).chain(() => readLine(file)))
  .chain(([line, file]) => log(line).chain(() => closeFile(file)))
  .chain((file) => log(`File is open: ${file.isOpen}`));
// => program :: Free<"Ops", void>
```

Seems really close to Tagless Final style – we've created an embedded DSL and expressed our intentions using its statements. We have our program as data, and during the next step we'll build some interpreters to run the actual code.

However, as I said earlier, there's a twist. Tagless Final is spinning around final algebras (i.e., set of operations of type `A => F<A>`), while Free Monads approach builds up expression trees using recursive data type `Free<F<A>>`, which serve as initial algebras.

You may read a bit more specialized categorical description of a free monads here: https://www.paolocapriotti.com/blog/2013/11/20/free-monads-part-1 (parts 2 and 3 are also available).

And, of course, I cannot recommend enough an awesome blog post by @BartoszMilewski: https://bartoszmilewski.com/2018/08/20/recursion-schemes-for-higher-algebras/. Must read!

Anyway, let's proceed with defining the interpreters for our program. As usual, I will define two of them: one for `Identity` monad, and the other – for `Task`:

```ts
import { Identity, identity as id } from 'fp-ts/lib/Identity';

const exaustive = (x: never): never => x;

// For sake of brevity I will use a "global state" here instead of `State` monad:
let position = 0;
const lines = ['first line', 'second line'];

const identityInterpreter = <A>(fa: OpsF<A>): Identity<A> => {
  switch (fa._tag) {
    case 'OpenFile':
      // I'm stubbing the `handle` with null here, as my Identity interpreter won't be using it:
      return id.of(fa.next({ isOpen: true, handle: null as unknown as fs.promises.FileHandle }));

    case 'ReadLine':
      return id.of(fa.next([lines[position++], fa.file]));

    case 'Log':
      return id.of(fa.next());

    case 'CloseFile':
      return id.of(fa.next({ isOpen: false, handle: null as unknown as fs.promises.FileHandle }));
  }

  // A little trick: if you add a call to `exaustive`,
  // the compiler will force you to use all possible `_tag` values in switch!
  return exaustive(fa);
};
```

Interpreter for `Task` will do some actual work using Node's filesystem API:

```ts
import { delay, Task, task } from 'fp-ts/lib/Task';
import fs from 'fs';

const taskInterpreter = <A>(fa: OpsF<A>): Task<A> => {
  switch (fa._tag) {
    case 'OpenFile':
      return new Task(
        () => fs.promises.open(fa.name, 'r')
          .then((handle) => fa.next({ isOpen: true, handle })),
      );

    case 'ReadLine':
      console.timeLog('TASK', 'Read line');
      return new Task(
        () => fa.file.handle.read(Buffer.from(new ArrayBuffer(24), 0, 24), 0, 24)
          .then(({ buffer }) => buffer.toString()),
      ).chain((line) => task.of(fa.next([line, fa.file])));

    case 'Log':
      console.log('>>>>>', fa.message);
      // Let's pretend that we're logging to DB here, hence the
      return delay(500, undefined).chain(() => {
        console.timeLog('TASK', 'Log');
        return task.of(fa.next());
      });

    case 'CloseFile':
      return new Task(() => fa.file.handle.close()).chain(() => {
        console.timeLog('TASK', 'Close file');
        return task.of(fa.next({ ...fa.file, isOpen: false }));
      });
  }

  return exaustive(fa);
};
```

When we run our interpreters using `foldFree` function, we'll see the following output (I've added some instrumentation so you could see actual duration):

```ts
console.log('Id interpreter');
const resId = foldFree(id)(identityInterpreter, program); // => Identity<void>
console.log(resId.value);

console.log('\nTask interpreter');
const resTask = foldFree(task)(taskInterpreter, program); // => Task<void>
console.time('TASK');
resTask.run().then(() => console.timeEnd('TASK'));

/*
> ts-node ./src/episode-8/free.ts

Id interpreter
undefined

Task interpreter
>>>>> File is open: true
TASK: 506.466ms Log
TASK: 506.756ms Read line
>>>>> this is our first line!

TASK: 1013.263ms Log
TASK: 1013.459ms Read line
>>>>> this is the second line

TASK: 1518.309ms Log
TASK: 1518.753ms Close file
>>>>> File is open: false
TASK: 2021.198ms Log
TASK: 2021.623ms
*/
```

Now let's speak a bit about stack safety I've mentioned earlier. When you're writing interpreters for free monads, it's very simple to cause stack overflow, as you are (de)composing possibly infinite recursive data structure. To help with solving this issue, a common technique called "trampoline" is used.
Basically, trampolining means executing recursive calls in a loop – i.e., doing tail-call optimization for the compiler. A general definition of trampolining function is such:

```ts
// Enter Trampoline!
type Trampoline<T> = T | (() => Trampoline<T>);

function trampoline<T>(firstResult: Trampoline<T>) {
  let result = firstResult;
  while (result instanceof Function) {
    result = result();
  }
  return result;
}
```

So let's take a look at another example. Let's hand-write simple `IO` monad and express with it infinite recursive loop:

```ts
// Free IO monad – representing generic computations:
const IOFURI = 'StackOps';
type IOFURI = typeof IOFURI;

type IO<A> = Return<A> | Suspend<A> | FlatMap<A>;

class Return<A> {
  readonly _tag: 'Return' = 'Return';
  readonly _A!: A;
  readonly _URI!: IOFURI;
  constructor(readonly a: A) { }
}

class Suspend<A> {
  readonly _tag: 'Suspend' = 'Suspend';
  readonly _A!: A;
  readonly _URI!: IOFURI;
  constructor(readonly resume: () => A) { }
}

class FlatMap<A> {
  readonly _tag: 'FlatMap' = 'FlatMap';
  readonly _A!: A;
  readonly _URI!: IOFURI;
  constructor(readonly fa: IO<A>, readonly f: (a: A) => IO<A>) { }
}

// Runs given `IO` forever:
const forever = <A>(a: IO<A>): IO<A> => new FlatMap(a, () => forever(a));

// Should log current timestamp infinitely:
const program = forever(new Suspend(() => console.log(Date.now())));
```

We can create a naïve implementation of `run` interpreter like this:

```ts
const run = <A>(fa: IO<A>): A => {
  switch (fa._tag) {
    case 'Return':
      return fa.a;
    case 'Suspend':
      return fa.resume();
    case 'FlatMap': {
      const x = fa.fa;
      const f = fa.f;
      switch (x._tag) {
        case 'Return':
          return run(f(x.a));
        case 'Suspend':
          return run(f(x.resume()));
        default:
        case 'FlatMap':
          const y = x.fa;
          const g = x.f;
          return run(new FlatMap(y, (a) => new FlatMap(g(a), f)));
      }
    }
  }

  return exaustive(fa);
};
```

However, when you try to run this example, it'll fail with:
```

RangeError: Maximum call stack size exceeded
    at RegExp.test (<anonymous>)
    at WriteStream.getColorDepth (internal/tty.js:128:22)
    at Console.(anonymous function) (console.js:183:16)
    at Console.(anonymous function) (console.js:190:40)
    at Console.log (console.js:202:31)
    at Suspend [as resume] (/Users/yuriy/Projects/monadic-mondays/src/episode-8/stack.ts:95:51)
    at run (/Users/yuriy/Projects/monadic-mondays/src/episode-8/stack.ts:80:27)
    at run (/Users/yuriy/Projects/monadic-mondays/src/episode-8/stack.ts:80:18)
    at run (/Users/yuriy/Projects/monadic-mondays/src/episode-8/stack.ts:80:18)
    at run (/Users/yuriy/Projects/monadic-mondays/src/episode-8/stack.ts:80:18)
```

So we need to use trampolining for this recursion Uroboros to unfold safely. We change `run` signature just a bit and return recursive calls as thunks:

```ts
const run = <A>(fa: IO<A>): Trampoline<A> => {
  switch (fa._tag) {
    case 'Return':
      return fa.a;
    case 'Suspend':
      return fa.resume();
    case 'FlatMap': {
      const x = fa.fa;
      const f = fa.f;
      switch (x._tag) {
        case 'Return':
          return () => run(f(x.a));
        case 'Suspend':
          return () => run(f(x.resume()));
        default:
        case 'FlatMap':
          const y = x.fa;
          const g = x.f;
          return () => run(new FlatMap(y, (a) => new FlatMap(g(a), f)));
      }
    }
  }

  return exaustive(fa);
};

// Now this won't fall and provide an infinite stream of Unix timestamps:
trampoline(run(program));
```

If you want to dive deeper into trampolining, I recommend reading these two great papers: 
http://functorial.com/stack-safety-for-free/index.pdf
http://blog.higher-order.com/assets/trampolines.pdf

As you've probably guessed, our hand-written `IO` is just a specialization of a `Free IO a` monad. I leave re-writing this example using `Free` from `fp-ts` to a curious reader.

I should admit that free monads in general are considered harmful due to heavy allocation on the stack. If you examine code more closely, you'll notice that Free regularly re-constructs its data structure, leading to overwhelming amount of allocations.
You can find more details in an article by Mark Karpov: https://markkarpov.com/post/free-monad-considered-harmful.html
So in general, IMO, it's better to use other stack-sparing techniques like ZIO (if you are writing Scala) and Tagless Final (otherwise).

This concludes my explanation of Free Monads and free constructions in general. As usual, all code examples are available at https://github.com/YBogomolov/monadic-mondays

# Episode 9: Type-Level Programming

Welcome to ninth episode of #monadicmonday! Today I would like to return to the roots of this series and provide you with a few "take and go" recepies for type-level programming in TypeScript.

Let's start with the basics. Type-level programming means that we express statements using types, and ask compiler to check them. In TypeScript, we use conditional types to infer our statements to be either `never` (bottom type) or some meaningful type.

The first useful operation is type-level `If` operator, which can be expressed like so:

```ts
type If<T, U, True, False> = [T] extends [U] ? True : False;
```

Let's write some tests for it! First of all, we need a way to run type-level assertions. We can use a specialized tool, `dtslint`, from Microsoft, but its setup is a bit painful, so let's stick with something simple like `jest`.

We start with an `assertType` function, which will compile only if its parameter is not `never`:

```ts
// Type-level assertions which is possible to compile 
// only if parameter is inferred to a non-bottom type:
const assertType = <T>(expect: [T] extends [never] ? never : T): T => expect;
```

Our test for `If` will look like this:

```ts
it('If<T, Eq, True, False>', () => {
  //                If<  A,       B,      T,    F  >;
  type Assertion1 = If<string, boolean, never, true>;
  expect(assertType<Assertion1>(true)).toBeTruthy();

  type Assertion2 = If<string, string, true, never>;
  expect(assertType<Assertion2>(true)).toBeTruthy();
});
```

Practically, I find myself mostly using checks against `never`, so let's define `IfDef` type operator and write tests for it:

```ts
type IfDef<T, True, False> = If<T, never, False, True>;

it('IfDef<T, True, False>', () => {
  type Assertion1 = IfDef<string, true, never>;
  expect(assertType<Assertion1>(true)).toBeTruthy();

  type Assertion2 = IfDef<string | never, true, never>; // a + 0 = a
  expect(assertType<Assertion2>(true)).toBeTruthy();

  type Assertion3 = IfDef<string & never, never, true>; // a * 0 = 0
  expect(assertType<Assertion3>(true)).toBeTruthy();
});
```

Another useful type operation is `OrElse`, which falls back to its second argument if the first is `never`:

```ts
type OrElse<MaybeNever, Fallback> = IfDef<MaybeNever, MaybeNever, Fallback>;

it('OrElse<A, B>', () => {
  type Assertion1 = OrElse<string, true>;
  expect(assertType<Assertion1>('some string')).toEqual('some string');

  type Assertion2 = OrElse<never, true>;
  expect(assertType<Assertion2>(true)).toBeTruthy();
});
```

Now let's define something more interesting. Say, let's check if two types intersect each other:

```ts
type Intersect<A extends {}, B extends {}> =
  Pick<A, Exclude<keyof A, Exclude<keyof A, keyof B>>> extends { [x: string]: never } & { [x: number]: never } ?
  never :
  Pick<A, Exclude<keyof A, Exclude<keyof A, keyof B>>>;

it('Intersect<A, B>', () => {
  interface A { foo: string; }
  interface B { bar: number; }
  interface C { foo: string; baz: boolean; }

  type AnB = Intersect<A, B>;
  type Assertion1 = IfDef<AnB, never, true>;
  expect(assertType<Assertion1>(true)).toBeTruthy();

  type AnC = Intersect<A, C>;
  type Assertion2 = If<AnC, { foo: string }, true, never>;
  expect(assertType<Assertion2>(true)).toBeTruthy();
});
```

Another great type operation I often use is `AtLeastOne`. It transforms a partial type into a union of types with one required field from the whole set:

```ts
type AtLeastOne<T, Keys extends keyof T = keyof T> = Partial<T> & { [K in Keys]: Required<Pick<T, K>> }[Keys];

it('AtLeastOne<T>', () => {
  interface T {
    foo?: string;
    bar?: number;
    baz?: boolean;
  }

  type ALOT = AtLeastOne<T>;
  type Assertion1 = If<ALOT, { foo: string } | { bar: number } | { baz: boolean }, true, never>;
  expect(assertType<Assertion1>(true)).toBeTruthy();
});
```

Finally, let's prohibit type from being widened:

```ts
type Exact<S extends {}, T extends S> = S & Record<Exclude<keyof T, keyof S>, never>;

it('Exact<S, T>', () => {
  interface A { foo: string; }
  interface B { foo: string; baz: boolean; }

  type Assertion1 = If<Exact<A, B>, A, true, never>;
  type Assertion2 = If<Exact<A, B>['baz'], never, true, never>;

  expect(assertType<Assertion1>(true)).toBeTruthy();
  expect(assertType<Assertion2>(true)).toBeTruthy();
});
```

Now I would like to show you how to use these type-level operations in your code. For example, you're writing a function which requires its arguments to never intersect:

```ts
const neverIntersect = <
  A extends {},
  B extends {},
  // Typelevel law: A and B should not intersect
  NeverIntersect = IfDef<Intersect<A, B>, never, {}>
>(a: A & NeverIntersect, b: B & NeverIntersect): A & B & NeverIntersect => ({ ...a, ...b });

it('neverIntersect', () => {
  interface A { foo: string; }
  interface B { bar: number; }

  const a1: A = { foo: 'foo' };
  const b1: B = { bar: 42 };

  const c1 = neverIntersect<A, B>(a1, b1);
  expect(c1).toStrictEqual({ foo: 'foo', bar: 42 });
  // Won't compile with: 
  // Argument of type 'A' is not assignable to parameter of type 'never'.ts(2345)
  // const c2 = neverIntersect<A, A>(a1, a1); 
});
```

And now let's have some type-level fun! As you (probably) know, TypeScript's type system is close to be Turing-complete:
https://github.com/Microsoft/TypeScript/issues/14833
So we can play a bit with it and see where it may lead us :)

Since TS@2.0 a lot happened to its inference engine, so it's hard to express recursive types. However, we still can, for example, describe a type-level list and reverse it. Or define a type-level Peano numbers. Let's dive in!

First of all, we need some Boolean values on type level – they are bread and butter of type-level checks:

```ts
type False = 'f';
type True = 't';
type Bool = False | True;
```

Next, let's write a definition of a heterogeneous list, which is either empty or consists of a head and a tail:

```ts
type HNil = { isNil: True; };
// I will be using `any` for head type, because making 
// HCons generic makes code really messy and cumbersome to read:
type HCons = { isNil: False; head: any; tail: HList; };
type HList = HNil | HCons;
```

Next, we'll need some "functions" to create and manipulate our lists:

```ts
type Cons<Head, Tail extends HList> = { isNil: False, head: Head, tail: Tail };
type Head<Xs extends HCons> = Xs['head'];
type Tail<Xs extends HCons> = Xs['tail'];
type IsNil<Xs extends HList> = Xs['isNil'];
```

Having these, we can finally write the `Reverse` type. To reverse a list, we take its head and append it to a reversed tail:

```ts
type Reverse<Xs extends HList> = Rev<Xs, HNil>;
type Rev<Xs extends HList, T extends HList> = {
  f: Xs extends HCons ? Rev<Tail<Xs>, Cons<Head<Xs>, T>> : HNil;
  t: T;
}[IsNil<Xs>];

type ABC = Cons<'a', Cons<'b', Cons<'c', HNil>>>;
type CBA = Reverse<ABC>; // => HCons<'c', HCons<'b', HCons<'a', HNil>>>
```

Using the exact same technique, you can define type-level Peano numbers:

```ts
type Zero = { isZero: True };
type Nat = Zero | { isZero: False, pred: Nat };

type Succ<N extends Nat> = { isZero: False, pred: N };
type Pred<N extends Succ<Nat>> = N['pred'];
type IsZero<N extends Nat> = N['isZero'];

type _0 = Zero;
type _1 = Succ<_0>;
type _2 = Succ<_1>;
type _3 = Succ<_2>;
```

And even combine these two types:

```ts
type UpTo3 = Cons<_0, Cons<_1, Cons<_2, Cons<_3, HNil>>>>;
type From3 = Reverse<UpTo3>; // => HCons<_3, HCons<_2, HCons<_1, HCons<_0, HNil>>>>
```

Curious reader may find familiar patterns – for example, `Nat` definition is almost the same we used in the third episode about recursion schemes: https://twitter.com/YuriyBogomolov/status/1117723946005213185

```ts
// Compare:
type Zero = { isZero: True };
type Nat = Zero | { isZero: False, pred: Nat };

// From ep.3:
export class Zero<A> {
  public value: never;
  readonly _tag: 'Zero' = 'Zero';
  readonly '_A': A;
  readonly '_URI': URI;
  private constructor() { }
}

export class Succ<A> {
  readonly _tag: 'Succ' = 'Succ';
  readonly '_A': A;
  readonly '_URI': URI;
  constructor(public value: A) { }
}

export type NatF<A> = Zero<A> | Succ<A>;
```

If we had proper recursive types in TypeScript, we could define recursion schemes on the type level. However, I see little to none practical day-to-day usage of such possiblity.

As a final note, I would like to reference a module with some neat (and practically useful!) type-level operations from @GiulioCanti: https://github.com/gcanti/typelevel-ts

Another great resource I've used during preparation of this episode is the following blog post in Japanese from @ryotakameoka:
https://ryota-ka.hatenablog.com/entry/2017/12/21/000000

As usual, all code examples are available at https://github.com/YBogomolov/monadic-mondays
