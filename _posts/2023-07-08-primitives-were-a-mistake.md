---
layout: post
title: "Primitives Were A Mistake"
author: yuriy
categories: [typescript, fp, data modelling]
image: assets/images/ropes.jpg
---

As developers, we deal with primitive data types — `string`, `number`, `boolean` — every day. But what if I tell you that in reality, you shouldn't be using them at all? 

Using primitive types for data modelling is a dangerous practice that should be avoided.

<!--more-->

Did you know that each time you write `field: string` or `field: number`, you're most probably making a mistake?

Seriously, in modern data modelling, there are so few cases for using the `string` type that in almost every case you're dealing with something else than a plain `string`.

— But hey, — you might ask, — how do I model user names? Or street addresses? Or various statuses? They _are_ strings, duh!

And you may be partially right. What if I tell you instead that deep down all this is a question of _semantics_? Primitive types, like `string`, `number`, or `boolean`, hold no additional semantics. They are _context-free_, thus they are overgeneralised. They merely describe _how data is stored_ — not _what it means_.

First, let's recall what a `string` is:
- it represents a sequence of characters;
- it can be empty (have zero characters);
- in Node.js, its size is limited by a V8 limit for a single object. Currently, it is around 2^29 characters — see [this V8 commit](https://github.com/v8/v8/commit/ea56bf5513d0cbd2a35a9035c5c2996272b8b728).

And... that's it? A `string` has no additional semantics attached, that's why it is so versatile and generic.

But data modelling is **all** about semantics. We model data to distinguish it from other data and to have some guarantees and invariants. Primitive types don't serve this purpose at all.

Consider this example:

```ts
interface CustomerOrder {
  readonly orderId: string;
  readonly customerId: string;
  readonly status: 'created' | 'processing' | 'shipped' | 'delivered' | 'cancelled';
  readonly items: OrderItem[];
  readonly totalAmount: number;
  readonly shippingAddress: ShippingAddress;
  readonly paymentMethod: PaymentMethod;
}

interface OrderItem {
  readonly productId: string;
  readonly quantity: number;
  readonly price: number;
}

interface ShippingAddress {
  readonly street: string;
  readonly city: string;
  readonly state: Option<string>;
  readonly country: string;
  readonly postalCode: string;
}

interface PaymentMethod {
  readonly cardNumber: string;
  readonly cardHolderName: string;
  readonly expiryMonth: number;
  readonly expiryYear: number;
  readonly cvv: string;
}
```

I won't be stopping for long at IDs of various kinds — I've described why they have to be modelled as branded/opaque types in [[https://ybogomolov.me/making-illegal-states-unrepresentable]]. Instead, let's look at other `string`s — like those in the `ShippingAddress` interface — and I'll explain why each of them is **not** a `string`:
- `street` is not a `string`, because street names cannot be empty (contain 0 characters), as well as they cannot be arbitrarily large (like "Lord Of The Rings" large). The longest street name in the world is **Laan van de landinrichtingscommissie Duiven-Westervoort** in Duiven, Netherlands. It has whopping 44 characters, but still, it is not _arbitrarily large_.
- `city` is not a `string` either, using the same arguments. The longest city name in the world is **Llanfairpwllgwyngyllgogerychwyrndrobwllllantysiliogogogoch**, or just Llanfair, in Wales. It is 58 characters long. Still not a `string`.
- `country` is even more interesting. As of 2023, there are 196 countries in the world, with the longest name belonging to the **United Kingdom of Great Britain and Northern Ireland** (52 characters), and the shortest name belonging to **Oman** with 4 characters. IMO, this is not a case for a `string` — it is a call for a literal union[^1].
- now, `postalCode` is definitely not a `string` — it has a special format, usually numeric, but in countries like UK or Canada they use alphanumeric postal codes. Still, it is pretty limited in length — the longest postal codes are in Iran and the USA (10 digits long).

So we can conclude that we need at least to have **size-limited** stings for data modelling.

Now, when we look at `number`s, we will have a similar perspective — some numbers should fall in a range, some should have an algebraic property (like be non-negative, or be rational, or be divisible by another number, etc.).

Unfortunately, the best we can do is to use some data-validation libraries like `@effect/schema`, `io-ts`, `zod`, `typebox`, or many, [many](https://github.com/akutruff/typescript-needs-types) others. I'll be using `@effect/schema`:

```ts
import { pipe } from '@effect/data/Function';
import * as Schema from '@effect/schema/Schema';

export const StreetName = pipe(Schema.string, Schema.minLength(1), Schema.maxLength(44), Schema.brand('StreetName'));
export type StreetName = Schema.To<typeof StreetName>;

export const CityName = pipe(Schema.string, Schema.minLength(1), Schema.maxLength(58), Schema.brand('CityName'));
export type CityName = Schema.To<typeof CityName>;

export const CountryName = pipe(
  Schema.union(
    Schema.literal('Afghanistan'),
    Schema.literal('Albania'),
    // ...193 more
    Schema.literal('Zimbabwe')
  )
);
export type CountryName = Schema.To<typeof CountryName>;

export const PostalCode = pipe(Schema.string, Schema.pattern(/[\w\d]+/), Schema.brand('PostalCode'));
export type PostalCode = Schema.To<typeof PostalCode>;

export const Price = pipe(Schema.number, Schema.nonNegative(), Schema.brand('Price'));
export type Price = Schema.To<typeof Price>;

export const TotalAmount = pipe(Schema.number, Schema.positive(), Schema.brand('TotalAmount'));
export type TotalAmount = Schema.To<typeof TotalAmount>;
```

And so on. As a rule of thumb, **wrap each primitive into its own distinct brand, and if it's possible — attach some extra validations**. Plus you shouldn't be creating instances of these branded types manually. Instead, use type constructors and/or parsers to make sure all your data is valid at the point where you actually work with it.

There are a few exceptions where I agree to model data as a primitive `string`:
- user-generated input — e.g., comments;
- arbitrary bits of text (placeholders, explanations, descriptions);

Probably, that's all. All other cases should be distinct enough to be modelled _at least_ using branded types. You don't want to mix and match different data bits in your code, trust me.

[^1]: Here I deliberately do not account for possible country renaming that still happens from time to time. In general, my point holds even w.r.t renaming.