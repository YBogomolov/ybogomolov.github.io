---
layout: post
title:  "Typesafe Frontend Development"
author: Yuriy
categories: [ fp, typescript, typelevel ]
image: assets/images/16.jpg
---

## Preface

Tony Hoare at QCon'09 [apologized](http://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare) for creating the most disasterous invention in computer science – `null reference exception`:

> I call it my billion-dollar mistake. It was the invention of the null reference in 1965. {...} This has led to innumerable errors, vulnerabilities, and system crashes, which have probably caused a billion dollars of pain and damage in the last forty years.

My daily job involves working with the language which has not one, but _two_ places of possible null refs – `null` and `undefined`. I'm talking about JavaScript. Although I do love this pet project of Brendan Eich for its sloping learning curve, expressiveness and rich tooling, I cannot allow myself or my colleagues to use it in its 'pure' form. The price of dealing with errors like `undefined is not a function`, `cannot call method 'foo' of null` and so on is just too high. I cannot risk my customer's money and suggest developing a product with such support costs.

As I am diving deeper and deeper into computer science, category theory and homotopy type theory, I want my code:

- to be reasonable about,
- to execute with runtime safety,
- to have good performance,
- and to be easy to refactor and support.

**Toxicity warning!** However, on the other hand, there exists free market and obvious lack of qualified specialists. I might be a little bit biased here, but, unfortunately, most _programmers_ are not _engineers_ and not even closely _mathematicians_. Still, I need to have a way of achieving my mission and spreading my vision via _creation_ of qualified resources from the market's raw material. I want programming world to be a better place.

Given those presuppositions, I have chosen strict functional programming as my preferred approach and TypeScript as my main language. It has best of the both worlds: it's quite popular (just read success stories from companies like Facebook), and has enough perks to satisfy my demands. At least, for now.

## What is type safety?

Speaking from a formal point of view, type safety is determined by two properties of the semantics of the programming language:

1. **(Type-)preservation** – "Well typedness" of programs remains invariant under the transition rules of the language.
2. **Progress** – A well typed (typable) program never gets into an undefined state where no further transitions are possible.

If translated from academic language, **type safety is making use of information about our data in the compile time to prevent errors in runtime**.

And the last point.

> – Snake, why are we still here? Just to suffer?..  
> Kazuhira Miller, _Metal Gear Solid V_

I am convinced that the developer who follows The Rules Of Type Safety should feel comfortable and use compiler as his best friend & main tool. But if the developer wants to write non-safe code, _he must suffer_. His "wrong" code should not compile, or his library design should not be easy to use with the rest of the system – which inevitably will lead to complaints from his colleagues, and then to heavy refactoring. 

Jokes aside, I really do want to achieve the state where incorrect programs could not be expressed at all. This involves usage of several techniques and patterns.

## Typesafe Patterns

