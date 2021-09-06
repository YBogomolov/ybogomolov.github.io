---
layout: post
title:  "Comments as education"
author: yuriy
categories: [ fp, typescript, education ]
image: assets/images/2.jpg
---

A little note about comments in code as a tool for education.

<!--more-->

Recently I've added an `absurd` function in a project which doesn't use fp-ts _yet_. It looked like this:

```ts
/**
 * Surprisingly, `absurd` is a function which should *not* be called in a correctly typechecked code.
 * It is especially useful in combination with algebraic data types (ADTs) for exhaustiveness checks.
 * Consider an example:
 *
 * ```ts
 * type FooBarBaz =
 *   | { tag: 'foo', foo: string }
 *   | { tag: 'bar', bar: number }
 *   | { tag: 'baz', baz: boolean };
 *
 * const x: FooBarBaz = getFooBarBazSomehow();
 *
 * switch (x.tag) {
 *   case 'foo': return x.foo; // foo is accessible only here
 *   case 'bar': return x.bar; // bar is accessible only here
 *   case 'baz': return x.baz; // you get it — baz is available only here
 *   default: return absurd(x); // and here `x` is inferred to have type `never`, so we can call `absurd` with it.
 * }
 * ```
 *
 * Now let's imagine we need to add another branch into `FooBarBaz` type:
 *
 * ```ts
 * type FooBarBaz =
 *   | ...previous branches...
 *   | { tag: 'qux', qux: Date };
 * ```
 *
 * Now our switch won't compile, as in the `default` case variable `x` will be of type `{ tag: 'qux', qux: Date }`
 * instead of `never`, and the compiler will warn us about this. But if you suppress this warning and leave code as is,
 * `absurd` will blow up in your face at runtime, causing you to go and fix this missing case.
 *
 * From type-theoretic point of view, as `absurd` requires an argument of type `never` (which doesn't have any runtime
 * reprsentation), then it can produce value of arbitrary type. That's why it's type is so strange — `absurd` states
 * the following: "if you give me a `never`, I'll give you back anything you want". Satisfying this requirement is
 * effectively impossible, that's why such return type is allowed. And due to a simple fact that we cannot
 * summon anything out of thin air, two viable implementations of `absurd` would be:
 * 1. throw an exception;
 * 2. return its argument of type `never` (which is a subtype of everything, thus assignable to anything).
 *
 * I've chosen here to throw an exception and blow up the application to ensure that calling `absurd` won't be missed.
 *
 * @param x unreachable value
 */
export const absurd = <A>(x: never): A => {
  throw new Error(`Function "absurd" should not be reachable; instead it was called with value: ${JSON.stringify(x)}`);
};
```

I like to live by the stance that any person who reads my code should be able to learn something useful. Writing a comment like this is a matter of 10-15 minutes, but it helps a lot for educating junior developers — e.g., here they will learn a) about exhaustiveness checks by example, and b) about `never` type and its application in production code. Moreover, such mini-article with nice code highlight will be available at a hover away:  
![vscode hover above functionn called absurd](/assets/images/absurd-hover.png)

P.S. I wish TypeScript supported [literate programming](https://en.wikipedia.org/wiki/Literate_programming) natively.