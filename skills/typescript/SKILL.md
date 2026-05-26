---
name: typescript
description: >
  Rodrigo's personal TypeScript-specific style preferences — TS-only
  rules layered on top of the `javascript` skill (which this skill
  requires; load it first so its rules apply before these). Use this
  skill whenever editing, creating, reviewing, or generating any `.ts` /
  `.tsx` file — even for small tweaks. Two themes: (1) don't lie to the
  compiler — avoid `as` / `as unknown as`, `any`, non-null `!`, and
  `@ts-ignore`; prefer type guards, `unknown` + narrowing, `satisfies`,
  and `as const`. (2) Let the type system do the work — generic
  inference instead of widening to a base type, pass generics at the
  call site instead of casting the return, method overloads when the
  return shape varies, utility types instead of duplicated shapes,
  object enums (`as const` + derived type) instead of TS `enum`,
  discriminated unions with `never` exhaustiveness checks. Trigger on
  phrases like "fix this type error", "type this function", "make this
  generic", "narrow this type", "this casts to", "add a type guard",
  "use satisfies", "avoid enum", "type a generator", "Iterable<T>",
  "AsyncIterable<T>", "Generator<T> type", "AsyncGenerator<T> type" —
  for sequence inputs prefer `Iterable<T>` / `AsyncIterable<T>` over
  `Generator<T>` / `AsyncGenerator<T>` so arrays and other iterables
  work too; return types can stay as `Generator<T>` / `AsyncGenerator<T>`
  with the trailing type params defaulted. As well as any unprompted
  TypeScript work.
---

# TypeScript style

**Prerequisite: load the `javascript` skill first.** Every rule in `javascript` (early returns over `else`, `const` over `let`, no chained ternaries, brace all control-flow bodies, no `.forEach`) applies to TypeScript too. This skill only adds the TS-specific rules layered on top — types, generics, enums, narrowing.

Personal TS-specific style rules to follow whenever touching TypeScript. Two themes underlie everything:

1. **Don't lie to the compiler.** Casts, `any`, `!`, and `@ts-ignore` all silently disable type checking. If you're wrong, nothing tells you — until production. Prefer constructs that prove the claim (type guards, `satisfies`, narrowing).
2. **Let the type system do the work.** Generics, overloads, utility types, and discriminated unions let TypeScript infer the exact shape at every call site. Stop hand-typing what the compiler can derive.

Both themes catch real bugs that show up when code is refactored or reused — exactly the situations where untyped escape hatches rot fastest.

---

## Part 1 — Don't lie to the compiler

### 1.1 No `as` casting — use type guards

`as X` and especially `as unknown as X` silently tell the compiler "trust me." A type guard does the same narrowing work _and_ checks reality at runtime, so the call site is honest about what it actually knows.

Only reach for `as` when there's no alternative: a third-party API whose types are wrong, asserting a discriminant the compiler can't follow, or the very narrow cases where `satisfies` and `as const` (see below) don't apply. If you can write a guard, write a guard.

**Wrong:**

```ts
interface Example {
  property: string;
}

function test(arg: unknown) {
  const casted = arg as Example;
}
```

**Right:**

```ts
interface Example {
  property: string;
}

const isExample = (arg: unknown): arg is Example =>
  !!arg && typeof arg === "object" && "property" in arg;

function test(arg: unknown) {
  if (!isExample(arg)) {
    // handle invalid type here — throw or return
    return;
  }
  // arg is now Example
}
```

### 1.2 `unknown`, not `any`

`any` is contagious — it disables type checking on everything it touches and propagates silently through the codebase. `unknown` is the safe version: it accepts anything, but forces you to narrow before use. If you find yourself writing `any` to "get past a type error," that's the type system telling you the shape isn't proven yet; reach for `unknown` + a guard.

**Wrong:**

```ts
function parse(input: any) {
  return input.payload.id; // any, no checks, no help
}
```

**Right:**

```ts
function parse(input: unknown) {
  if (!isPayload(input)) {
    throw new Error("bad input");
  }
  return input.payload.id; // typed
}
```

Same philosophy applies to `Record<string, any>`, `{ [key: string]: any }`, `Function`, `object` — they're all "trust me" shapes. Use `Record<string, unknown>` and narrow.

### 1.3 No non-null `!` — narrow instead

The non-null assertion `value!` is `as NonNullable<typeof value>` with prettier syntax. Same lie, same risk. If the value might be `null`/`undefined`, write the check; if it can't, fix the type so the compiler knows.

**Wrong:**

```ts
const user = users.find((u) => u.id === id)!;
console.log(user.name); // bombs if find returned undefined
```

**Right:**

```ts
const user = users.find((u) => u.id === id);
if (!user) {
  throw new Error(`user ${id} not found`);
}
console.log(user.name);
```

### 1.4 `satisfies` for literals, not `as`

When you want to constrain a literal to a shape but keep its narrow inferred type (so individual keys/values stay precise), `satisfies` is what you want. `as` widens or lies; `satisfies` checks and preserves.

