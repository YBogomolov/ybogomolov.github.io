---
layout: post
title:  "#MonadicMonday compilation: July"
author: yuriy
categories: [ fp, typescript, monadic monday ]
image: assets/images/monadicmonday.jpg
---

In 2019 I started an activity in Twitter called #monadicmonday – each Monday I posted a thread about some FP stuff which is useful and is easy to start using right away. This is a final compilation of the last month, July.

<!--more-->

# Episode 12: Futures

Welcome to twelfth episode of #monadicmonday! Today I want to talk about a Future – a concept of asynchronous lazy task, which is a far better replacement for an eager Promise. Due to their laziness, futures support cancellation, racing, parallelism and resource safety out of the box. The best library I've seen and used for Futures is [fluture-js](https://github.com/fluture-js/Fluture).

Fun fact: in Scala world `Future` is not a monad, but something akin to JS's `Promise` – i.e., eager referentially non-transparent construct. Instead, it's far better to use IO (cats), Task (monix & scalaz) or ZIO (zio) to achieve the described benefits of today's topic.

In `fp-ts` & StaticLand terminology, Future<T> could be described as Task<Either<Error, T>>. But there's much more to it than my simplification implies. So let's dive in!

You have quite a bunch of ways to create a Future. For full list of method go see https://github.com/fluture-js/Fluture/tree/11.x#creating-futures
Please note that due to TS's way of type inference I have to provide type arguments manually:

```ts
import * as F from 'fluture';

const f1 = F.of<Error, number>(42);
const f2 = F.node<Error, Buffer>((done) => fs.readFile('./foo.txt', done));
const f3 = F.attempt<Error, void>(() => fs.writeFileSync('./bar.txt', 'Hello!'));
const f4 = F.tryP<Error, string>(async () => {
  const file = await fs.promises.readFile('./foo.txt');
  const text = file.toString();
  await fetch('https://example.com', { method: 'POST', body: text });
  return text;
});
const f5 = F.after<Error, number>(500, 42);
```

Why Future is bright and cool? Well, first of all, it is a pure value – i.e., the Future will not be running until you call `fork` or `promise` methods:

```ts
const f3 = F.attempt(() => fs.writeFileSync('./bar.txt', 'Hello!')); // will not create a file...
f3.promise(); // ...up until this time!
f3.fork(console.error, console.log); // ...or this time :)
```

Second, Futures support cancellation, making timing out requests a breeze:

```ts
const fetchF = F.encaseP(fetch); // convert `fetch` into a Future-returning function

const getUserLogin = fetchF('https://api.github.com/users/YBogomolov')
  .chain((res) => F.tryP(() => res.json()))
  .map<string>((user) => user.login);

getUserLogin
  // all we need to do for introducing a timeout:
  .race(F.rejectAfter<Error, string>(1000)(new Error('timeout')))
  .fork(console.error, console.log);
```

Third, Futures are stack-safe:

```ts
const add100000 = function self(x: number): F.FutureInstance<never, number> {
  const mx = F.of<never, number>(x + 1);
  return x < 100000 ? mx.chain(self) : mx;
};

add100000(1).fork(console.error, console.log); // => 100001
```

Fourth, Futures are monads, bifunctors and alternatives, so you can use all the machinery you've used to:

```ts
getUserLogin // let's pretend we get a login somehow...
  .alt(F.of('admin')) // ...or fall back to 'admin'
  // then let's search for that user:
  .chain((login) => fetchF(`https://api.github.com/search/users?q=${login}`).chain((res) => F.tryP(() => res.json())))
  .chain((users) => users.total_count > 0 ? F.of(users.items[0]) : F.reject(new Error('not found')))
  // ...and get his/her repositories:
  .chain((user) => fetchF(user.repos_url).chain((res) => F.tryP(() => res.json())))
  // ...and simplify our container, throwing away redundant info:
  .bimap((reason: Error) => reason.toString(), (repos) => repos.length)
  .fork(console.error, console.log); // => 30
```

Finally, `fluture` has a nice touch – so kind of imitation for Haskell's `do` notation, available by names `do` and `go`. As a side note, I must admit that attempts to write types for its results are quite cumbersome, but TS can infer them flawlessly:

```ts
const calc200002 = F.go(function*() {
  const res: number = yield add100000(1);
  const res2: number = yield add100000(1);
  return res + res2;
});

