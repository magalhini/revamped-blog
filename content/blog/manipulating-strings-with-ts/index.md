---
title: "Manipulating strings with TypeScript"
description: "Some tips on how to remove prefixes or suffixes with TypeScript to ensure safer types"
date: "2023-02-20T11:31:24.019Z"
categories:
  - Javascript

published: true

---
I've been trying to up my TypeScript skills up a few notches recently, since my new role at Mercury implies dealing with pretty advanced TypeScript types on a daily basis, and I've been pleasantly surprised at how much advanced type manipulation it is possible with more recent versions of TS.

String manipulation with TypeScript was something I never really thought about: you either had a `string` for a given type, or you had a union of `types` when you could afford to be specific. But did you know that you can tell TS exactly what do expect for strings, including manipulated versions of them?

This is incredibly useful especially when you're dealing with APIs: no more need to create separate types because the payload is lowercase, or uppercase, or underscored. You can just infer these new types with some string literal rules.

### What we'll build

Before diving into string literals, let's look at what the problem I was faced with at work. **I wanted to remove a particular sufix of every property of a type and create new types from that prefix removal**. If that doesn't make sense, let's have a look at what the API gave me:

```ts
interface TCheckPayload {
  mailedDate: string
  inTransitDate: string
  reroutedDate: string
  returnedToSenderDate: string
  deliveredDate: string
}
```

The API gave me the payload with `<status>Date` as its keys, but later on I wanted types without this `Date` keyword. What I wanted to derive from the previous payload, without having to type it out, was:

```ts
type TCheckStatus = "mailed" | "inTransit" | "rerouted" | "returnedToSender" | "delivered"

```

We'll learn how to build this automated type generation in a second. But first, a gentle introduction.


### Enter Template Literals

The terms you want to be aware for this are "Template Literals Types". The [official documentation on it](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html) makes a great introduction to these, so I won't duplicate their fantastic examples here. Instead, I'll give you a gentle introduction before solving the problem above.

You may already know the terms _Template Literals_ from their native JavaScript implementation, so you'll be glad to know they're one and the same: you use them the way you always have, but TypeScript is able to infer them as types. Here's a quick example from their documentation:

```ts
type World = "world";

type Greeting = `hello ${World}`;
// type Greeting = "hello world"
```

This means that if you try to assign this type to a string which is not "hello world", it will be able to infer the contents of said string and fail:

```ts
const myVar: Greeting = "hello worl"

// Error: type "hello worl" is not assignable to type "hello world"
```

### Unions

Unions and template literal types go hand-in-hand. This can help you enforce stricter types to define your models when you'd typically assign them to a `string` even if they follow a preditable pattern.

Consider the following types for a store with books and audiobooks:


```ts
// types.ts
export type TProductTypes = "book" | "audiobook"

// api.ts
import {TProductTypes} from './types.ts'

type TProductTypeID = `${TProductTypes}_id` // <-- magic

const apiResponse: Record<TProductTypeID, string> = {
  book_id: '4234',
  audiobook_id: '3212'
}
```

Did you notice how we used our already existing product types `TProductTypes` to construct the shape of the `API` response? We used that string union to create a string literal type, `TProductTypeID`, which will append `_id` to all of its strings, and then we used it to define that as the keys of our API response. No need to define `book_id` and `audiobook_id` by hand.


### Matching shapes

Another simple, yet powerful use case for string literals is to have them match a particular shape for URLs and navigation in your app. Let's say in your app you have a `navigateTo` function that takes in a URL, but you keep forgetting if you should prepend a forward slash (`/`) to it or not. Well, string literals to the rescue:


```ts
type TNavigateToUrl = `/${string}`

const navigateToUrl = (url: TNavigateToUrl) => {}

// These pass TS's assertions
navigateToUrl('/admin')
navigateToUrl('/')
navigateToUrl('/profile')

// These don't pass
navigateToUrl('admin')
navigateToUrl('profile')
```

Now you're telling TS that `navigateToUrl` not only expects a string, but it expects a string that starts with a forward slash.

### Objects derived from string unions

One more example before we get into the challenge I was tasked with earlier: let's go back to our `book` and `audioBook` product types, and let's say we want to always combine them with certain properties: we know that all of them are going go have an id, a name and an author when they're modelled.

