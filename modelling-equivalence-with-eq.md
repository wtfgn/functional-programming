# Modelling equivalence with Eq

Yet again, we can model the notion of equality.

_Equivalence relations_ capture the concept of _equality_ of elements of the same type. The concept of an _equivalence relation_ can be implemented in TypeScript with the following interface:

```ts
interface Eq<A> {
  readonly equals: (first: A, second: A) => boolean
}
```

Intuitively:

* if `equals(x, y) = true` then we say `x` and `y` are equal
* if `equals(x, y) = false` then we say `x` and `y` are different

**Example**

This is an instance of `Eq` for the `number` type:

```ts
import { Eq } from 'fp-ts/Eq'
import { pipe } from 'fp-ts/function'

const EqNumber: Eq<number> = {
  equals: (first, second) => first === second
}

pipe(EqNumber.equals(1, 1), console.log) // => true
pipe(EqNumber.equals(1, 2), console.log) // => false
```

The following laws have to hold true:

1. **Reflexivity**: `equals(x, x) === true`, for every `x` in `A`
2. **Symmetry**: `equals(x, y) === equals(y, x)`, for every `x`, `y` in `A`
3. **Transitivity**: if `equals(x, y) === true` and `equals(y, z) === true`, then `equals(x, z) === true`, for every `x`, `y`, `z` in `A`

**Quiz**. Would a combinator `reverse: <A>(E: Eq<A>) => Eq<A>` make sense?

**Quiz**. Would a combinator `not: <A>(E: Eq<A>) => Eq<A>` make sense?

```ts
import { Eq } from 'fp-ts/Eq'

export const not = <A>(E: Eq<A>): Eq<A> => ({
  equals: (first, second) => !E.equals(first, second)
})
```

**Example**

Let's see the first example of the usage of the `Eq` abstraction by defining a function `elem` that checks whether a given value is an element of `ReadonlyArray`.

```ts
import { Eq } from 'fp-ts/Eq'
import { pipe } from 'fp-ts/function'
import * as N from 'fp-ts/number'

// returns `true` if the element `a` is included in the list `as`
const elem = <A>(E: Eq<A>) => (a: A) => (as: ReadonlyArray<A>): boolean =>
  as.some((e) => E.equals(a, e))

pipe([1, 2, 3], elem(N.Eq)(2), console.log) // => true
pipe([1, 2, 3], elem(N.Eq)(4), console.log) // => false
```

Why would we not use the native `includes` Array method?

```ts
console.log([1, 2, 3].includes(2)) // => true
console.log([1, 2, 3].includes(4)) // => false
```

Let's define some `Eq` instance for more complex types.

```ts
import { Eq } from 'fp-ts/Eq'

type Point = {
  readonly x: number
  readonly y: number
}

const EqPoint: Eq<Point> = {
  equals: (first, second) => first.x === second.x && first.y === second.y
}

console.log(EqPoint.equals({ x: 1, y: 2 }, { x: 1, y: 2 })) // => true
console.log(EqPoint.equals({ x: 1, y: 2 }, { x: 1, y: -2 })) // => false
```

and check the results of `elem` and `includes`

```ts
const points: ReadonlyArray<Point> = [
  { x: 0, y: 0 },
  { x: 1, y: 1 },
  { x: 2, y: 2 }
]

const search: Point = { x: 1, y: 1 }

console.log(points.includes(search)) // => false :(
console.log(pipe(points, elem(EqPoint)(search))) // => true :)
```

**Quiz** (JavaScript). Why does the `includes` method returns `false`?

\-> See the [answer here](quiz-answers/javascript-includes.md)

Abstracting the concept of equality is of paramount importance, especially in a language like JavaScript where some data types do not offer handy APIs for checking user-defined equality.

The JavaScript native `Set` datatype suffers by the same issue:

```ts
type Point = {
  readonly x: number
  readonly y: number
}

const points: Set<Point> = new Set([{ x: 0, y: 0 }])

points.add({ x: 0, y: 0 })

console.log(points)
// => Set { { x: 0, y: 0 }, { x: 0, y: 0 } }
```