calc200002.fork(console.error, console.log);
```

So as you can see, Futures are a very powerful functional concept, and working with them is really easy thanks to the awesome `fluture-js`! I encourage you to read its documentation and examples, and try it in your projects – it's battle-tested and highly performant: https://github.com/fluture-js/Fluture/tree/11.x

This concludes my short introduction to programming with Futures. As usual, all code examples are available at https://github.com/YBogomolov/monadic-mondays

# Episode 13: Property-based testing

Welcome to thirteenth episode of #monadicmonday! Today we'll talk a bit about property-based testing using the awesome library `fast-check`.

When you write a unit test, you'd normally do something like this:

```ts
import { expect } from 'chai';
import { left, right } from 'fp-ts/lib/Either';

import f from './function-to-test';

it('should return a Right<string> for given number', () => {
  const mockInput = 42;
  const mockOutput = 'foobar';

  expect(f(mockInput)).to.deep.equal(right(mockOutput));
});

it('should return a Left<Error> if input is not a number', () => {
  const mockInput = 'aaaa';
  f(mockInput).fold(
    (e) => expect(e)
      .to.be.an.instanceOf(Error)
      .and
      .to.have.property('message').include(mockInput),
    () => expect.fail('should not be Right'),
  );
});
```

However, this barely tests the logic of your module. You have to write a lot of unit tests to cover all boundaries, handle possible type errors and so on. For example, how many of English-speaking programmers test their string-accepting functions for Cyrillic or kanji inputs?

And if you wrote some tests, you know how difficult it is to prepare good test set for your mocks. Usually you end up with custome mock generators or a set of giant JSONs which contain pre-generated data. And still there's a chance that you've missed something.

So property-based testing to the rescue. Its idea is rather simple: we assert that some property of the tested algorithm holds, and the testing framework generates sample random data to prove or disprove this assertion.

For TypeScript & JavaScript there's a nice library called `fast-check`: https://github.com/dubzzz/fast-check. It integrates well with mocha, jest, ava, tape and jasmine. Let's see how we can write property-based tests.

We start with rather simple test: "check that any string contains itself". Something akin to Identity for strings:

```ts
import * as fc from 'fast-check';

describe('Identity laws for string', () => {
  it('any Unicode string should contain itself', () => {
  //runner:   property:   arbitrary:              actual assertion:
    fc.assert(fc.property(fc.fullUnicodeString(), (str) => str.includes(str)));
  });

  // this one has faulty logic, so it should fail and provide you with a counterexample:
  it('should fail with counterexample', () => {
    fc.assert(fc.property(fc.fullUnicodeString(), (str) => str.trim().includes(str)));
  });
});
```

Running these tests will give you a counterexample for the faulty logic, so you can use it to debug the test subject and fix the bug:

```
> jest ./src/episode-13

 FAIL  src/episode-13/fc.test.ts
  ● Property tests › Identity laws for string › should fail with counterexample

    Property failed after 17 tests
    { seed: -976548053, path: "16", endOnFailure: true }
    Counterexample: [" "]
    Shrunk 0 time(s)
    Got error: Property failed by returning false

    Hint: Enable verbose mode in order to have the list of all failing values encountered during the run

       9 | 
      10 |     it('should fail with counterexample', () => {
    > 11 |       fc.assert(fc.property(fc.fullUnicodeString(), (str) => str.trim().includes(str)));
         |          ^
      12 |     });
      13 |   });
      14 | 
```

I would not include the `fast-check` library in Monadic Monday episode if it didn't had some monads up its sleeves :)
Arbitraries are acutally monads, so you can use their sequential properties to chain several arbitraries and build a composite object.

Let's build a property test for this program: "if two users are older than 18 and have some common likes, pair them. Otherwise, report found errors".
The implementation will be as simple as:

```ts
type Like = 'cars' | 'cats' | 'football';

interface User {
  login: string;
  age: number;
  email: string;
  likes: Like[];
}

const validate = (user: User): Either<Error, User> =>
  user.age < 18 ? left(new Error(`User ${user.login} must be over 18`)) : right(user);

