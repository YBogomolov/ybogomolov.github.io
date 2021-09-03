---
layout: post
title:  "Intro to fp-ts: Higher-Kinded Types"
author: yuriy
categories: [ fp, typescript, higher-kinded ]
image: assets/images/17.jpg
---


I'm Yuriy Bogomolov, and you (probably) may know me from my work on the [#MonadicMondays](http://twitter.com/hashtag/monadicmonday) series on Twitter, by my [YouTube channel](https://youtube.com/c/Cronuscpp) or my articles on [Medium](https://medium.com/tag/monadicmonday/archive) or [dev.to](https://dev.to/ybogomolov). In the Russian-speaking segment of the Internet there is very little information on functional programming in TypeScript and one of the best ecosystems for this language — the [fp-ts](https://gcanti.github.io/fp-ts/) library, in whose ecosystem I've quite actively contributed some time ago. With this article I want to start a story about FP in TypeScript.

I don't think it will come as a revelation to anyone that TypeScript is one of the most popular strongly typed supersets of JS. After enabling the `strict` compilation mode and configuring the linter to prohibit usage of the `any` type, this language becomes suitable for industrial development in many areas — from CMS to banking and brokerage software. There were a few unofficial attempts to prove Turing completeness of the TypeScript type system, which allows application of the advanced type-level programming techniques to ensure the correctness of business logic by making illegal states unrepresentable.

All of the above gave rise to the creation of a wonderful library for functional programming for TypeScript — `fp-ts` by the Italian mathematician Giulio Canti. One of the first things a person comes across when wanting to master it is very specific type definitions like `Kind <URI, SomeType>` or `interface SomeKind <F extends URIS> {}`. In this article I want to lead the reader to an understanding of all these "difficulties" and show that in fact everything is very simple and understandable — you just have to start unwinding this puzzle.

# Higher-kinded types

When it comes to functional programming, JS developers _usually_ stop at composing pure functions and writing simple combinators. Few look into the territory of functional optics, and it is almost impossible to come across flirting with freemonadic APIs or recursion schemes. In fact, all these constructions are not overly complicated, and the type system greatly facilitates learning and understanding. TypeScript as a language has quite rich expressive capabilities, however, they have their own limit, which is inconvenient — the absence of kinds. To make it clearer, let's look at an example.

Let's take a look at the familiar and well-studied *array*. An array, like a list, is a data structure that expresses the idea of ​​non-determinism: it can store from 0 to N elements of a certain type A. Moreover, if we have a function of the form `A -> B`, we can “ask” this array to apply it by calling the `.map ()` method, yielding an array of the same size with elements of type B in the same order as in the original array:

```typescript
const as = [1, 2, 3, 4, 5, 6]; // as :: number[]
const f = (a: number): string => a.toString();

const bs = as.map(f); // bs :: string[]
console.log(bs); // => [ '1', '2', '3', '4', '5', '6' ]
```

Let's do a mental experiment. Let's move the `map` function from the array prototype into a separate interface. As a result, we get a higher-order function polymorphic by the type of the input and output types, which I will immediately make curried for ease of further reading:

```typescript
interface MappableArray {
  readonly map: <A, B>(f: (a: A) => B) => (as: A[]) => B[];
}
```

Everything seems to be fine. But if we continue our mental experiment and start looking at other data structures, we will very quickly realize that the `map` function can be implemented for a `Set`, or a hash table (`Map`), or a tree, or a stack, or... A lot of things, in general. Let's see how the signatures of the `map` functions for the mentioned data structures will change:

```typescript
type MapForSet   = <A, B>(f: (a: A) => B) => (as: Set<A>) => Set<B>;
type MapForMap   = <A, B>(f: (a: A) => B) => (as: Map<FixedKeyType, A>) => Map<FixedKeyType, B>;
type MapForTree  = <A, B>(f: (a: A) => B) => (as: Tree<A>) => Tree<B>;
type MapForStack = <A, B>(f: (a: A) => B) => (as: Stack<A>) => Stack<B>;
```

