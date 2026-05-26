---
name: javascript
description: >
  Rodrigo's personal JavaScript style preferences. Use this skill
  whenever editing, creating, reviewing, or generating any `.js` /
  `.jsx` file — even for small tweaks. The `typescript` skill also
  requires loading this one, so the same rules apply to `.ts` / `.tsx`
  (TS loads JS first; both rule sets apply to TS work). Rules: prefer
  early returns over `else` and flip the condition to flatten the happy
  path; prefer `const` over `let` (keep `let` only for genuine mutation
  like `do/while` body-computed values, loop accumulators, or
  imperative counters); avoid chained ternaries — a single `cond ? a :
  b` is fine but `a ? b : c ? d : e` belongs in a helper with early
  returns or a `switch`; always brace control-flow bodies — `if`,
  `for`, `while`, `do`, `else` get `{}` with the body on its own line;
  never use `.forEach`, always `for (const x of arr) { ... }`. Trigger
  on phrases like "remove the else", "flatten this", "early return",
  "guard clause", "prefer const", "avoid let", "use const", "const over
  let", "immutable binding", "chained ternary", "nested ternary",
  "ternary chain", "avoid ternary", "brace this", "add braces", "no
  inline if", "single-line if", "no forEach", "avoid forEach", "replace
  forEach", "use for of", "convert forEach"; and consider `function*` /
  `async function*` for lazy iteration when the sequence is large or
  unbounded, the caller may stop early, or you're walking a tree
  recursively — trigger on "use a generator", "async generator",
  "yield*", "lazy iteration", "streaming iterator", "for await of",
  "paginated iterator". As well as any unprompted JavaScript work.
---

# JavaScript style

Personal JS style rules to follow whenever touching JavaScript. All of these also apply to TypeScript — the `typescript` skill loads this one first.

The unifying theme: keep code flat and readable so the next reader (or you next month) can follow the happy path top-to-bottom without parsing branches in their head. These rules catch bugs that show up under refactor or reuse — the situations where dense, branchy code rots fastest.

---

## 1. Prefer early returns over `else`

`else` deepens indentation, multiplies branches, and pushes the happy path further from the function entry. In ~95% of cases an `if/else` rewrites cleanly as a guard with an early return — the function then reads top-to-bottom, edge cases handled and out of the way at the top, main logic flat below.

When the non-happy branch is more than a couple lines, extract it into a named function. The guard collapses to a one-liner, and naming the branch usually clarifies intent.

**Wrong:**

```js
function processOrder(order) {
  if (order.isValid) {
    const total = calculateTotal(order);
    applyDiscount(order, total);
    notifyWarehouse(order);
    return total;
  } else {
    logError(`invalid order ${order.id}`);
    notifyCustomer(order, "rejected");
    return 0;
  }
}
```

**Right:**

```js
function processOrder(order) {
  if (!order.isValid) {
    return rejectOrder(order);
  }

  const total = calculateTotal(order);
  applyDiscount(order, total);
  notifyWarehouse(order);
  return total;
}

function rejectOrder(order) {
  logError(`invalid order ${order.id}`);
  notifyCustomer(order, "rejected");
  return 0;
}
```

## 2. When `else` is acceptable

The exception is when `else` is just picking between two short calls **and** the function still has work to do after the branch. Flipping to an early return forces extracting the trailing code into a new function — churn for no real readability gain.

```js
const isNewObject = (obj) => {
  /* ... */
};
const create = (obj) => {
  /* ... */
};
const update = (obj) => {
  /* ... */
};

const save = (obj) => {
  if (isNewObject(obj)) {
    create(obj);
  } else {
    update(obj);
  }

  // post-processing that must run regardless
};
```

Both branches are one line, mutually exclusive, and the trailing code must run either way. Keep the `else`.

(When both branches produce a value rather than a side effect, a ternary works too: `const result = isNewObject(obj) ? create(obj) : update(obj)`.)

## 3. Prefer `const` over `let`

`const` locks the binding so accidental reassignment becomes an error, and signals to a reader "this value doesn't change beyond this point" — cognitive load drops. The most common `let` smell is "declare empty → assign in each branch → use below"; that's a `const` waiting to happen, via a ternary or an extracted helper.

**Wrong:**

```js
let label;
if (user.isAdmin) {
  label = "Admin";
} else if (user.isManager) {
  label = "Manager";
} else {
  label = "User";
}
render(label);
```

**Right:**

