---
layout: post
title:  "Intro to fp-ts, part 2: Type Class Pattern"
author: yuriy
categories: [ fp, typescript, typeclass ]
image: assets/images/4.jpg
---

In the second post of the series "Intro to fp-ts" I'll explain "type class" pattern, and show how to apply it for ad-hoc polymorphism.

<!--more-->

Previous articles:
1. [Higher-Kinded Types](https://ybogomolov.me/01-higher-kinded-types/)

---

In my previous article I've shown how to emulate higher-kinded polymorphism in TypeScript using defunctionalisation technique. Now let's see what it brings to the table, and we begin with "type class" pattern.

Type class as a concept came from Haskell, and was introduced by [Philip Wadler and Stephen Blott](https://www.researchgate.net/publication/2710954_How_to_Make_Ad-Hoc_Polymorphism_Less_Ad_Hoc) in 1988 as an approach for ad-hoc polymorphism. The type class defines a set of typed functions and constraints which should exist for each type which belongs to this type class. It may sound a bit scary at first, but in reality type class is quite simple and elegant structure.

## What Is A Type Class

<details>
  <summary>A note for those who knows Haskell or Scala well</summary>
  
  In this article I'll give a simplified explanation of type class concept, without going into details about instances dictionary, instance conflict resolution or type inference rules, all of which occur in Haskell and Scala. But we're talking TypeScript (and JavaScript as its runtime) here, and must keep in mind that its type system is way more simpler than Haskell's or Scala's. It lacks implicit arguments passing to functions (except `this`), so everything below will resemble GHC Core language, where type classes are passed to functions as explicit arguments.
</details>

Consider, as an example, one of the simplest type classes, `Show`, that defines a cast to a string. It is defined in the module `fp-ts/Show`:

```typescript
interface Show<A> {
  readonly show: (a: A) => string;
}
```

This definition is read like this: _type `A` belongs to a class `Show` iff for `A` a function `show : (a: A) => string` is defined_.

The `Show` type class is implemented in the following manner:

```typescript
const showString: Show<string> = {
  show: s => JSON.stringify(s)
};

const showNumber: Show<number> = {
  show: n => n.toString()
};

// Suppose that there exists a type "user" with fields "name" and "age":
const showUser: Show<User> = {
  show: user => `User "${user.name}", ${user.age} years old`
};
```

All the power of type classes is in their composition. For example, we can easily write an implementation of the `Show` typeclass for a specific structure — say, a tuple — if we have a `Show` instance for the contents of that structure:

```typescript
// In this case usage of `any` is justified, as for type T we don't care
// about concrete typing of Show — it'll be specialised later using `infer`.
// This trick allows omitting all elements from T which are not instances
// of a Show:
const getShowTuple = <T extends Array<Show<any>>>(
  ...shows: T
): Show<{ [K in keyof T]: T[K] extends Show<infer A> ? A : never }> => ({
  show: t => `[${t.map((a, i) => shows[i].show(a)).join(', ')}]`
});
```

The use of type classes allows the use of the principle of least power — when a function requests from its arguments only the set of functionality that will be used by it. In TypeScript, structured typing makes this approach very organic, and the use of type classes allows you to develop this idea even further.

Let's look at another synthetic example — we need to write a function that converts its contents to strings for an arbitrary data structure. Thanks to a trick from the previous article, type classes can be written not only for specific types, but also for higher-order types. The `Mappable` type, aka `Functor`, is just an example of such a type class. The functor allows you to perform transformations while preserving the structure — for example, if we have a list, then the `map` operation will change the type of elements, but preserve the order in this list; if we have a tree, then `map` will keep the sequence of branches and nodes; if we have a hash table — `map` will keep the keys intact. The functor will allow us to solve the problem:

```typescript
import { Kind } from 'fp-ts/HKT';
import { Functor } from 'fp-ts/Functor';
import { Show } from 'fp-ts/Show';

const stringify = <F extends URIS, A>(F: Functor<F>, A: Show<A>) =>
  (structure: Kind<F, A>): Kind<F, string> => F.map(structure, A.show);
```

It would seem like a lot of "syntactic noise" and incomprehensible advantages, right? But don't be in a rush to raise an eyebrow with skepticism — let's see how much flexibility this approach provides.

Separating the interface of the type class from the concrete implementation allows you to write polymorphic code that will remain functional even if the data structure changes. Suppose you are writing a comment module for your blog, and in the first implementation you decide that a simple linear structure will satisfy your needs — so you decide to store the comments in a regular list:

```typescript
interface Comment {
  readonly author: string;
  readonly text: string;
  readonly createdAt: Date;
}

const comments: Comment[] = ...;

const renderComments = (comments: Comment[]): Component => <List>{comments.map(renderOneComment)}</List>;
const renderOneComment = (comment: Comment): Component => <ListItem>{comment.text} by {comment.author} at {comment.createdAt}</ListItem>
```

When you realise that it would be better to store comments in a tree and not in a list, you'll have to rewrite all places where `comments` collection is treated like a list.

Instead you can use the approach with type classes, and structure your code in a different manner:

```typescript
interface ToComponent<A> {
  readonly render: (element: A) => Component;
}

const commentToComponent: ToComponent<Comment> = {
  render: comment => <>{comment.text} by {comment.author} at {comment.createdAt}</>
};

const arrayToComponent = <A>(TCA: ToComponent<A>): ToComponent<A[]> => ({
  render: as => <List>{as.map(a => <ListItem>{TCA.render(a)}</ListItem>)}</List>
});

const treeToComponent = <A>(TCA: ToComponent<A>): ToComponent<Tree<A>> => ({
  render: treeA => <div class="node">
    {TCA.render(treeA.value)}
    <div class="inset-relative-to-parent">
      {treeA.children.map(treeToComponent(TCA).render)}
    </div>
  </div>
});

const renderComments = 
  <F extends URIS>(TCF: ToComponent<Kind<F, Comment>>) => 
    (comments: Kind<F, Comment>) => TCF.render(comments);

...

// somewhere in a parent component you just change this line:
const commentArray: Comment[] = getFlatComments();
renderComments(arrayToComponent(commentToComponent))(commentArray);
// ...to this line, leaving the rendering code intact:
const commentTree: Tree<Comment> = getCommentHierarchy();
renderComments(treeToComponent(commentToComponent))(commentTree);
```

In general, the use of type classes as a design pattern in TypeScript can be described as follows:
1. The functionality that can be generalized is taken out of the base data type into a separate interface, polymorphic by data type or container type.
2. Every function that wants to use this functionality "requests" the desired set of type classes as the first curried argument. This is done in order not to be tied to a specific instance of a type class — the result is a more flexible and testable solution.
3. To distinguish an instance of type classes from ordinary function arguments, it makes sense to name them in `UPPER_SNAKE_CASE`, so that their use is conspicuous against the `camelCase` background in the rest of the code. It is clear that this works well if you write idiomatically — if your code is `$tyled_like_php`, then you should come up with your own notation.

## Some useful type classes
## Некоторые полезные классы типов

The `fp-ts` library provides a lot of type classes that it makes sense to understand if you want to understand the approaches of mature FP.

### Functor (fp-ts/Functor)

The functor is defined by the operation `map: <A, B> (f: (a: A) => B) => (fa: F<A>) => F<B>`, which can be viewed from two points of view:
1. A functor for some computational context `F` knows how to apply a pure function `A => B` to the value of `F<A>` to get `F<B>`.
2. The functor can lift the pure function `A => B` into the computational context `F` in order to get the function `F<A> => F<B>`.

Both of these definitions are equivalent, but the first, in my experience, is easier for developers to perceive, and the second is closer to mathematicians. In any case, the essence is the same — *the functor allows you to change the data inside any context without changing the structure of this context*.

Each instance of a functor should obey two laws:
1. Identity preservation: `map(id) ≡ id`
2. Composition preservation: `map(compose(f, g)) ≡ compose(map(f), map(g))`

We have already encountered functors in the previous and this article, and their usefulness in general cannot be underestimated — functors give rise to a whole universe of other type classes, so if you know that you have, for example, an instance of a monad, then automatically you have an operation from the Functor `map` typeclass.

### Monad (fp-ts/Monad)

Oh, this awful monad, aka burrito, aka railway, aka monoid in the monoidal category of endofunctors. In fact, a monad is an extremely simple thing. Attention, now there will be the shortest monadic tutorial!

**The monad is determined by the rule "1-2-3": 1 type, 2 operations and 3 laws**

1. A monad can be defined for a higher-order type — say, for type constructors like Array, List, Tree, Option, Reader, etc. — to be short, for everything that we are used to see in a generic form.
2. A monad is defined by two operations, and one of two equivalent ways — the operations `chain` and` join` are expressed through each other, therefore, to describe the monad, only `of` and one of these two operations are enough:
   1. First way:
      ```ts
      of: <A> (value: A) => F<A>
      chain: <A, B> (f: (a: A) => F<B>) => (fa: F<A>) => F<B>
      ```
   2. Second way:
      ```ts
      of: <A> (value: A) => F<A>
      join: <A> (ffa: F<F<A>>) => F<A>
      ```
3. Finally, any monad must obey three laws:
    1. The law of identity on the left: `chain(f)(of(a)) ≡ f(a)`
    2. The law of identity on the right: `chain(of)(m) ≡ m`
    3. The law of associativity: `chain(g)(chain(f)(m)) ≡ chain(x => chain(g)(f(x)))(m)`

> Using Haskell syntax, these laws are expressed simpler: `of` is `pure`, and `chain` is an infix operator `>>=` called "bind":
> 1. Identity on the left: `pure a >>= f ≡ f a`
> 2. Identity on the right: `m >>= pure ≡ m`
> 3. Associativity: `(m >>= f) >>= g ≡ m >>= (\x -> f x >>= g)`

**That's it, the tutorial is over, thank you everyone, everyone is free**. Homework: write a monad instance for type `type Reader <R, A> = (env: R) => A`.

Knowing this definition, you can say that you know what a monad is. There is nothing mystical, nothing implicit and nothing sacred in them — they are just a type, two operations and three laws, period. The situation with laws in languages without dependent typing is somewhat complicated, so it makes sense to check them using property-based testing.

The monad expresses the idea of ​​*sequential computation*. Look carefully at the signature of the `chain` function: one of its arguments is a value of type `A` "packed" into the computational context `F`, and the other is a function that takes a pure value of type `A` and which returns a new computational context with a value of type `B`. And there is no other way to get a value of type `A` from an argument of type `F <A>` other than to process this computational context `F`. The simplest example of such behavior — if we have `Promise<A>`, then we can get a value of type `A` from there only by “waiting” for the promise to be resolved. Unfortunately, the promise itself does not correspond to the interface and behavior of the monad, but it can illustrate the concept of _sequence_ computation.

For convenient work with monadic chains in mature FP languages there is syntactic sugar — do notation, for-comprehension — but we have nothing like that in TypeScript. There are attempts to do something using generators, but the most type-safe option is `Do` syntax from `fp-ts`. In the next articles I will try to show its use.

### Monoid (fp-ts/Monoid)

The monoid consists of:
1. A neutral element, also called a unit: `empty: A`
2. Binary associative operation: `combine: (left: A, right: A) => A`

A monoid must also obey 3 laws:
1. Law of identity on the left: `combine(empty, x) ≡ x`
2. The law of identity on the right: `combine(x, empty) ≡ x`
3. The law of associativity: `combine(combine(x, y), z) ≡ combine(x, combine(y, z))`

What can a monoid be useful for? First of all — in places where we want to combine entities with each other, and there can be just a huge number of such places. I will not describe everything here, but instead I will offer you to watch the wonderful talk by Luka Jakobowitz “[Monoids, monoids, monoids](https://www.youtube.com/watch?v=pLpxRnAPteA)”. The talk is using Scala, but any engineer should grasp the essence easily enough — Luka has gave this talk not for the first time and communicates the idea perfectly well.

---

There are many more useful type classes — for example, Foldable and Traversable allow you to traverse data structures, applying a certain operation in a certain context at each step; Applicative (which I did not discuss in this article, but I will definitely return in the article on type-safe validation) allows you to apply a function in context to data in context; Task/TaskEither/Future allows you to replace chaotic promises with law-abiding synchronization primitives, and so on. But I cannot afford to inflate this article any further. Therefore, this is where I propose to end this article, and in the next one to talk about more specific and practically applicable type classes and approach the idea of ​functional effects.
