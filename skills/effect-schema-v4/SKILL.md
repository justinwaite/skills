---
name: effect-schema-v4
description:
  'Use when building Effect Schemas. Contains important information about API
  changes to the v4 Schemas.'
model: haiku
---

# Effect Schema V4

This project uses Effect v4 beta. The Schema API changed significantly from v3.
**Before writing any Schema code, check this file.** Many v3 APIs are renamed,
restructured, or removed.

## Step 1: Check if you're using a v3 API

Before writing schema code, scan the quick-reference table below. If what you
want to write appears in the "v3" column, use the v4 equivalent instead.

## Step 3: When in doubt, read the full migration guide

Full details with examples are in `./references/v3-v4-migration.md`.

---

## Quick-reference: Most common v3 → v4 changes

| What you want to do | ❌ v3 (broken)                            | ✅ v4 (correct)                                                           |
| ------------------- | ----------------------------------------- | ------------------------------------------------------------------------- |
| Union type          | `Schema.Union(A, B)`                      | `Schema.Union([A, B])`                                                    |
| Tuple type          | `Schema.Tuple(A, B)`                      | `Schema.Tuple([A, B])`                                                    |
| Template literal    | `Schema.TemplateLiteral(A, B)`            | `Schema.TemplateLiteral([A, B])`                                          |
| Add annotation      | `.annotations({ title: "..." })`          | `.annotate({ title: "..." })`                                             |
| Custom filter       | `Schema.filter(predicate)`                | `Schema.check(Schema.makeFilter(predicate))`                              |
| Type refinement     | `Schema.filter(refinement)`               | `Schema.refine(refinement)`                                               |
| Compose/transform   | `Schema.compose(schemaB)`                 | `.pipe(Schema.decodeTo(schemaB, ...))`                                    |
| Transform schema    | `Schema.transform(from, to, opts)`        | `from.pipe(Schema.decodeTo(to, transformation))`                          |
| Record schema       | `Schema.Record({ key, value })`           | `Schema.Record(key, value)`                                               |
| Decode              | `Schema.decodeUnknown(s)(x)`              | `Schema.decodeUnknownEffect(s)(x)`                                        |
| Decode sync         | `Schema.decodeUnknownSync(s)(x)`          | unchanged — still `Schema.decodeUnknownSync`                              |
| Decodeing fallback  | `.annotations({ decodingFallback: ... })` | `.pipe(Schema.catchDecoding(...))`                                        |
| Pick fields         | `.pipe(Schema.pick('a'))`                 | `.mapFields(Struct.pick(['a']))`                                          |
| Omit fields         | `.pipe(Schema.omit('b'))`                 | `.mapFields(Struct.omit(['b']))`                                          |
| Partial struct      | `.pipe(Schema.partial)`                   | `.mapFields(Struct.map(Schema.optional))`                                 |
| Extend struct       | `.pipe(Schema.extend(...))`               | `.mapFields(Struct.assign({...}))` or `.pipe(Schema.fieldsAssign({...}))` |
| Literals            | `Schema.Literal('a', 'b')`                | `Schema.Literals(['a', 'b'])`                                             |

---

## Critical patterns in detail

### Union — always use array form

```ts
// ❌ v3 — TypeScript error in v4
Schema.Union(Schema.String, Schema.Number);

// ✅ v4
Schema.Union([Schema.String, Schema.Number]);
```

### Annotations — method renamed

```ts
// ❌ v3
Schema.String.annotations({ title: 'Name' });

// ✅ v4
Schema.String.annotate({ title: 'Name' });
```

### Custom validation messages

Use `makeFilter` returning a string for a custom message:

```ts
// ❌ v3
Schema.String.pipe(Schema.filter((s) => s.length > 0 || 'Required'));

// ✅ v4
Schema.String.check(Schema.makeFilter((s) => s.length > 0 || 'Required'));
```

`makeFilter` predicates can return:

- `undefined` / `true` — success
- `false` — generic failure
- `string` — failure with that message
- `{ path, issue }` — failure at a nested path
- `SchemaIssue.Issue` — a fully-formed issue
- `ReadonlyArray<Schema.FilterIssue>` — multiple failures

### Struct field validation at a nested path

```ts
Schema.Struct({ password: Schema.String, confirm: Schema.String }).check(
  Schema.makeFilter((o) =>
    o.password === o.confirm
      ? undefined
      : { path: ['confirm'], issue: 'Passwords must match' },
  ),
);
```

### transform / transformOrFail

```ts
// ❌ v3
Schema.transform(From, To, { decode: ..., encode: ... })

// ✅ v4 — pipe decodeTo with SchemaTransformation or SchemaGetter
import { Schema, SchemaTransformation, SchemaGetter } from 'effect'

const schema = From.pipe(
  Schema.decodeTo(To, SchemaTransformation.transform({ decode, encode }))
)

// For fallible transforms:
const schema = From.pipe(
  Schema.decodeTo(To, {
    decode: SchemaGetter.transformOrFail((input) =>
      Effect.fail(new SchemaIssue.InvalidValue(Option.some(input)))
    ),
    encode: SchemaGetter.passthrough(),
  })
)
```

---

## Helpers

Common utility schemas for things like number and date parsing exist in
app/utils/effect/schema-utils.ts. Check if there is an existing utility for the
type of field you are trying to validate.

---

## Gotchas

- `Schema.Union` **always takes an array** — passing members as variadic args
  compiles with a misleading error about `ReadonlyArray<Top>`.
- `.annotate()` not `.annotations()` — `.annotations()` does not exist in v4.
- `Schema.filter` is **gone** — use `.check(Schema.makeFilter(...))` or
  `.pipe(Schema.refine(...))`.
- Built-in filters are now prefixed with `is`: `greaterThan` → `isGreaterThan`,
  `minLength` → `isMinLength`, etc.
- `Schema.positive`, `Schema.negative`, `Schema.nonNegative`, and
  `Schema.nonPositive` are removed — use `isGreaterThan(0)` etc.
- `Schema.validate*` APIs are removed — use `Schema.decode*` + `Schema.toType`.