We could type the entire shape by hand:

```ts
type TCatalogueRecord = {
  bookId: string,
  bookName: string,
  bookAuthor: string,
  audiobookId: string,
  // ...etc
}
```

... or we could make use of string literal types and make sure we don't forget a type, by having TS generate them for us!

```ts
type TProductTypes = "book" | "audiobook"

type TProductProperties = "Id" | "Name" | "Author"

// Create a string literal of ProductTypeProductProperty
type TCatalogueRecord = `${TProductTypes}${TProductProperties}`

// Consume said type to create a record
const catalogueRecord: Record<TCatalogueRecord, string> = {
  bookId: '12',
  bookName: 'Station Eleven',
  bookAuthor: 'Emily St. John Mandel',
  audiobookId: '11',
  audiobookName: 'Men Without Women',
  audiobookAuthor: 'Haruki Murakami'
}
```

Notice that we didn't have to write the full type ourselves: TypeScript was smart enough to infer that it should create a combination of _every_ product type for _every_ product property. The one liner `TCatalogueRecord` actually contains the definition for our 6 properties.

### Minor string adjustment types

And I hear you ask: "what if my product properties aren't capitalized like the ones above?" What if I already use those properties but I need them in lowercase form, then your entire `nameProperty` convention falls apart!"

Well, not really, because TypeScript has a `Capitalize` native function! So if you had the lowercased properties, you could still construct the same object:

```ts
type TProductProperties = "id" | "name" | "author"

type TCatalogueRecord = `${TProductTypes}${Capitalize<TProductProperties>}`

const catalogueRecord: Record<TCatalogueRecord, string> = {
    bookId: '12',
    bookName: 'Station Eleven',
    // ...etc
```

### The challenge

Back to the type challenge I wanted to solve at work: As a reminder, here's what I had from the API:

```ts
interface TCheckPayload {
  mailedDate: string
  inTransitDate: string
  reroutedDate: string
  returnedToSenderDate: string
  deliveredDate: string
}
```

And what I wanted to generate without having to type it:

```ts
type TCheckStatus = "mailed" | "inTransit" | "rerouted" | "returnedToSender" | "delivered"
```

I knew I wanted a helper type method to remove the suffix of a type for this. In this case, I wanted to remove the `Date` part of each key of those strings. So I named my helper `RemoveSuffix`:

```ts
// TDB
type RemoveSuffix = ???

// Its usage
type TCheckStatus = RemoveSuffix<keyof TCheckStatusPayload, 'Date'>
```

Here's the final definition of `RemoveSuffix`. It can look a little daunting, since the `infer` keyword is a difficult concept, but let's have a look at it first then break it down:

```ts
type RemoveSuffix<
    Key extends string,
    Suffix extends string
> = Key extends `${infer Prefix}${Suffix}` ? Prefix : never
```

Ok! So:

- `Key` refers to the object key we're passing in (like `mailedDate`)
- `Suffix` refers to what we want to extract from the string (`Date` in our case)
- `never` is a way of telling "whatever happened here, this should never happen"

`infer` is tricky and I recommend its own [separate reading](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html), but for now just know that it's used to "extract" the type when we're conditionally checking a type, which we are doing here. What this condition is essentially doing, in plain english, is this:

> When the object Key has a Prefix AND a suffix that matches our Suffix, return ONLY the Prefix, otherwise, "never"

And so, if we hover over the following usage:

```ts
type TCheckStatus = RemoveSuffix<keyof TCheckStatusPayload, 'Date'>
```
We see that we get what we wanted to have, the `Date` part of the string removed:

```ts
type TCheckStatus = "mailed" | "inTransit" | "rerouted" | "returnedToSender" | "delivered"
```

### Why bother with this, can't I just type them out by hand?

Of course you can, and very often I still do. But the beauty of having types that you don't repeat and duplicate often allow you to catch errors early when, say, another developer has just added a type of `signedDate` to the payload and forgot to update the list of statuses by not adding `signed`.
When we infer these types, TypeScript will know that something (`signed`) is missing now, because we're inferring everything from a single source of type truth.

There's so much more to learn about template literals and I highly encourage you to read the [official documentation on it](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html).