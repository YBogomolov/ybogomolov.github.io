---
layout: post
title:  "Compile-time validation of UUIDs"
author: yuriy
categories: [ typescript, uuid, type-level ]
image: assets/images/uuid_example.png
---

In this post I show how to use TypeScript literal string template types to get a compile-time UUID validation.

<!--more-->

# Type-level UUID numbers

It happens so that we can use recent TypeScript's type system improvements to get a compile-time validation of UUID number, including validation of correctness of its version. We start with defining a set of allowed symbols for both hex alphabet and (separately) for version:

```ts
type VersionChar =
    | '1' | '2' | '3' | '4' | '5';

type Char =
    | '0' | '1' | '2' | '3'
    | '4' | '5' | '6' | '7'
    | '8' | '9' | 'a' | 'b'
    | 'c' | 'd' | 'e' | 'f';
```

UUID format is the following: `xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx`, with `M` representing version (special case for 0, and actual versions are 1-5) and `N` representing variant. To encode this, let's create a helper which will check if a string `S` consists of `Char`s and has length of `Len` symbols.

Unfortunately, TypeScript's strings, even literal ones, don't have `length` as a constant — it's just a `number`. So we need to implement this check using a recursive type with an accumulator. 

I'm also defining a helper `Prev` to calculate previous number for a given one. It uses accessing tuple by index to get a *previous* number regarding the index — e.g., for `X = 5` it will get fifth element of a tuple, which is `4`. As the longest group of symbols in UUID has length of 12, so maximal number present in a tuple is `11`, and rest are `never`s:

```ts
type Prev<X extends number> =
    [never, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, ...never[]][X];

type HasLength<S extends string, Len extends number> = [Len] extends [0]
    ? (S extends '' ? true : never)
    : (S extends `${infer C}${infer Rest}`
        ? (C extends Char ? HasLength<Rest, Prev<Len>> : never)
        : never);
```

Now we can define helpers to check whether a string is a group of 4, 8, 12, a special case — version and 3 more bytes, and a special case of nil UUID:

```ts
type Char4<S extends string> = true extends HasLength<S, 4> ? S : never;
type Char8<S extends string> = true extends HasLength<S, 8> ? S : never;
type Char12<S extends string> = true extends HasLength<S, 12> ? S : never;

type VersionGroup<S extends string> = S extends `${infer Version}${infer Rest}`
    ? (Version extends VersionChar
        ? (true extends HasLength<Rest, 3> ? S : never)
        : never)
    : never;

type NilUUID = '00000000-0000-0000-0000-000000000000';
```

Finally, we can write a (rather lengthy and mouthful) definition of UUID number, using previously defined helpers:

```ts
type UUID<S extends string> = S extends NilUUID
    ? S
    : (S extends `${infer S8}-${infer S4_1}-${infer S4_2}-${infer S4_3}-${infer S12}`
        ? (S8 extends Char8<S8>
            ? (S4_1 extends Char4<S4_1>
                ? (S4_2 extends VersionGroup<S4_2>
                    ? (S4_3 extends Char4<S4_3>
                        ? (S12 extends Char12<S12>
                            ? S
                            : never)
                        : never)
                    : never)
                : never)
            : never)
        : never);
```

Time for some examples! Let's imagine we have a function which gets a user by its UUID. I don't really care about the implementation, so I'll just log that id. But you can clearly see that incorrect cases are rejected by the compiler, and correct ones are allowed:

![Type-level UUID example](assets/images/uuid_example.png)
