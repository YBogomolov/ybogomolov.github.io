---
layout: post
title:  "Higher-Kinded Data in TypeScript"
author: yuriy
categories: [ fp, typescript, ddd, higher-kinded data, hkd ]
image: assets/images/9.jpg
---

Higher-kinded data is an approach of making data types kind-polymorphic, which supercharges their usage scenarios! In this article I show how to encode HKD in TypeScript, and what benefits it brings to the table.

<!--more-->

# Data of Higher Kinds

This is a technique I've encountered in a [blogpost by Sandy Maguire](https://reasonablypolymorphic.com/blog/higher-kinded-data/). Its core idea is simple: make field types of an interface kind-polymorphic, so that you can obtain a concrete data type by substituting type constructor for the fields. Sounds complicated, but in reality its quite simple.

To illustrate this idea, I will use a type of order in some generic e-commerce solution:

```ts
type OrderId = string;

type OrderState = 'active' | 'deleted';

interface User {
  readonly name: string;
  readonly isActive: boolean;
}

interface Order {
  readonly id: OrderId;
  readonly issuer: User;
  readonly date: Date;
  readonly comment: string;
  readonly state: OrderState;
}
```

In order to make it higher-kinded, we need to make it accept a polymorphic kind (generic type) and apply it to types of fields. In an imaginary TypeScript syntax it may look like this:

```ts
interface Order<F<_>> { // ← we cannot write like this :(
  readonly id: F<OrderId>;
  readonly issuer: F<User>;
  readonly date: F<Date>;
  readonly comment: F<string>;
  readonly state: F<OrderState>;
}
```

In order to make this code compile, we need to apply a defunctionalization technique called "lightweight higher-kinded polymorphism" — I gave a description of it in my "[Intro to fp-ts, part 1](https://ybogomolov.me/01-higher-kinded-types)" article. So we turn `F<_>` into a type URI:

```ts
import type { Kind, URIS } from 'fp-ts/HKT';

interface OrderHKD<F extends URIS> {
  readonly id: Kind<F, OrderId>;
  readonly issuer: Kind<F, User>;
  readonly date: Kind<F, Date>;
  readonly comment: Kind<F, string>;
  readonly state: Kind<F, OrderState>;
}
```

In order to get the concrete types suitable for creating orders (where every field is defined), or updating (where fields are optional), we substitute `F` with a URI of `Identity`, `Option`, `Array`, etc., etc., etc.. For example, here's a POJO for the order:

```ts
import type { identity } from 'fp-ts';

type Order = OrderHKD<identity.URI>;
/*
Will be inferred as:
type Order = {
  readonly id: OrderId;
  readonly issuer: User;
  readonly date: Date;
  readonly comment: string;
  readonly state: OrderState;
}
*/
```

> N.B. Unlike Haskell, fp-ts's `Identity` doesn't need defining a type family in order to get rid of `Identity` wrapper.

You can use this type of order as a result of querying the database, or as a model for the UI, or for dosen of other use cases. Now let's imagine our users are filling the order fields on the front-end, and each field may be missing from the front-end request. We use `Option` for such scenarios, and our HKD-encoded order will allow us to easily get the model:

```ts
import type { option } from 'fp-ts';

type OrderOption = OrderHKD<option.URI>>;
/*
type OrderOption = {
  readonly id: option.Option<OrderId>;
  readonly issuer: option.Option<User>;
  readonly date: option.Option<Date>;
  readonly comment: option.Option<string>;
  readonly state: option.Option<OrderState>;
}
*/
```

We can use `OrderOption` not only for updates, but also for validating user's input — given an `OrderOption`, we can return `Option<Order>` if all fields are `Some`:

```ts
import { option, apply } from 'fp-ts';

const validateOrder = (inputOrder: OrderOption): option.Option<Order> =>
  pipe(inputOrder, apply.sequenceS(option.Apply));
```

