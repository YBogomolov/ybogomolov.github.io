---
layout: post
title: "Functional Programming Jargon, Part 1"
author: yuriy
categories: [ fp, typescript, basics ]
image: assets/images/fp-jargon-part-1.jpeg
---

Functional programming is infamously known for its cryptic math-like jargon: terms like monads, monoids, functors and isomorphisms seem to be very intimidating for inexperienced developers. But if we take a look at those concepts as programming patterns, everything becomes much clearer. In the first part of the series, I will take a closer look at the most common terms from FP jargon.

<!--more-->

Functional programming is infamously known for its cryptic math-like jargon: terms like monads, monoids, functors and isomorphisms seem to be very intimidating for inexperienced developers. But if we take a look at those concepts as programming patterns, everything becomes much clearer. Let's take a look at those terms which you may encounter when using FP libraries like `fp-ts`, `effect-ts` or `sanctuary`, and code examples which illustrate them.

# Algebraic Data Type (ADT)

We start with _algebraic data types_. A data type is called _algebraic_ if it's composed of _product types_ (interfaces in TypeScript terminology) and _sum types_ (unions, enumerations). Here's an example of a small ADT:

```ts
// Product types — interfaces:
interface Nothing {
  readonly tag: 'Nothing';
}

interface Just<T> {
  readonly tag: 'Just';
  readonly value: T;
}

// Sum type — union:
type Maybe<T> = Nothing | Just<T>;
```

Algebraic data types got their name due to their mathematical properties, and it relates to _counting type inhabitants_. Product types — interfaces, or records, — got their name because the number of possible type inhabitants is equal to the _product_ of inhabitants of each field. Say, type `{ foo: boolean, bar: boolean | null }` will have a number of its inhabitants equal to 6 — 2 for `foo` times 3 for `bar`. The number of inhabitants for _sum types_, as you probably have already guessed, is equal to the _sum_ of inhabitants for each union member. Type `boolean | null` will have exactly 3 inhabitants: 2 for `boolean` plus 1 for `null`.

> These properties preserve even further. Here's a riddle for you: can you guess which data type corresponds to an exponent `A^B`?

Algebraic data types are the core tool of data modelling. The ultimate goal of data modelling in FP is to write so precise types that it's impossible to construct _bad data_. This is known as the principle of [making illegal states unrepresentable](https://ybogomolov.me/making-illegal-states-unrepresentable), and I talked about it in my previous article.

# Laws

A special term you will encounter quite often is _law_. Law is a special property which should hold for some type. For example, `Array.prototype.reverse` should hold a "double application identity law": `arr.reverse().reverse()` should be structurally equal to just `arr`, i.e. all elements should return to their places after a second reverse.

> When you write your own functional structures and want to test that it holds some laws, it is good to use some property-based testing library to test your program on myriad of randomised inputs — for free! Are you interested in learning more about this kind of testing? Then keep an eye on this blog! ;-)

# Domain and codomain of a function

A _domain_ of a function is a type of its argument. A _codomain_, correspondingly, is a type of its result. An example: for `f: (x: number) => string` its domain will be `number`, and its codomain will be `string`.

# Injection, surjection, bijection

These terms are rate, but they are tightly tied to the terms "domain" and "codomain", and you still can meet them in some discussions, so I decided to showcase them as well.

A function is _injective_ if each of the elements from its domain is uniquely mapped to an element from its codomain. To put it simply, if no two elements from the domain are mapped to the same element from the codomain, then the function is injective (or "is an injection"):

```ts
const injection = (x: number): string => x.toString(); // no different two numbers are mapped to a same string

const notInjection = (x: number): number => Math.abs(x); // e.g., both 2 and -2 are mapped to 2
```

A function is _surjective_ if it fully covers all possible values of its codomain, i.e. there's no such `x` that `f(x)` is not defined:

```ts
const surjection = (x: number): number => x * x; // for each and every `x` there exists `x²`
```

Finally, a function which is _injective_ and _surjective_ simultaneously is called a _bijection_ (or "is bijective"). It is also known as "one-to-one correspondence" or "invertible function". We will meet such functions in this text — can you find them?

> What are `String.prototype.toUpperCase` and `String.prototype.toLowerCase`: injective, surjective, bijective, something else?

# Morphism, endomorphism, isomorphism

A _morphism_ is a term originated from Category Theory and applied to programming it means… just a function:

```ts
type Morphism<A, B> = (a: A) => B;
```

A special case of morphisms you may encounter is called _endomorphism_. It's a function from some type back to that exact type:

```ts
type Endomorphism<A> = (a: A) => A;
```

If you immediately thought of `identity` function, you are correct, but it's just a particular example of endomorphism. Functions like `String.prototype.toUpperCase`, or `Array.prototype.reverse` are examples of endomorphisms as well.

You may also meet the term "isomorphism", or the phrase "X isomorphic to Y". An _isomorphism_ is a function which has a _reverse_ operation. In TypeScript, isomorphism is usually modelled using a pair of functions, which hold together a law of information preserving:

```ts
interface Isomorphism<A, B> {
  readonly to: (a: A) => B;
  readonly from: (b: B) => A;
}

// Law 1: `to(from(x))` is the same as `identity(x)` — or just `x`.
// Law 2: `from(to(x))` is the same as `identity(x)` — or just `x`, as well.
```

When you hear that "_X isomorphic to Y_", you can translate this as "_X is reversibly transformable into Y, and vice versa_". One of the core principles of isomorphism — it should not lose any information. 

> An exercise for the curious ones: can functions `String.toLowerCase` and `String.toUpperCase` form an isomorphism? What about `Date.prototype.toISOString` and `new Date` constructor?

# Natural transformation

A _natural transformation_ can be thought of as a function which can replace _type constructors_, and this time we will need to get some help from `fp-ts` and its notation of higher-order types. Please refer to [this article](https://ybogomolov.me/01-higher-kinded-types) to get a better understanding of how higher-order and higher-kinded types are implemented. The definition of a natural transformation can be written like this:

```ts
type NaturalTransformation<F, G> = <A>(fx: HKT<F, A>) => HKT<G, A>;

// or in imaginary "TypeScript with kinds" syntax:

type NaturalTransformation<F, G> = (fx: F<_>) => G<_>;
```

Let take a look at some examples. We can create a natural transformation between an array and a set:

```ts
const arrayToSet = <A>(arr: Array<A>): Set<A> => new Set(arr);
```

Note that `arrayToSet` will replace the type of _container_, but leave the type of _content_ intact.

> A fun example: define an isomorphism between some arbitrary type `T` and its lazy representation `Lazy<T>`, where `type Lazy<A> = () => A`. What if we swap the direction — can you define an isomorphism between `Lazy<T>` and `T`?

Another important note: our `arrayToSet` implementation **will** lose data, but note that the definition of a natural transformation said nothing about information preserving! But if we insist on adding a law of information preserving, we will eventually arrive at the definition of a _natural isomorphism_ — a reversible transformation between two type constructors:

```ts
interface NaturalIsomorphism<F, G> {
  readonly to: <A>(fx: HKT<F, A>) => HKT<G, A>;
  readonly from: <A>(fx: HKT<G, A>) => HKT<F, A>;
}

const arrayMapIsomorphism: NaturalIsomorphism<'Array', 'Map'> = {
  to: <A>(arr: Array<A>): Map<number, A> => new Map(arr.map((el, idx) => [idx, el])),
  from: <A>(map: Map<number, A>): Array<A> => Array.from(map.values()),
};
```

> Can you write a natural isomorphism between an `Array` and a `Set`? Which laws can you come up with for it?

# Functor

A _functor_ is also a term from Category Theory, and it means "structure-preserving mapping". Imagine `Array.prototype.map` function, or its equivalent for `Map` or `Tree`, plus add some laws, and you get yourself a functor! We cannot define a generic functor in pure TypeScript, but thanks to HKT notation of `fp-ts` we can write this definition:

```ts
interface Functor<F> {
  readonly map: <A, B>(f: (a: A) => B) => (fa: HKT<F, A>) => HKT<F, B>;

  // Law 1: `pipe(fa, map(identity))` is equal to just `fa`
  // Law 2: `pipe(fa, map(f), map(g))` is equal to `pipe(fa, map(compose(f, g)))`
}
```

> N.B.: I intentionally simplify these examples, because in `fp-ts` the `Functor` interface is defined for types of higher kinds — from 1-parameteric to 4-parametric, named `Functor1` to `Functor4`, correspondingly, plus one numberless which operates on `HKT` and not `KindX`. My examples show only interfaces with `HKT` to declutter the code.

For a higher-order type — like our `Maybe` example from the ADT section up above — we usually can define a functor:

```ts
// First we need to show fp-ts that Maybe is higher-order type:

declare 'fp-ts/HKT' {
  interface URItoKind<A> {
    readonly Maybe: Maybe<A>;
  }
}

const maybeFunctor: Functor<'Maybe'> = {
  map: <A, B>(f: (a: A) => B) => (maybeA: Maybe<A>): Maybe<B> => {
    switch (maybeA.tag) {
      case 'Nothing': return { tag: 'Nothing' };
      case 'Some':    return { tag: 'Some', value: f(maybeA.value) };
    }
  }
};

// Usage examples:

const someX: Maybe<number> = { tag: 'Some', value: 20 };
const someY: Maybe<number> = { tag: 'Nothing' };

const stringifyX = pipe(someX, maybeFunctor.map(n => n.toString())); // => { tag: 'Some', value: '20' }
const stringifyY = pipe(someY, maybeFunctor.map(n => n.toString())); // => { tag: 'Nothing' }
```

A functor is an example of a _type class_. I've talked about what is it and how to define them in [this article](https://ybogomolov.me/02-type-classes) — check it out!

---

This is the first article of the series. In the next instalment, we will get familiar with more complex terms — magmas, semigroups, monoids, rings and much more, so stay tuned!