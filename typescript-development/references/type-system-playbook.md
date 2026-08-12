# Type-System Playbook

Use this reference for non-trivial TypeScript design. Apply the skill's priority order throughout: runtime truth, intuitive use, one source of truth, refactor safety, then measured tooling concerns. Use inference-first design across all of them: infer local values and implementation returns while preserving useful literals and relationships. Choose the least complex implementation that produces the best overall abstraction; sophisticated internal types are appropriate when they are localized and make consuming code meaningfully simpler. Keep examples minimal and framework-neutral: demonstrate one mechanism, then explain variations in prose instead of adding several near-duplicate examples.

## Contents

1. [Choose the source of truth](#choose-the-source-of-truth)
2. [Construct and validate objects](#construct-and-validate-objects)
3. [Infer function types deliberately](#infer-function-types-deliberately)
4. [Design inference-friendly generics](#design-inference-friendly-generics)
5. [Compose types without repetition](#compose-types-without-repetition)
6. [Preserve correlated data](#preserve-correlated-data)
7. [Model finite states](#model-finite-states)
8. [Narrow known unions with control flow](#narrow-known-unions-with-control-flow)
9. [Enforce exhaustive decisions](#enforce-exhaustive-decisions)
10. [Use never deliberately](#use-never-deliberately)
11. [Narrow boundary data safely](#narrow-boundary-data-safely)
12. [Repair type mismatches before asserting](#repair-type-mismatches-before-asserting)
13. [Control assertions and escape hatches](#control-assertions-and-escape-hatches)
14. [Represent mutability and variance](#represent-mutability-and-variance)
15. [Use mapped and conditional types intentionally](#use-mapped-and-conditional-types-intentionally)
16. [Use nominal distinctions only when valuable](#use-nominal-distinctions-only-when-valuable)
17. [Handle async code and errors](#handle-async-code-and-errors)
18. [Keep types readable and fast](#keep-types-readable-and-fast)
19. [Recognize common failure modes](#recognize-common-failure-modes)
20. [Primary references](#primary-references)

## Choose the source of truth

### Derive types from authoritative values

Use a runtime value as the source when the application must enumerate the same choices at runtime:

```ts
export const orderStates = ['draft', 'submitted', 'fulfilled'] as const;

export type OrderState = (typeof orderStates)[number];

export const stateLabels = {
  draft: 'Draft',
  submitted: 'Submitted',
  fulfilled: 'Fulfilled',
} as const satisfies Record<OrderState, string>;
```

This design makes additions fail until every required lookup is updated. Prefer it to a separately maintained string union plus array.

Use `keyof typeof value` for object keys, `typeof value[keyof typeof value]` for object values, and `(typeof values)[number]` for tuple/array elements.

### Derive types from authoritative functions

Use the standard helpers instead of reproducing signatures:

```ts
function loadAccount(id: string, options?: { includeHistory?: boolean }) {
  return Promise.resolve({
    id,
    active: true,
    history: options?.includeHistory ? ['created'] : undefined,
  });
}

type LoadAccountArgs = Parameters<typeof loadAccount>;
type LoadAccountResult = Awaited<ReturnType<typeof loadAccount>>;
```

Do not derive a public type from a private function if that couples consumers to accidental implementation detail. Promote a deliberate named contract instead.

### Define the contract first when it is independently authoritative

Prefer an interface first for a network protocol, database record, published package API, or cross-team contract that must remain stable even if one implementation changes:

```ts
export interface Clock {
  now(): Date;
}

export function createSystemClock(): Clock {
  return { now: () => new Date() };
}
```

The explicit return is justified because `Clock`, not the implementation object, is the promised public surface.

## Construct and validate objects

### Use `satisfies` to check without broadening

```ts
interface RouteDefinition {
  readonly method: 'GET' | 'POST';
  readonly path: `/${string}`;
  readonly roles: readonly string[];
}

const routes = {
  listUsers: { method: 'GET', path: '/users', roles: ['admin'] },
  createUser: { method: 'POST', path: '/users', roles: ['admin'] },
} as const satisfies Record<string, RouteDefinition>;

type RouteName = keyof typeof routes;
type RoutePath = (typeof routes)[RouteName]['path'];
```

Use a direct annotation when intentional widening or information hiding is the point:

```ts
const cache: Map<string, number> = new Map();
```

Use an assertion only when the runtime fact is proven but TypeScript cannot express the proof. `satisfies` is a check; `as` is a claim.

### Give inference actual evidence

Inference cannot invent the future element type of an empty mutable collection. `satisfies` checks the expression but does not replace its resulting type:

```ts
interface Job {
  id: string;
}

const jobs: Job[] = [];
```

The annotation is justified because mutation will supply values later. Do not write `const jobs = [] satisfies Job[]` and expect `jobs.push(job)` to work; the expression can remain `never[]`. The same principle applies to empty maps/sets and objects populated after construction. Prefer immediate construction with evidence when practical, otherwise annotate the mutable container.

### Choose `as const` intentionally

Use `as const` when all three are desirable:

- literal values must remain literal;
- properties/tuples should be readonly to consumers;
- the value is declarative data such as configuration, routes, commands, or lookup tables.

Omit it for state that must mutate or when literal detail creates noisy types. Design contracts with `readonly` arrays/properties when they are expected to accept `as const` values.

Remember that `as const` does not call `Object.freeze`; aliases and nested runtime objects can still mutate outside the static view.

### Preserve definition-driven inference

Treat schema, route, command, form, and validator definitions as small value-level type languages. Preserve the complete definition type, interpret it with mapped/conditional types, and carry the result into the returned API:

```ts
type FieldKind = 'text' | 'number';
type FieldValue<K extends FieldKind> = K extends 'text' ? string : number;
type DataOf<D extends Record<string, FieldKind>> = {
  -readonly [K in keyof D]: FieldValue<D[K]>;
};

interface Model<D extends Record<string, FieldKind>> {
  create(value: DataOf<D>): DataOf<D>;
}

declare function defineModel<const D extends Record<string, FieldKind>>(
  definition: D,
): Model<D>;

type InferModel<M> = M extends Model<infer D> ? DataOf<D> : never;

const fields = {
  title: 'text',
  score: 'number',
} as const satisfies Record<string, FieldKind>;

const model = defineModel(fields);
type ModelData = InferModel<typeof model>;
```

Call this pattern **definition-driven inference**, **schema-driven inference**, or a **typed builder/factory DSL**. The names vary; the mechanism is the important part:

1. Capture the entire input literal in a generic parameter, preferably a `const` type parameter for inline calls.
2. Preserve literal discriminants, boolean flags, and descriptor tuples. A `const` type parameter cannot recover precision already lost in an earlier variable.
3. Map `keyof D` to output properties. Use conditional types to interpret field descriptors and mapped modifiers to calculate required/optional/readonly fields.
4. Thread `D` or the interpreted type through the constructor/factory result so instances, methods, callbacks, and helpers retain the relationship.
5. Export or use a supported `Infer...` helper for consumers that need the resulting data type. Prefer a library's helper over rebuilding its private type logic.

Avoid inference barriers: broad annotations such as `Record<string, FieldSpec>`, non-generic wrapper functions, broadly annotated returns, `any`, widened arrays, or copying the definition into a different shape. Keep a separate definition literal `as const` and optionally add `satisfies` for validation; otherwise pass an inline literal to a `const`-generic API.

Interpret only semantics guaranteed by the runtime API. A literal `required: true` can drive requiredness; a default value should do so only if the runtime guarantees presence. Preserve one-element descriptor arrays as tuples when they encode element types. Represent explicitly mixed/untyped fields honestly and isolate any library-provided `any`. Distinguish raw data from hydrated/wrapped instances when the runtime changes collection or method types. Keep `strict`/`strictNullChecks` enabled when requiredness depends on null and undefined, and follow the library's documented inference prerequisites.

### Understand excess-property limits

`satisfies` performs useful checks on fresh object literals, but TypeScript remains structurally typed. It does not make every later value “exact.” Do not add recursive `Exact<T>` machinery unless exactness is a genuine API requirement and has focused tests. Prefer runtime schema validation at data boundaries.

## Infer function types deliberately

### Infer implementation returns

```ts
function formatUser(user: { firstName: string; lastName: string }) {
  return `${user.firstName} ${user.lastName}`;
}
```

Annotate parameters when they are not contextually typed; infer the return. A return annotation is justified when it:

- defines a published or cross-module contract;
- intentionally hides fields or widens literals;
- prevents an accidental API change;
- breaks direct or indirect recursive inference;
- describes an overload or assertion function;
- is required by `isolatedDeclarations`;
- reduces declaration size or compiler work in a measured hotspot.

Do not add explicit return types only because a blanket lint rule demands them. Prefer configuring that rule consistently with inference-first design when authorized.

### Use contextual typing for callbacks

```ts
interface Processor {
  (input: string): { normalized: string };
}

const processInput: Processor = (input) => ({
  normalized: input.trim().toLowerCase(),
});
```

The variable annotation establishes the callable contract and contextually types the parameter; the implementation return remains checked against it.

### Prefer unions or optionals over unnecessary overloads

Use a union parameter when every branch returns the same type. Use an optional parameter when overloads differ only by trailing arguments and share a return. Use overloads when caller-visible return types genuinely depend on distinct call shapes and a generic conditional signature would be less clear.

Remember that `ReturnType` and conditional inference over an overloaded function use its final, most permissive signature.

### Let simple predicates infer

```ts
const isNonNullish = <T>(value: T) => value != null;

const values = [0, null, 1, undefined, 2].filter(isNonNullish);
```

TypeScript can infer a predicate for a single-return boolean refinement that does not mutate its parameter. Use an explicit predicate for multi-step guards, and verify its “if and only if” meaning. A predicate that accepts a subset on `true` but lies about the `false` branch is unsound.

## Design inference-friendly generics

### Relate values; do not decorate them

```ts
function getProperty<T, K extends keyof T>(value: T, key: K) {
  return value[key];
}
```

Each type parameter should relate meaningful positions. Strongly reconsider a type parameter used only once.

Push the parameter down:

```ts
function first<T>(values: readonly T[]) {
  return values[0];
}
```

Avoid constraining an entire value to a broad container merely to reach its element; inference may resolve through the constraint and lose precision.

### Preserve caller literals with `const` parameters

```ts
function tuple<const T extends readonly unknown[]>(...values: T) {
  return values;
}

const coordinates = tuple(10, 20, 'north');
```

Here `unknown` is appropriate as a universal meta-level constraint: the function does not inspect elements. Do not replace known application data with `unknown`.

### Select the inference source with `NoInfer`

```ts
function choose<const T extends string>(
  choices: readonly T[],
  fallback?: NoInfer<T>,
) {
  return fallback ?? choices[0];
}

choose(['red', 'green'] as const, 'red');
// choose(['red', 'green'] as const, 'blue'); // error
```

Use `NoInfer` when one parameter validates against a type chosen elsewhere. Do not use it to patch a confused API; split genuinely independent type parameters.

### Preserve argument lists with variadic tuples

```ts
function bindFirst<First, Rest extends readonly unknown[], Result>(
  fn: (first: First, ...rest: Rest) => Result,
  first: First,
) {
  return (...rest: Rest) => fn(first, ...rest);
}
```

Use variadic tuples to retain parameter order and optional/rest information. Avoid degrading wrappers to `Function`, `CallableFunction`, or `(...args: any[]) => any`.

## Compose types without repetition

### Extend object hierarchies

```ts
interface Entity {
  id: string;
  createdAt: Date;
}

interface Customer extends Entity {
  email: string;
}
```

Prefer `interface extends` over `type Child = Parent & {...}` for named object hierarchies. Interfaces report conflicting fields early, display better, and are easier for the compiler to cache.

### Transform instead of copying

```ts
interface Account {
  id: string;
  email: string;
  role: 'admin' | 'member';
  secretHash: string;
}

interface AccountListItem extends Pick<Account, 'id' | 'email' | 'role'> {
  selected: boolean;
}

type AccountUpdate = Partial<Omit<Account, 'id' | 'secretHash'>>;

interface AuditEntry {
  actorId: Account['id'];
  previousRole: Account['role'];
  nextRole: Account['role'];
}
```

Use property references for semantic linkage. Use a fresh primitive when two fields happen to share a representation but may evolve independently.

Name a transformation when it appears more than once or when the expression obscures intent. Avoid deep chains such as `Partial<Pick<Omit<...>>>`; a short interface can be more DRY in practice because readers do not have to reconstruct it repeatedly.

Be careful with utilities over unions. Many mapped utilities operate on the keys common to the union, not independently on each member. Add a distributive helper only when member-wise behavior is required and tested.

Avoid generic `DeepPartial`/`DeepReadonly` helpers over arbitrary values. Functions, tuples, dates, maps, sets, branded values, and class instances need explicit semantics.

## Preserve correlated data

Do not split correlated fields into unrelated unions:

```ts
interface FieldTypes {
  text: string;
  count: number;
  active: boolean;
}

type Field<K extends keyof FieldTypes = keyof FieldTypes> = {
  [P in K]: {
    kind: P;
    value: FieldTypes[P];
    validate: (value: FieldTypes[P]) => boolean;
  };
}[K];

function validateField<K extends keyof FieldTypes>(field: Field<K>) {
  return field.validate(field.value);
}
```

The mapped-and-indexed union keeps `kind`, `value`, and `validate` paired. Use this pattern for event maps, command payloads, and registries. Prefer a direct discriminated union when there are only a few cases.

## Model finite states

```ts
type RequestState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'failure'; error: Error };
```

Prefer a discriminated union over multiple booleans and optional fields that allow impossible combinations. Keep the discriminant literal, required, and stable. Put only properties valid for a state on that member; do not make every member carry optional placeholders.

Derive related state names or subsets from the union rather than restating them. For example, `RequestState<unknown>['status']` is the status union, and `Extract<RequestState<T>, { status: 'success' }>` selects the successful member.

## Narrow known unions with control flow

Use ordinary runtime checks directly when the value already has a precise union type:

```ts
type Notification =
  | { kind: 'email'; address: string }
  | { kind: 'sms'; phoneNumber: string };

function destination(notification: Notification) {
  if (notification.kind === 'email') {
    return notification.address;
  }
  return notification.phoneNumber;
}
```

The discriminant check narrows the first branch; the early return narrows the remainder. The same control-flow analysis understands equality, `typeof`, `instanceof`, `Array.isArray`, `in`, assignments, `if`/`else`, and `switch`. Prefer a discriminant for domain types under your control. Use `in` when a pre-existing structural union has distinct properties and no natural tag.

Do not extract a one-use predicate merely to restate an inline check. Create a named guard when it expresses a reused semantic rule or performs multi-step boundary validation. An explicit `value is T` predicate is an unchecked promise that must be correct in both directions; it is not more type-safe than control flow the compiler can analyze itself.

Normal guards can eliminate every member of a union, at which point the remaining value becomes `never`. A predicate such as `value is never` is almost always a type lie, not a useful guard. Use normal narrowing to reach `never`, then check the impossible remainder.

## Enforce exhaustive decisions

Keep one small project-standard terminator and use it at the impossible remainder of `switch` statements or `if` chains:

```ts
export function assertNever(value: never, message = 'Unexpected value'): never {
  throw new TypeError(`${message}: ${String(value)}`);
}

function renderState<T>(state: RequestState<T>) {
  switch (state.status) {
    case 'idle':
      return 'Idle';
    case 'loading':
      return 'Loading';
    case 'success':
      return `Loaded: ${String(state.data)}`;
    case 'failure':
      return state.error.message;
    default:
      return assertNever(state, 'Unknown request state');
  }
}
```

If every member is handled, control-flow analysis narrows `state` to `never` and the call compiles. Adding a member makes the argument fail to typecheck until its case is implemented. The throw also protects runtime code from unvalidated input, stale declarations, or client/server version skew. The explicit `: never` return is justified because it is the helper's control-flow contract.

Do not use a generic `default` that returns a fallback for a closed domain; it masks newly added members. A bare throw catches unexpected runtime data but does not prove compile-time completeness because it accepts every type. `value satisfies never` can perform the static check without a helper, but it supplies no runtime policy by itself; prefer the reusable terminator when unexpected runtime values are possible or when it reads more clearly.

Use a complete lookup instead of a `switch` when the operation is data mapping:

```ts
const stateLabels = {
  idle: 'Idle',
  loading: 'Loading',
  success: 'Loaded',
  failure: 'Failed',
} as const satisfies Record<RequestState<unknown>['status'], string>;
```

Adding a status now fails at the table. This is often more readable and more reusable than repeating branching logic. An explicit function return type combined with `strictNullChecks`/`noImplicitReturns` can expose some incomplete paths, and a type-aware switch-exhaustiveness lint rule can find missing cases, but neither replaces a clear domain model.

The typescript-eslint rule requires typed linting. Keep `considerDefaultExhaustiveForUnions: false` when new members must receive explicit cases; otherwise a generic default can mask them. Set `allowDefaultCaseForExhaustiveSwitch: false` only when redundant defaults are genuinely unwanted. If runtime values can exceed the compile-time union, retain the throwing default and permit it globally or with a focused rule exception.

## Use `never` deliberately

`never` is the bottom type: it represents a value that cannot occur. It is assignable to every type, no ordinary value is assignable to it, and it disappears from unions such as `Result | never`. These properties make it useful in a few focused roles.

### Mark code that cannot complete normally

```ts
function fail(message: string): never {
  throw new Error(message);
}

function requireApiKey(apiKey: string | undefined) {
  return apiKey ?? fail('API key is required');
}
```

The call tells control-flow analysis that execution cannot continue through that branch, so the returned expression is `string`. Use `never` only when every path throws, exits, or loops forever. Use `void` when a function completes normally without producing a useful result. An async function that can never fulfill may return `Promise<never>`; do not use that type if it can resolve successfully.

Inference has an intentional compatibility nuance: an unannotated throwing function declaration infers `void`, while a throwing function expression or arrow infers `never`; async forms similarly infer `Promise<void>` versus `Promise<never>`. Give shared declaration-style failure and exhaustiveness helpers an explicit `never` return so their control-flow contract is unambiguous. Continue inferring ordinary implementation returns.

### Remove impossible members in type transformations

Distributive conditional types use `never` to filter union members. Prefer the standard utilities `Exclude` and `Extract` when they express the operation. Use a custom conditional only for a domain-specific relationship:

```ts
type WithPayload<T> = T extends { payload: unknown } ? T : never;
```

Mapped types can remove keys by remapping an unwanted key to `never`:

```ts
type PickByValue<T, Value> = {
  [K in keyof T as T[K] extends Value ? K : never]: T[K];
};
```

Because conditional types distribute over a naked type parameter, `[T] extends [never]` is the reliable rare-case test for whether a type itself is `never`:

```ts
type IsNever<T> = [T] extends [never] ? true : false;
```

Use `IsNever` in type-library tests or a genuinely generic abstraction, not normal application code. A type-level API may also default a covariant result branch to `never` when “no result is possible” is the honest neutral member of a union. Name and document that meaning rather than exposing unexplained `never` diagnostics to callers.

### Describe a callable that will not be invoked

For type-level inspection or a generic constraint, this avoids `any` while accepting arbitrary parameter lists:

```ts
type AnyFunction = (...args: never[]) => unknown;
```

This is appropriate only when the implementation never calls the function, such as extracting its return type. If code invokes or wraps the function, capture its arguments and result with generic variadic tuples so the relationship is preserved.

### Forbid a property only when no natural discriminant fits

An optional `never` property can express an exclusive structural choice:

```ts
type LinkAction =
  | { href: string; onClick?: never }
  | { href?: never; onClick: () => void };
```

Prefer a named discriminant when the domain permits one; it narrows and explains better. Use the exclusive-property pattern for APIs whose runtime shape is already fixed, name the union clearly, and enable `exactOptionalPropertyTypes` so `property?: never` cannot be bypassed with an explicit `undefined`.

### Treat unexpected `never` as a diagnostic

An inferred `never` often means the compiler has no evidence or found a contradiction:

- an empty mutable literal became `never[]`;
- control flow eliminated every possible member;
- an intersection or generic constraint was impossible;
- a conditional type matched no union member or a mapped type removed every key.

Investigate which relationship collapsed. Add inference evidence, correct the domain model, or fix the condition; do not reach first for `as never`. Outside an exhaustiveness helper or type-level constraint, a public parameter of type `never` creates an API nobody can call. `never` is not runtime validation, and a cast or explicit predicate that forces a value to `never` defeats the refactor protection it is meant to provide.

## Narrow boundary data safely

Use `unknown` for values that truly may be anything: parsed JSON, caught errors, postMessage payloads, storage, environment decoding, and untyped dependencies. Narrow it at the boundary and return a precise domain type.

Choose a more meaningful alternative when possible: use a generic `T` to preserve an arbitrary caller type, `object` for any non-primitive when no properties are read, `void` for an ignored callback result, and `never` for an impossible value. Avoid `{}` as a broad type because it accepts every non-nullish primitive, and avoid boxed `Object`, `String`, `Number`, and `Boolean` types.

Prefer these runtime checks:

- `typeof` for primitives, while remembering `typeof null === 'object'`;
- `Array.isArray` for arrays;
- `instanceof` for trustworthy class realms;
- equality/discriminants for unions;
- `in` only after proving a non-null object;
- schema validation when shape complexity or error reporting warrants it.

Avoid broad truthiness checks if zero, empty string, or false are valid. Do not claim that a cast validates JSON.

An explicit type predicate must prove every field needed by its target and remain correct as the target evolves. Consider colocating guards with types, adding tests, or using the project's schema library to reduce drift.

Use assertion functions when failure should throw:

```ts
function assertString(value: unknown, name: string): asserts value is string {
  if (typeof value !== 'string') {
    throw new TypeError(`${name} must be a string`);
  }
}
```

Assertion signatures require explicit return annotations because the annotation itself controls narrowing.

## Repair type mismatches before asserting

Treat a rejected assignment or assertion as a request to explain a relationship. Before overriding it:

1. inspect the source declaration and the runtime value it describes;
2. find lost inference, widened literals, missing generics, wrong overloads, or an incorrect augmentation;
3. narrow or validate unions and boundary data;
4. reconcile readonly versus mutable collections, tuples, brands, and library-specific wrappers;
5. change the destination contract if it is the inaccurate side;
6. reuse a sound project helper or add a focused adapter only after the relationship is understood.

Do not mechanically replace one escape hatch with another. A non-null assertion, generic helper, overload, predicate, or `@ts-expect-error` can hide the same false claim as a cast.

### Normalize nullish values truthfully

Use `??`, not `||`, when only `null` and `undefined` mean absence. In the ordinary case, contextual and union inference already preserves the element type:

```ts
declare const maybeTags: string[] | null | undefined;

const tags = maybeTags ?? [];
```

If this does not produce the needed type, diagnose why. Common causes are a source typed as `unknown` or `object`, a lost generic relationship, mutable/readonly disagreement, an exact tuple or brand, a union whose correlations were lost, or a dependency collection with runtime methods that a plain array does not have.

For a genuinely repeated defaulting operation, require the fallback to be a real value of the inferred type:

```ts
function valueOr<T>(
  value: T | null | undefined,
  fallback: NoInfer<T>,
) {
  return value ?? fallback;
}
```

`NoInfer<T>` keeps the nullable value authoritative and prevents the fallback from widening `T`. This helper accepts `[]` when `T` is an ordinary array and `{}` when `T` is a property bag whose fields are all optional. It correctly rejects them for non-empty tuples, required object shapes, brands, array subclasses, and library wrapper collections. For an exact subtype, require a fallback or factory that constructs that actual subtype. If consumers need only a broader view, return that honest view—such as `readonly Element[]`—instead of promising the wrapper type.

Do not write a helper equivalent to either of these patterns:

- `return value ?? ([] as T)` for arbitrary array-like `T`;
- `return value ?? ({} as T)` for arbitrary `T extends object`.

An empty array is not every tuple, branded collection, subclass, or library document array. An empty object does not satisfy arbitrary required properties. Hiding the assertion in a shared helper increases its blast radius and makes every call appear trustworthy. Overloads are appropriate only when every signature truthfully describes the same runtime implementation; they do not repair an impossible default.

Search project utilities and the dependency's supported APIs before introducing a helper. Prefer a boundary-specific name when behavior is domain-specific. Test mutable and readonly inputs, nullish inputs, exact subtypes, and the runtime identity or methods of library collections that must be preserved.

## Control assertions and escape hatches

Allow `as` only for:

- a runtime invariant checked immediately before the assertion;
- a nominal brand applied by a validated constructor;
- interop with an incomplete or incorrect external declaration;
- a known compiler limitation documented by a focused test;
- intentional broadening that cannot be expressed more clearly with an annotation.

A normal assertion is permitted only when the source and target overlap enough that one could be a more or less specific view of the other. `value as unknown as T` deliberately bypasses that compiler check: the intermediate `unknown` is assignable from the source, and the second assertion can claim an unrelated target. It is no safer than `as any` in this use because no narrowing occurs, and it can be worse by giving downstream code a precise but fabricated type. It can invent fields, methods, discriminants, brands, mutability, or collection behavior and suppress the refactor errors that should expose the mismatch.

Keep assertions local and never export asserted raw data as though it were validated. Do not use a generic assertion helper to make arbitrary conversions look routine. Permit a double assertion only when verified runtime identity conflicts with an external declaration or a known compiler limitation and no correction, augmentation, narrowing, validation, overload, or typed adapter can express the fact. Every remaining double assertion must have an adjacent comment that states:

- the runtime evidence that makes the target true;
- why the source declaration or compiler model is insufficient;
- an issue, specification, or removal condition when one exists.

Add a focused type regression and a runtime test of the asserted behavior. Confine any lint suppression to the exact expression.

Use non-null `!` to consume a possibly nullish value only when lifecycle or framework behavior proves presence and no safer initialization or guard can express it. Prefer narrowing, constructor initialization, an assertion function, or explicit missing-value handling.

A literal `null!` is a distinct, narrowly permitted escape hatch when the program intentionally must emit `null` and the immediate consumer's verified runtime contract accepts the sentinel, but the destination type intentionally omits that construction-only value or has an inaccurate declaration:

```ts
renderRows([
  firstRow,
  // The renderer consumes null as a separator before rows reach typed callbacks.
  null!,
  secondRow,
]);
```

Under strict TypeScript checking, `null!` has type `never`, which is assignable to the destination, while the emitted JavaScript value remains `null`. It does not prove non-nullness or make the runtime safe. Use it only on the exact sentinel expression, always explain the mismatch, and test the consuming runtime path. Ensure the immediate consumer removes or interprets the sentinel before code promised a non-null value can observe it. Do not use it to read a nullable value, initialize a field that is not ready, or make ordinary application types deny a real `null` state. If the pattern repeats, repair or augment the declaration, create a boundary-specific adapter with an honest input type, or contribute the upstream fix.

Use `@ts-expect-error` only for an intentional negative type test or a documented external typing defect. It is useful for a temporary defect because it reports when the suppression becomes unnecessary, but it suppresses every diagnostic on the following line; isolate the expression and include the reason. Do not use `@ts-ignore`.

## Represent mutability and variance

Accept the least mutable input the implementation needs:

```ts
function total(values: readonly number[]) {
  return values.reduce((sum, value) => sum + value, 0);
}
```

This accepts mutable and readonly arrays while preventing mutation inside the function. Return mutable collections only when callers are intended to mutate them.

With `strictFunctionTypes`, function-property parameter types are checked more safely than method syntax in some variance scenarios. Use property callbacks when accepting consumer-supplied handlers whose parameter soundness matters:

```ts
interface Consumer<T> {
  consume: (value: T) => void;
}
```

Do not add explicit `in`/`out` variance annotations routinely. TypeScript infers variance; explicit annotations are specialized tools for correctness investigation or measured performance and must match structural behavior.

With `exactOptionalPropertyTypes`, `field?: T` means absent or present with `T`, not present with `undefined`. Write `field?: T | undefined` only when explicit `undefined` is meaningful.

## Use mapped and conditional types intentionally

### Remap keys when the runtime API does the same

```ts
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
```

Mapped types should mirror a real systematic relationship. Prefer a direct interface when members are curated independently.

### Control distribution

```ts
type BoxEach<T> = T extends unknown ? { value: T } : never;
type BoxWhole<T> = [T] extends [unknown] ? { value: T } : never;

type Each = BoxEach<string | number>;
// { value: string } | { value: number }

type Whole = BoxWhole<string | number>;
// { value: string | number }
```

A naked type parameter distributes over unions. Tuple-wrap both sides when the union must be evaluated as a whole.

### Use `infer` to extract structure

Prefer built-in utilities when available. Write a custom conditional extractor only for domain-specific structure:

```ts
type EventPayload<T> = T extends { payload: infer Payload } ? Payload : never;
```

Avoid unbounded recursive conditionals and union cross-products. Add recursion-depth controls only when the domain truly needs recursion; otherwise model the concrete supported depths.

TypeScript 7 treats Unicode code points—not UTF-16 halves—when inferring through template literal character patterns. Read [typescript-7.md](typescript-7.md) before relying on type-level string splitting across versions.

## Use nominal distinctions only when valuable

Structural strings do not distinguish identifiers. Brand only when mixing same-shaped values would be a meaningful bug:

```ts
declare const userIdBrand: unique symbol;

export type UserId = string & { readonly [userIdBrand]: true };

export function parseUserId(value: string) {
  if (!/^usr_[a-z0-9]+$/.test(value)) {
    throw new TypeError('Invalid user id');
  }
  return value as UserId;
}
```

The localized assertion is justified by runtime validation. Export constructors/parsers rather than inviting consumers to assert brands themselves. Prefer a wrapper object if runtime identity and serialization overhead are acceptable.

## Handle async code and errors

- Infer internal async returns; annotate published promises when the resolved contract must be stable.
- Use `Awaited<ReturnType<typeof fn>>` to derive resolved results.
- Preserve rejection semantics; do not annotate a function as returning `Promise<Result | Error>` unless errors are actually values.
- Treat caught values as `unknown`; narrow with `instanceof Error` or a project-specific guard before reading properties.
- Await or deliberately return promises. Mark intentionally detached promises with the project's established pattern and handle rejection.
- Do not pass async functions to callbacks that ignore returned promises unless the API explicitly supports them.

## Keep types readable and fast

- Judge complexity across the type implementation and every call site. Concentrating complexity in one reusable abstraction can be substantially simpler than repeating annotations, assertions, overload logic, or manual narrowing throughout the codebase.
- Make advanced abstractions feel intuitive to consumers: infer expected types, reject invalid combinations at the relevant expression, and avoid requiring callers to understand the internal machinery.
- Give sophisticated types a domain-relevant name, concise explanation of the relationship they preserve, focused positive and negative type tests, and actionable diagnostics.
- Prefer interface extension over intersections for named objects.
- Name complex conditional or mapped results so the compiler can cache and humans can inspect them.
- Prefer base interfaces over very large unions when the behavior is genuinely shared.
- Avoid intersecting large unions, deeply recursive transformations, and repeatedly inlined conditional return types.
- Add return annotations at measured hot declarations, not universally.
- Use project references and focused `include`/`types` settings for large repositories.
- Do not assume a `Simplify<T>` mapped type improves compiler performance; it often changes only display and may hide useful aliases.
- Measure with compiler diagnostics/traces before making code less clear for hypothetical performance.
- Replace an advanced type only when the alternative reduces total complexity or improves correctness, usage, diagnostics, or measured compiler health—not merely because the implementation is sophisticated.

## Recognize common failure modes

- Writing the same field in several interfaces instead of extending, picking, omitting, or indexing the source.
- Annotating a config literal and losing literal keys/values that callers need.
- Widening a schema/definition literal before it reaches its generic factory, losing field names, literal flags, descriptor tuples, or requiredness.
- Reimplementing a library's schema inference instead of using its supported `Infer...` helper.
- Expecting `satisfies` to turn an empty mutable literal into the target collection type instead of annotating the container.
- Applying `as const` to mutable application state.
- Exporting a huge anonymous inferred type that accidentally becomes a public API.
- Replacing a known type with `unknown` and forcing every consumer to narrow again.
- Using `{}`, boxed primitive types, or `Function` when `object`, primitive types, or a precise call signature is intended.
- Replacing an error with `any`, a double assertion, a non-null assertion, a misleading overload, or an assertion hidden inside a generic helper.
- Returning `[] as T` or `{} as T` for an arbitrary generic even though `T` may require tuple elements, properties, a brand, a subclass, or library-specific runtime behavior.
- Using `||` for a nullish default and accidentally replacing valid falsy values.
- Using `null!` as routine null handling instead of reserving it for a documented intentional sentinel consumed at a verified boundary.
- Creating a generic whose parameter does not relate values.
- Letting a default argument widen the authoritative generic instead of using `NoInfer`.
- Losing correlation by using parallel unions such as `kind: Kinds; payload: Payloads`.
- Modeling mutually exclusive states with optional fields and booleans.
- Writing a reusable predicate for a one-line discriminant check instead of letting local control flow narrow the value.
- Using a permissive `default` that prevents a new union member from producing an exhaustive-handling error.
- Throwing from a default branch without checking that the remaining value is `never`.
- Casting to `never`, declaring a `value is never` predicate, or otherwise forcing an impossible state instead of proving it.
- Annotating a function `never` even though a path can complete or an async operation can fulfill.
- Treating an unexpected `never` or `never[]` as something to silence instead of evidence of missing inference input or a contradictory type relationship.
- Using optional `never` properties for exclusivity when a named discriminant would explain and narrow the domain better.
- Assuming `satisfies` changes the expression's resulting type or performs runtime validation.
- Assuming `readonly` or `as const` freezes runtime data.
- Using a type predicate that proves only its true branch.
- Casting `Object.keys` globally to `(keyof T)[]` even though runtime objects may have extra keys.
- Applying `Pick`/`Omit` to a union without checking whether member-wise distribution is required.
- Publishing a deep utility type whose error messages and compile cost exceed its value.

## Primary references

- [TypeScript narrowing, control-flow analysis, predicates, discriminated unions, and exhaustiveness](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)
- [TypeScript `never` semantics and return inference](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-0.html#the-never-type)
- [TypeScript control-flow support for `never`-returning functions and assertion signatures](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#better-support-for-never-returning-functions)
- [TypeScript inferred type predicates and their “if and only if” requirement](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-5.html#inferred-type-predicates)
- [TypeScript conditional and distributive conditional types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html)
- [TypeScript mapped-type key remapping and filtering](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html)
- [TypeScript `NoInfer`, `Exclude`, `Extract`, and other utility types](https://www.typescriptlang.org/docs/handbook/utility-types.html#noinfertype)
- [TypeScript type assertions and non-null assertions](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#type-assertions)
- [TypeScript `@ts-expect-error` behavior and tradeoffs](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-9.html#ts-expect-error-comments)
- [typescript-eslint unsafe type assertion rule](https://typescript-eslint.io/rules/no-unsafe-type-assertion/)
- [typescript-eslint non-null assertion rule](https://typescript-eslint.io/rules/no-non-null-assertion/)
- [typescript-eslint switch exhaustiveness rule and default-case tradeoffs](https://typescript-eslint.io/rules/switch-exhaustiveness-check/)
- [Google TypeScript style guide on assertions, double assertions, and explanatory comments](https://google.github.io/styleguide/tsguide.html#type-and-non-nullability-assertions)
- [Effective TypeScript: sources of unsoundness and limiting assertion scope](https://effectivetypescript.com/2021/05/06/unsoundness/)
- [Mongoose raw versus hydrated subdocument-array types](https://mongoosejs.com/docs/typescript/subdocuments.html#subdocument-arrays)