Given the fact that `Set` uses `===` ("strict equality") for comparing values, `points` now contains **two identical copies** of `{ x: 0, y: 0 }`, a result we definitely did not want. Thus it is convenient to define a new API to add an element to a `Set`, one that leverages the `Eq` abstraction.

**Quiz**. What would be the signature of this API?

Does `EqPoint` require too much boilerplate? The good news is that theory offers us yet again the possibility of implementing an `Eq` instance for a struct like `Point` if we are able to define an `Eq` instance for each of its fields.

Conveniently the `fp-ts/Eq` module exports a `struct` combinator:

```ts
import { Eq, struct } from 'fp-ts/Eq'
import * as N from 'fp-ts/number'

type Point = {
  readonly x: number
  readonly y: number
}

const EqPoint: Eq<Point> = struct({
  x: N.Eq,
  y: N.Eq
})
```

**Note**. Like for Semigroup, we aren't limited to `struct`-like data types, we also have combinators for working with tuples: `tuple`

```ts
import { Eq, tuple } from 'fp-ts/Eq'
import * as N from 'fp-ts/number'

type Point = readonly [number, number]

const EqPoint: Eq<Point> = tuple(N.Eq, N.Eq)

console.log(EqPoint.equals([1, 2], [1, 2])) // => true
console.log(EqPoint.equals([1, 2], [1, -2])) // => false
```

There are other combinators exported by `fp-ts`, here we can see a combinator that allows us to derive an `Eq` instance for `ReadonlyArray`s.

```ts
import { Eq, tuple } from 'fp-ts/Eq'
import * as N from 'fp-ts/number'
import * as RA from 'fp-ts/ReadonlyArray'

type Point = readonly [number, number]

const EqPoint: Eq<Point> = tuple(N.Eq, N.Eq)

const EqPoints: Eq<ReadonlyArray<Point>> = RA.getEq(EqPoint)
```

Similarly to Semigroups, it is possible to define more than one `Eq` instance for the same given type. Suppose we have modeled a `User` with the following type:

```ts
type User = {
  readonly id: number
  readonly name: string
}
```

we can define a "standard" `Eq<User>` instance using the `struct` combinator:

```ts
import { Eq, struct } from 'fp-ts/Eq'
import * as N from 'fp-ts/number'
import * as S from 'fp-ts/string'

type User = {
  readonly id: number
  readonly name: string
}

const EqStandard: Eq<User> = struct({
  id: N.Eq,
  name: S.Eq
})
```

Several languages, even pure functional languages like Haskell, do not allow to have more than one `Eq` instance per data type. But we may have different contexts where the meaning of `User` equality might differ. One common context is where two `User`s are equal if their `id` field is equal.

```ts
/** two users are equal if their `id` fields are equal */
const EqID: Eq<User> = {
  equals: (first, second) => N.Eq.equals(first.id, second.id)
}
```

Now that we made an abstract concept concrete by representing it as a data structure, we can programmatically manipulate `Eq` instances like we do with other data structures. Let's see an example.

**Example**. Rather than manually defining `EqId` we can use the combinator `contramap`: given an instance `Eq<A>` and a function from `B` to `A`, we can derive an `Eq<B>`

```ts
import { Eq, struct, contramap } from 'fp-ts/Eq'
import { pipe } from 'fp-ts/function'
import * as N from 'fp-ts/number'
import * as S from 'fp-ts/string'

type User = {
  readonly id: number
  readonly name: string
}

const EqStandard: Eq<User> = struct({
  id: N.Eq,
  name: S.Eq
})

const EqID: Eq<User> = pipe(
  N.Eq,
  contramap((user: User) => user.id)
)

console.log(
  EqStandard.equals({ id: 1, name: 'Giulio' }, { id: 1, name: 'Giulio Canti' })
) // => false (because the `name` property differs)

console.log(
  EqID.equals({ id: 1, name: 'Giulio' }, { id: 1, name: 'Giulio Canti' })
) // => true (even though the `name` property differs)

console.log(EqID.equals({ id: 1, name: 'Giulio' }, { id: 2, name: 'Giulio' }))
// => false (even though the `name` property is equal)
```

**Quiz**. Given a data type `A`, is it possible to define a `Semigroup<Eq<A>>`? What could it represent?