I think you have already seen the general pattern and are thinking: how can you abstract from the data structure and write a generalized interface `Mappable`? For such abstraction to be possible, it is necessary for the language to fully support higher-order kinds, i.e. to be able to abstract from type constructors. Translating into TypeScript terminology, you need to be able to write an interface that can accept other generic types as generic arguments:

```typescript
interface Mappable<F> {
  // Type 'F' is not generic. ts(2315)
  readonly map: <A, B>(f: (a: A) => B) => (as: F<A>) => F<B>;
}
```

Unfortunately, this code won't compile because TypeScript doesn't know that the `F` argument type must be generic. We cannot write Scala-like syntax `F<_>` or anything alike — the language simply does not have expressive means for this. Does this mean that you need to give up and duplicate the code? No, the wonderful academic article “[Lightweight higher-kinded polymorphism](https://www.cl.cam.ac.uk/~jdy22/papers/lightweight-higher-kinded-polymorphism.pdf)” comes to the rescue.

# Lightweight higher-kinded polymorphism

To emulate kind polymorphism in TypeScript, we use a technique called *defunctionalization*, a technique for translating higher-order programs into a first-order language. Simply put, function calls turn into calls to data constructors with arguments that match the function arguments. In the future, such constructors are pattern-matched and interpreted as a need arise. For those who want to dig deeper into the topic, I recommend the original article by John Reynolds "[Definitional interpreters for higher-order programming languages](https://surface.syr.edu/cgi/viewcontent.cgi?article=1012&context=lcsmith_other)". In the meantime, we'll see how this technique can be applied to emulate kinds.

So, we want to express the following idea: there is a generic type `Mappable`, which takes as an argument a certain type variable `F`, which itself is a _first-order type constructor_, that is, a generic type that takes an ordinary non-polymorphic type as an argument. By applying the defunctionalization technique, we will do the following:
1. Replace the variable type `F` with a _unique type identifier_ — a certain string literal that will unambiguously indicate which type constructor we want to call: 'Array', 'Promise', 'Set', 'Tree', and so on.
2. Create a utility type constructor `Kind<IdF, A>`, which will _represent_ a call of the `F` type as a generic with an argument of type `A`: `Kind<'F', A> ~ F<A>`.
3. To simplify the interpretation of the `Kind` constructors, we will create a set of dictionary types, which will store the relations between the _type identifier_ and the _polymorphic type itself_ — single such dictionary for all types of each arity.

Let's see how it looks in practice:

```typescript
interface URItoKind<A> {
  'Array': Array<A>;
} // a dictionary for 1-arity types: Array, Set, Tree, Promise, Maybe, Task...
interface URItoKind2<A, B> {
  'Map': Map<A, B>;
} // a dictionary for 2-arity types: Map, Either, Bifunctor...

type URIS = keyof URItoKind<unknown>; // sum type of names of all 1-arity types
type URIS2 = keyof URItoKind2<unknown, unknown>; // sum type of names of all 2-arity types
// and so on, as you desire

type Kind<F extends URIS, A> = URItoKind<A>[F];
type Kind2<F extends URIS2, A> = URItoKind2<A>[F];
// and so on
```

The only thing left to do is to give any programmer the opportunity to extend the `URItoKindN` dictionaries, and not rely on the authors of the library in which this technique is used. This is where a great TypeScript feature comes to the rescue — [module augmentation](https://www.typescriptlang.org/docs/handbook/declaration-merging.html#module-augmentation). With this feature it'll be enough for us to place the code with defunctionalized kinds in the main library, and from the custom code, the definition of a higher-order type will be simple:

```typescript
type Tree<A> = ...

declare module 'my-lib/path/to/uri-dictionaries' {
  interface URItoKind<A> {
    'Tree': Tree<A>;
  }
}

type Test1 = Kind<'Tree', string> // will be inferred as Tree<string>
```

# Back to Mappable

Now we can define our Mappable type — polymorphically for any 1-ary constructors, and implement instances of it for different data structures:

```typescript
interface Mappable<F extends URIS> {
  readonly map: <A, B>(f: (a: A) => B) => (as: Kind<F, A>) => Kind<F, B>;
}

const mappableArray: Mappable<'Array'> = {
  // here `as` will have type A[], without any menthioning of the utility type `Kind`:
  map: f => as => as.map(f)
};
const mappableSet: Mappable<'Set'> = {
  // a little bit unfair — you can make it more efficient by iterating over the iterator for the set manually,
  // but the purpose of this article is not to make the implementation as efficient as possible, but to explain the concept
  map: f => as => new Set(Array.from(as).map(f))
};
// here I will assume that Tree is a normal inductive type with two constructors: Leaf and Node,
// leaves store data, nodes store a set of subtrees:
const mappableTree: Mappable<'Tree'> = {
  map: f => as => {
    switch (true) {
      case as.tag === 'Leaf': return f(as.value);
      case as.tag === 'Node': return node(as.children.map(mappableTree.map(f)));
    }
  }
};
```

Finally, I can unmask the `Mappable` type and say that it is called `Functor`. The functor consists of the type `T` and the operation` fmap`, which allows using the function `A => B` to convert `T<A>` to `T<B>`. You can also say that the functor lifts the function `A => B` into some computational context `T` (this look will be very useful later when we talk about the Reader/Writer/State trinity).

# fp-ts ecosystem

Actually, the idea of ​​defunctionalization and lightweight polymorphism of higher-order genera has become the key for the [fp-ts](https://gcanti.github.io/fp-ts/) library. Giulio wrote a pragmatic and concise guide on how to define your higher-order types: https://gcanti.github.io/fp-ts/guides/HKT.html. Therefore, there is no need to apply defunctionalization in your programs every time — just install `fp-ts` and put type identifiers in the `URItoKind`/`URItoKind2`/`URItoKind3` dictionaries located in the `fp-ts/lib/HKT` module.

There are many great libraries in the [ecosystem](https://gcanti.github.io/fp-ts/ecosystem/) of `fp-ts`:
* [io-ts](https://github.com/gcanti/io-ts) — a library for writing runtime type validators with a syntax that is as close as possible to the syntax of TS types
* [parser-ts](https://github.com/gcanti/parser-ts) — a library of parser combinators, a kind of minimal `parsec`/`megaparsec`/`attoparsec`
* [monocle-ts](https://github.com/gcanti/monocle-ts) — a library for functional optics, port of the `monocle` Scala library
* [remote-data-ts](https://github.com/devex-web-frontend/remote-data-ts) — a library with the `RemoteData` container type that greatly simplifies secure data processing on the front-end
* [retry-ts](https://github.com/gcanti/retry-ts) — a library with combinators of different strategies for retrying monadic operations
* [elm-ts](https://github.com/gcanti/elm-ts) — micro-framework for programming in the style of Elm Architecture using TS
* [waveguide](https://github.com/rzeigler/waveguide), [matechs-effect](https://github.com/mikearnaldi/matechs-effect) — very powerful algebraic effects systems for TS inspired by [ZIO](https://zio.dev)

And a few my libraries for `fp-ts` ecosystem:
* [circuit-breaker-monad](https://github.com/YBogomolov/circuit-breaker-monad) — Circuit Breaker pattern with monadic interface
* [kleisli-ts](https://github.com/YBogomolov/kleisli-ts) — library for programming with Kleisli arrows, inspired by the early design of ZIO
* [fetcher-ts](https://github.com/YBogomolov/fetcher-ts) — wrapper around `fetch` that supports server response validation using io-ts types
* [alga-ts](https://github.com/algebraic-graphs/typescript) — port of a wonderful `alga` library for describing algebraic graphs to TS

---

This is where I would like to conclude the introduction. Please write in the comments how interesting this material is to you personally. I have already done several iterations of teaching this material, and each time I've found points that could be improved. Considering the technical advancement of Habr's audience, perhaps it makes no sense to explain technical things using Mappable/Chainable, etc., and rather call things by their proper names right away — functor, monad, applicative? Let's discuss, I will be glad to chat in the comments.