**Wrong:**

```ts
const routes = {
  home: "/",
  about: "/about",
} as Record<string, string>;
// routes.home is now `string` — narrow info lost
```

**Right:**

```ts
const routes = {
  home: "/",
  about: "/about",
} satisfies Record<string, string>;
// routes.home is `'/'`, and unknown keys are caught at the literal
```

### 1.5 `as const` is the legitimate exception

`as const` is not a cast — it's an instruction to TypeScript to keep literal types literal and arrays/objects readonly. Use it freely for fixture data, action-type strings, tuple returns, etc. It's the one `as` that doesn't lie.

```ts
const STATUSES = ["idle", "loading", "error"] as const;
type Status = (typeof STATUSES)[number]; // 'idle' | 'loading' | 'error'
```

### 1.6 `@ts-ignore` / `@ts-expect-error` — last resort, documented

If you absolutely must silence the compiler, prefer `@ts-expect-error` (which itself errors if the line stops needing it) over `@ts-ignore` (which silently rots), and always leave a one-line comment explaining _why_. A bare `@ts-ignore` is a future bug with no owner.

```ts
// @ts-expect-error — upstream types missing for <feature>, tracking in <link>
externalLib.undocumentedMethod();
```

---

## Part 2 — Let the type system do the work

### 2.1 Generics for inference — don't widen to a base type

When a function accepts a base type, typing the parameter as the base type erases the caller's specific type. Use a generic constrained to the base — TS will infer the concrete type at the call site and the callback gets the real shape.

**Wrong:**

```ts
interface BaseObject {
  id: string;
}
interface ObjectA extends BaseObject {
  name: string;
}

const tapObject = (
  obj: BaseObject,
  tap: (obj: BaseObject) => void,
): BaseObject => {
  tap(obj);
  return obj;
};

const objectA: ObjectA = { id: "a", name: "Name" };

tapObject(objectA, (obj) => {
  console.log(obj.name); // ❌ Property 'name' does not exist on BaseObject
});
```

**Right:**

```ts
const tapObject = <T extends BaseObject>(obj: T, tap: (obj: T) => void): T => {
  tap(obj);
  return obj;
};

tapObject(objectA, (obj) => {
  console.log(obj.name); // ✅ inferred as ObjectA
});
```

### 2.2 Pass the generic at the call site — don't cast the return

If a function is generic over its return type, supply the type argument when you call it. Casting the result throws away the type-system relationship between the call and the variable.

**Wrong:**

```ts
async function fetch<T = unknown>(url: string): Promise<{ data: T }> {
  // implementation
}

interface Response {
  id: string;
}

const response = await fetch("url");
const data = response.data as Response; // ❌ cast
```

**Right:**

```ts
const response = await fetch<Response>("url");
const data = response.data; // ✅ inferred as Response
```

### 2.3 Method overloads when return type depends on arguments

When a function's return shape changes based on what's passed (e.g. a default makes the return non-nullable), express that with overload signatures instead of one wide signature. Callers get the precise type for their actual call.

```ts
async function getFromCache<T = unknown>(key: string): Promise<T | null> {
  // implementation
}

export async function get<T = unknown>(key: string): Promise<T | null>;
export async function get<T = unknown>(key: string, fallback: T): Promise<T>;
export async function get<T = unknown>(
  key: string,
  fallback?: T,
): Promise<T | null> {
  const value = await getFromCache<T>(key);
  return value ?? fallback ?? null;
}

// Usage:
const a = await get<string>("k"); // string | null
const b = await get<string>("k", "default"); // string
```

### 2.4 Utility types — don't redeclare shapes

When a type is a slice or transform of another, derive it. Hand-rewriting the shape duplicates the source of truth, and the copy will drift when the original changes. `Pick`, `Omit`, `Partial`, `Required`, `Readonly`, `ReturnType`, `Parameters`, `Awaited`, and `typeof` cover most cases.

**Wrong:**

```ts
interface User {
  id: string;
  name: string;
  email: string;
  passwordHash: string;
}

interface PublicUser {
  id: string;
  name: string;
  email: string;
}
```

**Right:**

```ts
type PublicUser = Omit<User, "passwordHash">;

// And derive from values when shapes already exist:
const defaultConfig = { retries: 3, timeout: 5000 };
type Config = typeof defaultConfig;

// And derive from functions:
type FetchResult = Awaited<ReturnType<typeof fetchUser>>;
```

### 2.5 Object enums, not TS `enum`

Avoid TypeScript's `enum`. It emits runtime code that doesn't always play well with `isolatedModules` / bundlers, mixes numeric and string semantics, and the type isn't a plain union. Prefer an object-as-const plus a derived type — same ergonomics, no runtime weirdness, and the type _is_ the literal union for free.

```ts
export const TranslationStatus = {
  Match: "match",
  NoMatch: "no_match",
  Reviewed: "reviewed",
  Excluded: "excluded",
} as const;

export type TranslationStatus =
  (typeof TranslationStatus)[keyof typeof TranslationStatus];
```