const match = (user1: User, user2: User): Either<Error, [User, User]> => {
  // if both users have at least something in common:
  if (user1.likes.some((like) => user2.likes.includes(like))) {
    return right([user1, user2]);
  }

  return left(new Error(`No common likes for ${user1.login} and ${user2.login}`));
};
```

Let's start with writing a test for validation. First, we need a custom Arbitrary – generator for the `User` type:

```ts
const userArbitrary = fc.unicodeString().chain(
  // any Unicode login:
  (login) => fc.integer().chain(
    // any age – from -2^32 to 2^32, you'll see why in a moment:
    (age) => fc.emailAddress().chain(
      // any email address:
      (email) => fc.subarray<Like>(['cars', 'cats', 'football']).map<User>(
        // and `likes` could only contain `Like` type:
        (likes) => ({ login, age, email, likes }),
      ),
    ),
  ),
);
```

Now we can check that `validate` function passes through only valid users, and reports age errors for invalid:

```ts
it('should validate a user', () => {
  fc.assert(fc.property(userArbitrary, (user) => validate(user).fold(
    (e) => {
      expect(e).to.be.an.instanceOf(Error).and.to.have.property('message').include(user.login);
    },
    (validatedUser) => {
      expect(validatedUser).to.deep.equal(user);
    },
  )));
});
```

Note that due to `userArbitrary` is able to generate users of any integer age, we can test `validate` on the full range of possible values.
Now we can write a test for `match` function, using precondition to filter the arbitraries:

```ts
it('should match two valid users', () => {
  fc.assert(fc.property(userArbitrary, userArbitrary, (user1, user2) => {
    fc.pre(user1.age >= 18 && user2.age >= 18);

    array.sequence(either)([validate(user1), validate(user2)]).fold(
      (vErr) => {
        expect(vErr.message.includes(user1.login) || vErr.message.includes(user2.login)).to.be.true;
      },
      ([u1, u2]) => match(u1, u2).fold(
        (pairError) => {
          expect(pairError.message.includes(user1.login) || pairError.message.includes(user2.login)).to.be.true;
        },
        (pair) => {
          expect(pair[0]).to.deep.equal(u1).and.deep.equal(user1);
          expect(pair[1]).to.deep.equal(u2).and.deep.equal(user2);
        },
      ),
    );
  }));
});
```

The real spot where property-based testing shines is when you need to check that your algorithm holds up to algebraic laws – like laws for Functor or Monad.
Say, we want to ensure that `Either` holds functorial laws. Note that I generate two random functions and verify that their composition is preserved:

```ts
import * as fc from 'fast-check';
import { left, right } from 'fp-ts/lib/Either';
import { compose, identity } from 'fp-ts/lib/function';

describe('Either', () => {
  describe('Functor laws', () => {
    it('should preserve identity morphism', () => {
      // for any `x`, `either(x) map id` is isomorphic to `either(x)`:
      fc.assert(fc.property(fc.anything(), (x) => {
        expect(right(x).map(identity)).toEqual(right(x));
        expect(left(x).map(identity)).toEqual(left(x));
      }));
    });

    // for any `x`, `either(x) map f map g` is isomorphic to `either(x) map (g ∘ f)`:
    it('should preserve composition of morphisms', () => {
      fc.assert(fc.property(fc.anything(), fc.func(fc.anything()), fc.func(fc.anything()), (x, f, g) => {
        const g_o_f = compose(g, f);

        expect(right(x).map(f).map(g)).toEqual(right(x).map(g_o_f));
        expect(left(x).map(f).map(g)).toEqual(left(x).map(g_o_f));
      }));
    });
  });
});
```

So there you have it, a bried introduction to property-based testing using `fast-check` (https://github.com/dubzzz/fast-check). I encourage you to read its documentation, which is thorough and nicely written.

This concludes my short introduction to property-based testing. As usual, all code examples are available at https://github.com/YBogomolov/monadic-mondays

# Episode 14: Final

Welcome to fourteenth episode of #monadicmonday! Today's episode will be very short, as I decided to finish the series for now.

First of all, I would like to thank all my readers and followers – your feedback and support was great, and I'm really astonished how many people had supported my efforts with likes & retweets.

I wanted Monadic Mondays to be a bite-sized pragmatic stories about useful monad-ish structures, tips and tricks which you can use in your day-to-day coding. These intentions meant that I needed to focus on quite simple things. On the other hand, I didn't want to start from the ground up and describe what a Monad is, or what an Applicative is, or what does it mean to "fold" something… And finally I was limited by TypeScript's type system, which was seriously nerfed down in v3 comparing to v2.

So basically up to now I covered almost all topics I wanted and I could express in TypeScript. I got into a writer's block, and need some time to recover. I plan to occasionally write an article or two, but most likely they will be published separately from this hashtag.

I want to express my sincere gratitude to people who inspired me and provided feedback about the episodes. This means a lot, and fills my heart with happiness.

See you around! :)