<details>
  <summary>A short explanation of `sequenceS` and `sequenceT`</summary>
  
For those who hasn't worked with `sequenceS` and `sequenceT` functions from `fp-ts/Apply` module before, here's a short explanation. Both `sequenceX` functions work with a colleciton of wrapped into some computation context `F` values — `sequenceT` works with tuples, `sequenceS` works with records, — and return a combined value wrapped in the same context `F`:

```ts
// Again, in imaginary simplified syntax:
const sequenceT: <F, A, B, C, ...>(
  fa: F<A>, 
  fb: F<B>, 
  fc: F<C>, 
  ...
) => F<[A, B, C, ...]>;

const sequenceS: <F, A, B, C, ...>(
  fs: { [fieldA]: F<A>, [fieldB]: F<B>, [fieldC]: F<C>, ... }
) => F<{ [fieldA]: A, [fieldB]: B, [fieldC]: C, ... }>;
```

To put it even more simply, `sequenceT`/`sequenceS` "swap" the type of `F` and type of tuple/record inside out:
- we had a tuple of `F`s → we get a `F` of a tuple,
- we had a record of `F`s → we get a `F` of a record.
</details>
<br />

If we define our custom `Nullable` type and make it a higher-kinded type, we can get a type of order update, which is more familiar to non-FP developers:

```ts
type Nullable<A> = A | null | undefined;
type NullableURI = 'Nullable';

// We need to turn Nullable into a higher-kinded types first:
declare module 'fp-ts/HKT' {
  interface URItoKind<A> {
    [NullableURI]: Nullable<A>;
  }
}

type OrderPartial = OrderHKD<'Nullable'>>;
/*
type OrderPartial = {
  readonly id: OrderId | null | undefined;
  readonly issuer: User | null | undefined;
  readonly date: Date | null | undefined;
  readonly comment: string | null | undefined;
  readonly state: OrderState | null | undefined;
}
*/
```

We can work our way around data validation scenarios as well. Using the awesome [io-ts](https://github.com/gcanti/io-ts), we can write a function which will accept an unknown value, an order-shared validator object, and will return a `Validation<Order>` — either a set of validation errors, or a validated order.

To start, we should define a order validator:

```ts
import { identity } from 'fp-ts';
import * as t from 'io-ts';
import * as tt from 'io-ts-types';
import type * as iots from 'io-ts/Type';

type Order = OrderHKD<identity.URI>;
type OrderValidator = OrderHKD<iots.URI>;

const validator: OrderValidator = {
  id: t.string,
  issuer: t.interface({ name: t.string, isActive: t.boolean }),
  date: tt.date,
  comment: t.string,
  state: t.keyof({ active: null, deleted: null }),
};
```

Now we can pass some user input through this validator, and get the desired behaviour:

```ts
import { apply, either } from 'fp-ts';

const validateOrder = (maybeOrder: Order): t.Validation<Order> =>
  pipe(
    {
      id: validator.id.decode(maybeOrder.id),
      issuer: validator.issuer.decode(maybeOrder.issuer),
      date: validator.date.decode(maybeOrder.date),
      comment: validator.comment.decode(maybeOrder.comment),
      state: validator.state.decode(maybeOrder.state),
    },
    apply.sequenceS(either.Apply)
  );
```

Of course, the same effect could be achieved by writing an io-ts validator for `Order` itself:

```ts
const Order = t.interface({
  id: t.string,
  issuer: t.interface({ name: t.string, isActive: t.boolean }),
  date: tt.date,
  comment: t.string,
  state: t.keyof({ active: null, deleted: null }),
});

const validateOrder = Order.decode;
```

But having it like this will require keeping it in sync with our HKD-encoded type *manually*, which I recommend to avoid.

---

As a conclusion — higher-kinded data is a powerful domain-modelling tool, which allows defining the shape of data and reuse it in different scenarios: for CRUD operations, for validation, for any other types of processing.