This pattern is a soft preference over a bare union (`type TranslationStatus = 'match' | 'no_match' | 'reviewed' | 'excluded'`). Bare unions are fine when you only need the type; the object form is better when you also want a runtime value to iterate, reference, or use as a namespace. Don't be dogmatic — pick the one that fits the call sites.

**Reference enum members, not the underlying string literal**, at comparison sites. If the literal ever changes (`'match'` → `'matched'`), updating the const propagates; updating literals scattered across the codebase doesn't.

**Wrong:**

```ts
const isMatch = (status: TranslationStatus) => status === "match";
```

**Right:**

```ts
const isMatch = (status: TranslationStatus) =>
  status === TranslationStatus.Match;
```

### 2.6 Discriminated unions + `never` exhaustiveness

For "one of these shapes" data, use a discriminated union with a literal tag — TypeScript narrows automatically on the tag, no guards needed. Pair with a `never`-typed default branch so adding a new variant becomes a compile error everywhere it's handled.

**Wrong:**

```ts
interface State {
  status: string;
  data?: User;
  error?: Error;
}
```

**Right:**

```ts
type State =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: User }
  | { status: 'error'; error: Error }

function render(state: State) {
  switch (state.status) {
    case 'idle': return null
    case 'loading': return <Spinner />
    case 'success': return <User user={state.data} />
    case 'error': return <Err error={state.error} />
    default: {
      const _exhaustive: never = state
      throw new Error(`unhandled: ${_exhaustive}`)
    }
  }
}
```

Now if someone adds `{ status: 'cancelled' }` to `State`, the compiler points at every `switch` that forgot to handle it.

### 2.7 Type iterable inputs as `Iterable<T>` / `AsyncIterable<T>`

When a function consumes a sequence, type the parameter as `Iterable<T>` or `AsyncIterable<T>` rather than `Generator<T>` or `AsyncGenerator<T>`. The iterable types accept arrays, sets, maps, *and* generators — the generator types reject everything except a generator, which makes the function pointlessly picky for no real safety gain.

For *return* types, `Generator<T>` / `AsyncGenerator<T>` is fine. The two trailing type parameters (`TReturn = void`, `TNext = unknown`) default sensibly for the yield-only generators that section 8 of the `javascript` skill describes, so writing `Generator<T, void, unknown>` in full is noise — leave them off unless you actually use the `return` value or `next(value)` argument.

**Wrong:**

```ts
function summarize(rows: Generator<Row>): Summary {
  // rejects callers who already have a Row[]
}
```

**Right:**

```ts
function summarize(rows: Iterable<Row>): Summary {
  // accepts arrays, sets, generators — anything iterable
}

function* readRows(path: string): Generator<Row> {
  // return type stays as Generator — TReturn / TNext defaults are fine
}
```

Same shape on the async side: `AsyncIterable<T>` in, `AsyncGenerator<T>` out. This is the type-system half of the JS skill's section 8 — the runtime case for generators is there; this is just "type the boundary so callers aren't forced into one shape."

---

## Summary checklist

Before finishing TS work, scan the diff for:

- [ ] Any `as X` or `as unknown as X` — can it become a type guard, a generic call-site argument, `satisfies`, or `as const`? If you keep it, it should be at a real boundary (JSON.parse output, third-party types) with a one-line comment.
- [ ] Any `any` — should it be `unknown` + a guard?
- [ ] Any non-null `!` — should it be an explicit `if (!value)` check?
- [ ] Any object literal typed with `: SomeType` — would `satisfies SomeType` keep more info?
- [ ] Any `@ts-ignore` — convert to `@ts-expect-error` with an explaining comment.
- [ ] Any function whose parameter is a base type but whose callback / return uses that param — should it be `<T extends Base>` instead?
- [ ] Any function whose return type varies with arguments — does it deserve overloads?
- [ ] Any `response.data as X` / `result as X` after a generic call — pass `<X>` at the call site instead.
- [ ] Any type that copy-pastes another type's fields — can it be `Pick`/`Omit`/`Partial`/`ReturnType`/`typeof X`?
- [ ] Any TS `enum` — can it become an `as const` object + `(typeof X)[keyof typeof X]` derived type?
- [ ] Any string-literal comparison against an enum-typed value (`status === 'match'`) — reference the enum member instead (`status === TranslationStatus.Match`).
- [ ] Any `switch` on a discriminant — does it have a `never` default to catch missed variants?
- [ ] Any function parameter typed `Generator<T>` / `AsyncGenerator<T>` — should it be `Iterable<T>` / `AsyncIterable<T>` so arrays and other iterables also work? Return types can stay as `Generator<T>` / `AsyncGenerator<T>` — the trailing type params default sensibly for yield-only generators.

Plus everything in the `javascript` skill's checklist — assume it ran first (early returns / `else`, `const` over `let`, chained ternaries, brace bodies, `.forEach`, generators for lazy iteration).
