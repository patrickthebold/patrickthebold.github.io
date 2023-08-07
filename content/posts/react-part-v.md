---
title: "How to Not Build a React App (Part V)"
subtitle: "Better Record Types"
date: 2023-06-18T22:16:09-04:00
draft: false
---

Back to: [Part I]({{< ref "how-to-not-build-a-react-app" >}} "Part I")

Meta: I'm going to start doing pull requests, please feel free to make comments on the PR. The [first PR](https://github.com/patrickthebold/mpd-client/pull/1) is kind of big, and this post is just a small [part](https://github.com/patrickthebold/mpd-client/pull/1/files#diff-0be871a483628d519358d5ba3c621e82882d8a0ae7e1f30551c248b97e0cfb30) of it.

I guess at this point the blog is like extended PR commentary, but let's see how that format works.

# Immutable.js Records

I immediately regretted my earlier decision to go with `immutable.js` over `immer`. Their [Records](https://immutable-js.com/docs/v4.3.0/Record/) do two things I find odd and annoying:

1. It takes two steps to construct an Record. First you provide defaults to get a "Factory", then you can construct objects.
1. If a property is not provided in the defaults you cannot add it later by setting it.

I'll copy their [example]() to illustrate the first point:
```ts
import type { RecordFactory, RecordOf } from 'immutable';

// Use RecordFactory<TProps> for defining new Record factory functions.
type Point3DProps = { x: number, y: number, z: number };
const defaultValues: Point3DProps = { x: 0, y: 0, z: 0 };
const makePoint3D: RecordFactory<Point3DProps> = Record(defaultValues);
export makePoint3D;

// Use RecordOf<T> for defining new instances of that Record.
export type Point3D = RecordOf<Point3DProps>;
const some3DPoint: Point3D = makePoint3D({ x: 10, y: 20, z: 30 });
```

Ok, so I don't want to give up and decided I can deal with it. The two main problems I had were

1. If a property is optional, you need to explicitly pass `undefined` in the defaults. Otherwise you can't set it later.
1. Some properties are not really optional, but don't have a sensible default. I want to require them to be passed into the factory function.

## Explicit Optional Properties.

For the first issue, recall that if an object has a property set to `undefined`, that is different from the property not being on the object at all. In typescript:

```ts
// Optional Property
type OptionalFoo = {
  foo?: string
}

// Required Property
type Foo = {
  foo: string | undefined
}
```

Here `Foo` requires the property `foo` to be set, but allows it to be an undefined value. It's fairly common for javascript code to not make 
a distinction between the two cases. And typescript (by default) will allow explicitly setting `undefined` for optional properties:
```ts
const x: OptionalFoo = {
    foo: undefined // This is OK by default
  }
```

You can turn off that behavior with [`exact-optional-property-types`](https://devblogs.microsoft.com/typescript/announcing-typescript-4-4/#exact-optional-property-types) compiler option. 

Of course for `Record`s in `immutable.js` the distinction is quite important. So the first typescript helper is to turn optional properties into, required properties with `| undefined`. (Note: This requires `exact-optional-property-types` to be set, for reasons I don't quite understand.)

```ts
type ExplicitUndefined<T extends object> = {
  [k in keyof T]-?: T extends Record<k, T[k]> ? T[k] : T[k] | undefined;
};
```

For the example above, `ExplicitUndefined<OptionalFoo> == Foo`. A few notes since this uses a few typescript features:

`T` is the object passed which potentially has some optional properties that we want to modify. Overall, this is a [mapped type](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html), `[k in keyof T]` loops over all the keys in `T`, the `-?` is a [mapping modifier](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html#mapping-modifiers) which removes the optionality of each property. At this point, this is the same as the [`Required`](https://www.typescriptlang.org/docs/handbook/utility-types.html#requiredtype) utility type. The remaining part is to find which properties of `T` were optional and add "`| undefined`". So we need a [conditional type](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html) and a way to check if a property is optional.

The `T extends Record<k, T[k]>` part is `true` whenever the property `k` is _not_ optional. `k` is the property in question, `Record<k, T[k]>` is (the type of) an object with one property `k` which is _not_ optional. if `k` were optional in the original `T`, then `T` would not extend `Record<k, T[k]>`, because if a type has a required property `k` you can't replace it with a type that has `k` optional.

## Properties with no default.

The next bit is even more of a hack, and we will lose some type safety. My state is the union of three records:
```ts
type State = ConnectedState | DisconnectedState | ConnectingState;
```
I'd like to keep the websocket on the connected and connecting states as a required field. But there's no globally good default value for a websocket. In retrospect, the best thing is probably to make a new factory every time we have a new websocket connection, then immediately create a record. Like:

```ts
Record({ws, other, fields})(/*pass in nothing to create a new record with defaults*/)
```

Instead, I allow the user to specify "`requiredKeys`". These will be required to be passed in the factory, and you must pass `undefined` as the default value. We lose type safety because the property will _not_ be optional (either with a `?:` or a `| undefined`) on the record, but because the default is `undefined` then that will be the value if the property is `removed` from the record.

In my case, I think it would be hard to `remove` the websocket by mistake, so I'm happy for forgo the type safety, to avoid needing to check if the websocket is actually present on the state object.

The whole code is:
```ts
import { type RecordOf, Record as iRecord } from "immutable";

type ExplicitUndefined<T extends object> = {
  [k in keyof T]-?: T extends Record<k, T[k]> ? T[k] : T[k] | undefined;
};

type SetUndefined<T extends object, requiredKeys> = {
  [k in keyof T]: k extends requiredKeys ? undefined : T[k];
};

type DefaultProps<T extends object, requiredKeys> = SetUndefined<
  ExplicitUndefined<T>,
  requiredKeys
>;

type FactoryProps<
  T extends object,
  requiredKeys extends keyof T
> = Partial<T> & {
  [k in requiredKeys]: T[k];
};

export const BetterRecord = <
  Props extends object,
  requiredKeys extends keyof Props = never
>(
  defaults: DefaultProps<Props, requiredKeys>
): Partial<Props> extends FactoryProps<Props, requiredKeys>
  ? (props?: Partial<Props>) => RecordOf<Props>
  : (props: FactoryProps<Props, requiredKeys>) => RecordOf<Props> =>
  iRecord<Props>(defaults as Props);
```

A few comments, `SetUndefined` is used for the default properties: any required key must be set to `undefined`. [Normally](https://github.com/immutable-js/immutable-js/blob/c41e7fb428c76fe9d22d0e1640af2dea71ae4051/type-definitions/immutable.d.ts#L2620) the factory just uses `Partial` to allow you to pick which properties you want to set in the factory. In `FactoryProps` I intersect the partial with an object that has all the properties of the `requiredKeys`, that forces those to _not_ be optional anymore. 

The ternary in the return type of `BetterRecord`
```ts
Partial<Props> extends FactoryProps<Props, requiredKeys>
  ? (props?: Partial<Props>) => RecordOf<Props>
  : (props: FactoryProps<Props, requiredKeys>) => RecordOf<Props>
```
is just so you don't have to pass the empty object when there are no `requiredKeys`. Note that `requiredKeys` defaults to `never`, which is the empty union.