> **N.B.** As I am using the incredible [fp-ts](https://github.com/gcanti/fp-ts) package from Giulio Canti, I would stick to using its terminology.

### Use `const` to allow easy reasoning

As you probably know, nowadays JavaScript has _three_ ways of declaring a variable: `var`, `let` and `const`. The former demanded from a programmer to keep in his head all hoisting rules, so in ES6 TC39 committee introduced `let` and `const` keywords, which – what a surprize! – don't hoist. From those two only `const` allows us to reason about our code with confidence. Let me explain with a quick example:

```ts
// Line 48:
let user = await getUser();
// somewhere around line 74:
user = await getAdmin();
// Line 153:
return user; // Oops! We think that we return a user here, but it's actually an admin
```

Can someone with enough confidence say that `user` variable wasn't reassigned between lines 48 and 153, without looking at those lines and following all method calls? Consider this instead:

```ts
// ...
// Line 48:
const user = await getUser();
// Line 74:
user = await getAdmin(); // TS2588: Cannot assign to 'user' because it is a constant.
// Line 153:
return user; // No way this will compile
```

Much better.

Rule of thumb: **always use `const`**.

### Avoid `null` and `undefined`

I started this article with a Tony Hoare's quote not just for the sake of appearing literate. I really, _really_ hate working with `null`s in JS and TS. Thankfully, functional programming has given us an awesome tool just for this case – an `Option` monad (also known as `Maybe`).

A quick example:

```ts
// Bad:
function bad(registry: Registry<Module>): TransformedModule[] | null {
  const modules = getModules(registry); // => null | Module[]

  if (modules.length === 0) { return null; }

  return modules.map(doStuffWithModules);
}

// Good:
function good(registry: Registry<Modules>): Option<TransformedModule[]> {
  const modulesO = getModulesO(registry); // => Option<Module[]>
  return modulesO.map((modules: Module[]) => modules.map(doStuffWithModules));
}
```

So a rule of thumb #2: **if you need to introduce an entity which value can be absent in any time of program's execution, use `Option` to encode this idea**.

### Avoid throwing exceptions and side effects

You know what's worse than returning a `null`? _Throwing an exception_. There's no way in 'traditional' JS/TS to mark a method with a warning that it may blow up. In plain ol' Java we have the `throws` keyword, but it leads to another set of issues and (in my opinion) should be banned in any production code as well.

So what do we do instead? Use an another tool from our bag of monads – `Either` monad and its successors, of course! It encodes a computation which may yield a business result or fail with a **recoverable** error. I cannot stress enough how this is important. If you are familiar with Scala ecosystem (and even if you're not), – go see an [incredible article by John De Goes](http://degoes.net/articles/bifunctor-io#types-of-errors) about bifunctor IO in ScalaZ. John describes motivation for this approach very well.

On the frontend side it's painful to introduce an `IO` monad, as the most of the frameworks/libraries like React/Angular/Vue are written in traditionalistic OOP-esque approach, and require quite a lot of boilerplate to introduce this concept. However, using an `Either` in any part of the code is completely unintrusive and requires no changes to the code of the framework or library.

> By the way, I would like to recommend a great module called [remote-data-ts](https://github.com/devex-web-frontend/remote-data-ts), which encodes the state of a network request. I've built an entire rendering system for my current customer using this approach, and developers are happy with it. Go check it out!

>> To be honest, I've created my own subtype of `RemoteData` called `ApiData`, which encodes __six__ states instead of just __four__, but this is a specific case of my customer and easily could be a topic for another full article.

### Compute as much as possible in compile time

This is one of my favorite topics in functional programming – typelevel coding! It means we encode our data types in such way that **they represent only valid states of the program**, and incorrect program just won't compile!

I often say that I use TDD – Type-Driven Development. It is obviously a joke, but it's only a half-joke. Typelevel programming forces you to think of values and types in the same way, and your coding starts with designing a flow of types – and when it's done, your program literally writes itself, as you just need to ensure that types match. Using this approach, you will no longer see the difference between a _function operating on values_ and a _generic type operating on type parameters_, which makes your reasoning even more clear and expressive.

Using this technique requires quite a lot from the type system of the chosen language. I still dream about times when I can write production code in dependently-typed languages like Agda or Coq. But TypeScript still has some aces up its sleeve – due to its structural type system we can express types like `NonEmptyList`:

```ts
type NonEmptyList<T> = T[] & { 0: T };
```

Of course, this doesn't make TypeScript a dependently-typed language, but it is something :)

So, let me share an example – it may be more clear than any description:

```ts
// Conditional: if `T` extends `U`, then returns `True` type, otherwise `False` type
type If<T, U, True, False> = [T] extends [U] ? True : False;

// If `T` is defined (not `never`), then resulting type is equivalent to `Yep`, otherwise to `Nope`.
type IfDef<T, Yep, Nope> = If<T, never, Nope, Yep>;

// Makes keys `K` required:
type With<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>> & { [P in K]-?: T[P] };

// For typelevel tests:
const assertType = <T>(expect: IfDef<T, T, never>): T => expect;

/*
 * Actual usage
 */

// Our tested type which makes its parts optional or required, depending on passed parameters:
type ComponentProps<Routes extends string, Permissions extends string> =
  IfDef<Routes, { routes: Record<Routes, Lazy<string>>; }, { routes?: never }> &
  IfDef<Permissions, { permissions: Record<Permissions, Lazy<boolean>>; }, { permissions?: never }>;

// Code of the test using `jest`:
type Routes = 'goHere' | 'orHere';
type Permissions = 'permissionToDo' | 'permissionToBe';
type Props1 = ComponentProps<Routes, Permissions>;
type Props2 = { foo: 'bar' | 'baz';  routes: Routes; permissions: Permissions };

type Assertion1 = If<Props2, Props1, true, never>;
expect(assertType<Assertion1>(true)).toBeTruthy();
```

### Finally Tagless, Partially Evaluated

This is just a design pattern, but its importance is incredibly high. Using tagless final encoding, the developer creates a custom [eDSL](https://wiki.haskell.org/Embedded_domain_specific_language), in which incorrect program states are impossible to express. If you want to dive deep in the math & original design, please refer to the [initial paper by Oleg Kiselyov et al.](http://okmij.org/ftp/tagless-final/JFP.pdf).

I created [a sample gist](https://gist.github.com/YBogomolov/03b742e68f35ee963d2097814200b269) of implementing TF in TypeScript, so go check it out for the details!

BTW, some people [suggested to call 2017 a year of Final Tagless](https://typelevel.org/blog/2017/12/27/optimizing-final-tagless.html), and this means a lot. However, every buzz thing sooner or later comes to its end. On 25th of February John De Goes will [postulate his vision of the next big thing in FP](https://skillsmatter.com/meetups/11945-scala-matters) in Scala. I'm quite excited to see the results.

## Conclusion

I hope my article gets you interested in a topic of functional programming and typelevel approach. It really broadens the horizons of any programmer, as well as enables businesses to have a reliable, performant, supportable software. I hope to see this approach being spread even further when WebAssembly becomes widely available, and we as frontend engineers will have more awesome tools at our disposal. Who knows, maybe, even dependently-typed languages :)

Please ping me back at [@ybogomolov](https://t.me/ybogomolov) in Telegram, [@YuriyBogomolov](https://twitter.com/YuriyBogomolov) in Twitter or via [yuriy.bogomolov@gmail.com](mailto:yuriy.bogomolov@gmail.com?subject=Typesafe%20Frontend%20Development%20feedback).