```js
const label = getLabel(user);
render(label);

function getLabel(user) {
  if (user.isAdmin) {
    return "Admin";
  }
  if (user.isManager) {
    return "Manager";
  }
  return "User";
}
```

For a two-value pick a single ternary works too: `const label = user.isAdmin ? "Admin" : "User"`. Don't chain ternaries — see section 5.

Note that this rule and section 1 reinforce each other: extracting a helper to flatten an `if/else` usually also lets you drop a `let` at the call site.

## 4. When `let` is acceptable

`let` is fine when the mutation is the _point_ of the construct, not a workaround for branching. Concrete cases:

- **`do { } while ()` body-computed values.** The `while` condition tests a value that the body produces, so the binding has to live outside the body:

  ```js
  let value;
  do {
    value = await poll();
  } while (!isReady(value));
  ```

- **Loop accumulators where `reduce` would hurt readability.** Multi-key tallying, side-effectful aggregation, or iteration over something that isn't a clean array (Map entries, generators, paginated APIs):

  ```js
  let total = 0;
  for (const row of rows) {
    total += row.amount;
  }
  ```

  (Use `reduce` when it's clearly tidier; don't force it when it isn't.)

- **Counters in imperative loops where `for...of` doesn't fit** — driving an external iterator, polling with backoff, cursor-based pagination, etc. (`for (let i = 0; ...)` doesn't count — that `let` is required by syntax, not a style choice.)

The rule of thumb: ask whether the variable is genuinely mutated over time, or just assigned once. If it's the latter, it's a `const` in disguise.

## 5. Avoid chained ternaries

A single `cond ? a : b` is one decision and reads cleanly. Chaining them (`a ? b : c ? d : e`) stacks multiple decisions into one expression and forces the reader to parse right-associative precedence in their head. Three-or-more-way picks belong somewhere with named branches: an extracted helper with early returns, or a `switch` on a tag.

**Wrong:**

```js
const label = user.isAdmin ? "Admin" : user.isManager ? "Manager" : "User";
```

**Right (helper with early returns):**

```js
const label = getLabel(user);

function getLabel(user) {
  if (user.isAdmin) {
    return "Admin";
  }
  if (user.isManager) {
    return "Manager";
  }
  return "User";
}
```

**Right (`switch` on a tag field)** — when dispatching on a discriminant string rather than a boolean cascade:

```js
function label(state) {
  switch (state.kind) {
    case "loading":
      return "Loading…";
    case "error":
      return `Error: ${state.message}`;
    case "ok":
      return state.data.title;
  }
}
```

A single ternary is fine — it's the chaining that hurts.

## 6. Always brace control-flow bodies

`if`, `else`, `else if`, `for`, `for…of`, `for…in`, `while`, and `do` always get `{}` with the body on its own line(s) — no inline `if (cond) return x;` or `for (...) arr.push(...)`. A brace-less body shares a line with the condition, which blurs the visual scope of the branch, and the classic footgun is adding a second statement under the `if` later that silently runs unconditionally. Braces + line breaks make the body's extent explicit and let `prettier`/`eslint` format it consistently.

**Wrong:**

```js
if (!order.isValid) return reject(order);

for (const row of rows) total += row.amount;
```

**Right:**

```js
if (!order.isValid) {
  return reject(order);
}

for (const row of rows) {
  total += row.amount;
}
```

Scope: this rule is about *statement bodies* on control-flow keywords. It doesn't apply to arrow-function expression bodies (`(x) => x + 1`), single ternaries (see section 5), or conventional `switch` `case:` clauses (which aren't braced in the existing examples).

## 7. No `array.forEach` — use `for…of`

`.forEach` is callback ceremony around something that's just a loop — and it removes capabilities you usually want. `for…of` supports `break`, `continue`, and `return` from the enclosing function; `await` inside the body actually waits (an `await` inside a `forEach` callback is silently ignored by `forEach`, which fires-and-moves-on); stack traces don't pick up an extra anonymous frame; and the code reads as the imperative iteration it actually is. `.forEach` also returns `undefined`, so it's not even composable.

**Wrong:**

```js
users.forEach((user) => sendEmail(user));
```

**Right:**

```js
for (const user of users) {
  sendEmail(user);
}
```

If `.forEach` sits at the end of a chain like `xs.filter(...).forEach(...)`, pull the chain result into a `const` and `for…of` it:

```js
const active = xs.filter(isActive);
for (const x of active) {
  process(x);
}
```

Scope: only `.forEach` is forbidden. `.map` / `.filter` / `.reduce` / `.some` / `.every` / `.find` / `.flatMap` are all fine — they return values and compose as expressions, which is what makes them worth using over a raw loop.

## 8. Consider generators for lazy iteration

Generators (`function*`) and async generators (`async function*`) are an underused tool. Reach for them when one of three patterns shows up: a sequence that's too large or open-ended to materialize into an array, a caller that should be free to stop early, or a recursive walk that's currently piling items into an accumulator. Soft recommendation, not a hard rule — but worth a beat of thought whenever you see those shapes.

**Memory — yield items instead of building an array.** When the full sequence doesn't need to live in memory at once, yielding holds one item at a time. The classic case is a paginated source where the caller iterates without ever knowing pages exist:

```js
async function* allUsers() {
  let cursor = null;
  do {
    const page = await fetchPage(cursor);
    for (const user of page.items) {
      yield user;
    }
    cursor = page.nextCursor;
  } while (cursor);
}

for await (const user of allUsers()) {
  await process(user);
}
```

No materialized result set exists at any point — peak memory is one page.

**Performance — laziness lets the caller stop.** An eager `.map(...).filter(...).find(...)` chain walks the whole input. A generator + `for…of` runs only until the caller `break`s, so unwanted work is never done:

```js
function* matchingLines(lines, pattern) {
  for (const line of lines) {
    if (pattern.test(line)) {
      yield line;
    }
  }
}

for (const hit of matchingLines(lines, /ERROR/)) {
  console.log(hit);
  break; // first match only — nothing past this runs
}
```

**Recursion — `yield*` flattens accumulators.** Tree and graph walkers usually build up an array and return it; the generator form composes via `yield*` and the caller iterates. The helper shrinks and the `out.push(...walk(child))` spread disappears.

**Wrong:**

```js
function walk(node) {
  const out = [];
  out.push(node.value);
  for (const child of node.children) {
    out.push(...walk(child));
  }
  return out;
}
```

**Right:**

```js
function* walk(node) {
  yield node.value;
  for (const child of node.children) {
    yield* walk(child);
  }
}
```

Bonus: the recursive form composes with the laziness from the previous point — the caller can `break` partway through the traversal and the unvisited subtrees are never touched.

**Async generators carry the same wins.** `async function*` covers paginated fetches, streaming reads, websocket-style sources. Consumers use `for await…of`. One thing worth calling out: `await` inside the body works the way you'd expect — the next item isn't produced until the awaited work resolves. This is a deliberate contrast to section 7 — `await` inside a `.forEach` callback is silently ignored and the iteration races on. Async generators are the right shape for "iterate over an async source, awaiting per item."

**When not to reach for generators.** Small fixed arrays don't pay for the indirection. Code that needs `.length`, `arr[i]`, or any random access needs an array. Generators are also single-shot — iterating the same sequence twice means either calling the factory again or materializing with `[...gen]` / `Array.from(gen)`. If either keeps happening, the array form is the cleaner shape.

---

## Summary checklist

Before finishing JS work, scan the diff for:

- [ ] Any `else` block — can it flip to an early return (extracting the branch into a named function if it's long)? Keep `else` only when both branches are short calls and there's work after the branch.
- [ ] Any `let` — is the variable actually mutated, or just assigned once across branches? If the latter, rewrite as `const` with a ternary or an extracted helper. Keep `let` for genuine mutation: `do/while` body-computed values, loop accumulators, imperative counters.
- [ ] Any chained ternary (`a ? b : c ? d : e`) — extract into a helper with early returns, or `switch` on a tag. A single `cond ? a : b` is fine.
- [ ] Any inline single-line control-flow body (`if (cond) return;`, `for (...) arr.push(...)`) — always wrap in `{}` with the body on the next line. Applies to `if`/`else`/`for`/`while`/`do`; not to arrow expression bodies or `switch` cases.
- [ ] Any `arr.forEach(...)` — rewrite as `for (const x of arr) { ... }`. `.map`/`.filter`/`.reduce`/`.find` etc. are fine; only `.forEach` is forbidden.
- [ ] Any function that builds an accumulator array in a recursive walk, or a loop that materializes a full sequence the caller will iterate once — consider `function*` / `async function*` so items stream lazily and callers can `break` early. Soft recommendation; small fixed arrays don't need it.
