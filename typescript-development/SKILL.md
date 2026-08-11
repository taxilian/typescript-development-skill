---
name: typescript-development
description: Develop, refactor, review, debug, configure, run, and migrate TypeScript 7 code with runtime-faithful, readable, inference-first, strict, and DRY type design. Use for .ts, .tsx, .mts, .cts, .d.ts, tsconfig files, TypeScript package APIs, type-heavy application or library code, advanced generics, schema-driven or definition-driven inference, control-flow narrowing, discriminated unions, never and exhaustiveness, declaration merging, global or module augmentation, ambient modules, globalThis, Node and Express extension points, inferred configuration, eliminating any or unnecessary unknown, shared domain models, TypeScript 6-to-7 migration, compiler and type errors, or development loops involving tsc watch mode, tsc-watch, tsx, process restarts, and package scripts.
---

# TypeScript Development

Build code whose runtime behavior and domain relationships the compiler describes truthfully and humans can understand quickly. Use advanced types to remove duplication and catch real mistakes, not to maximize type cleverness.

## Apply priorities in order

Use this order to resolve tradeoffs:

1. **Runtime truth and domain correctness.** Model values the program can actually receive, validate untrusted inputs, and never use an assertion or predicate to tell a convenient lie.
2. **Readability and understandability.** Prefer recognizable domain models, ordinary control flow, and focused helpers over compressed type puzzles.
3. **One source of truth and reuse.** Derive linked values and types instead of copying them, but name or simplify a derivation when readers would otherwise have to decode it repeatedly.
4. **Refactor safety.** Make new union members, keys, states, and schema fields produce useful errors at every decision or lookup that must change.
5. **Precise local inference with low annotation noise.** Preserve useful literals and relationships while writing explicit contracts only where they add meaning.
6. **Measured compiler and tooling health.** Keep diagnostics actionable and optimize type-checking or declaration size only from evidence.

The first priority is intentionally ahead of readability: a simple type that misrepresents runtime data is still wrong. Readability is intentionally ahead of DRY and refactor machinery: do not replace a short, clear domain type with an opaque transformation merely to eliminate a few repeated tokens.

## Load the right reference

- Read [type-system-playbook.md](references/type-system-playbook.md) when designing or refactoring configs, exported types, generics, guards, discriminated unions, exhaustive decisions, deliberate `never` usage, conditional or mapped types, assertions, or `any`/`unknown` usage.
- Read [type-augmentation.md](references/type-augmentation.md) when typing existing globals, adding properties supplied by middleware or plugins, extending third-party declarations, declaring untyped modules or assets, or designing an augmentable library API.
- Read [typescript-7.md](references/typescript-7.md) when installing, configuring, migrating, compiling, linting, integrating editor/build tooling, or relying on TypeScript 7 behavior. Re-verify its time-sensitive compatibility notes against current primary documentation before changing dependencies.
- Read [development-loop.md](references/development-loop.md) when choosing or changing watch mode, source execution, emitted execution, process restart hooks, source maps, or `package.json` development scripts.
- Read [review-checklist.md](references/review-checklist.md) before finishing a material TypeScript implementation, migration, or review.

## Apply the working contract

1. Make every type match the values and behavior that can occur at runtime. TypeScript types are erased; validate external data.
2. Prefer the simplest readable representation that preserves the required relationships. A clever type that obscures behavior is a defect.
3. Keep one authoritative definition. Derive related types with `typeof`, `keyof`, indexed access, `ReturnType`, `Parameters`, `Awaited`, `Pick`, `Omit`, or interface extension.
4. Infer local values and function returns by default.
5. Annotate only where the annotation expresses a real contract, breaks a cycle, enables an intentional abstraction or assertion signature, marks a deliberately non-returning function, satisfies `isolatedDeclarations`, or fixes a measured compiler/declaration performance problem.
6. Validate object literals with `satisfies`; do not erase their useful inferred detail with a broad annotation.
7. Use `as const` for configuration and lookup data when literal preservation and readonly intent are correct. Do not use it as decoration or claim runtime immutability.
8. Keep `unknown` at genuinely untrusted or universal boundaries and narrow it immediately. Keep `any` inside the smallest documented interoperability adapter when no precise alternative exists.
9. Treat type assertions, explicit predicates, assertion signatures, and non-null assertions as proof obligations. Never use them merely to silence the compiler.
10. Prefer interfaces and `extends` for named object hierarchies. Prefer type aliases for unions, tuples, primitives, mapped types, conditional types, and other transformations.
11. Model finite states with discriminated unions and make decisions over them exhaustive. Treat an unexpected `never` as a diagnostic to investigate, not a type to cast away.
12. Separate type-checking, emission, execution, and restart responsibilities. Give process restarts to exactly one tool, and never mistake fast transpilation for type-checking.
13. Treat augmentation as a runtime proof obligation. Target the exact existing declaration, ensure the declaration file is loaded, and create or validate the augmented member before use.

Use this escalation order when the compiler needs help:

`inference` → `satisfies` → `annotation` → localized `assertion` → isolated `any`

## Follow the agentic workflow

### 1. Discover the actual project

- Read repository instructions and inspect `package.json`, the full `tsconfig` extension chain, lockfile, source layout, framework/build tooling, lint config, and existing tests.
- Run the local compiler version command or package-manager equivalent. Do not assume the project is on TypeScript 7 because the request mentions it.
- Determine whether the code is an application, an internal package, or a published library. Public API and declaration requirements change annotation decisions.
- Identify existing typecheck, lint, test, and build scripts. Preserve the project's package manager and conventions.
- Identify the production runtime and the current owner of type-checking, emission, source execution, and process restarts. Avoid adding a second watcher for the same responsibility.
- Inspect existing `.d.ts` files and dependency declarations before augmenting a global or module. Find the documented open interface and the exact module specifier that owns it.
- Inspect nearby types and values before inventing a new type. Search for the domain concept, not only the proposed identifier.

### 2. Identify sources of truth and boundaries

For every requested type change, identify:

- the authoritative runtime value, domain contract, schema, or function;
- the inputs that are trusted and those that require validation;
- the consumers that need a stable named type;
- mutability requirements;
- the relationship that inference must preserve;
- whether a configuration/schema literal is a definition language whose exact keys, flags, or tuple shapes must flow through a generic API.
- whether an alleged global or plugin-provided member exists at runtime, on which objects, and after which initialization step.

Choose value-first design when runtime data is authoritative. Choose a named interface first when an external or published contract is authoritative. Do not create a value/type dependency cycle merely to avoid writing an interface.

### 3. Choose the narrowest useful typing mechanism

| Situation | Default | Annotate or change course when |
| --- | --- | --- |
| Local function | Infer its return | Recursive cycle, intentional widening, overload contract, or measured performance issue |
| Exported application helper | Usually infer | Other modules require a deliberately stable contract |
| Published library export | Deliberate named return contract | Inference is intentionally part of the API and generated declarations are reviewed |
| Config or lookup literal | `as const satisfies Contract` | Consumers require mutation; then use `satisfies` without `as const` |
| Schema/definition-driven API | Preserve the full literal and pass it directly, or use `as const satisfies`; use the library's inference helper | An external contract is authoritative or the library documents an inference gap |
| Core type produced by a function | `ReturnType<typeof createThing>` | A separately versioned external contract is the real authority |
| Type tied to a property | `Model['property']` | The property relationship is accidental rather than semantic |
| Untrusted input | `unknown`, schema/guard, then domain type | A framework already returns a validated precise type |
| Existing global/module member missing from types | Documented interface or module augmentation plus runtime proof | The shape is local to one pipeline; use a generic, refined type, or wrapper instead |
| Reusable object hierarchy | `interface Child extends Parent` | The result is a union or type-level transformation |
| Literal-preserving generic API | `const` type parameter | Callers need widening or mutation |
| Generic inference has two candidate sources | `NoInfer<T>` on the non-authoritative source | Separate type parameters represent genuinely different values |
| Known discriminated union inside a function | Inline `if`, `switch`, equality, or `in` narrowing | A reusable semantic check or complex boundary validation justifies a named guard |
| Finite decision over a union or enum | Explicit cases plus a reusable `assertNever`-style terminator | A complete `Record` lookup communicates a data mapping more clearly |
| Complete lookup over finite keys | Literal with `satisfies Record<Keys, Value>` | Missing keys have a deliberate, validated runtime fallback |

### 4. Implement inference-first

Use `satisfies` to check shape while retaining useful property types:

```ts
interface ServiceConfig {
  readonly mode: 'development' | 'production';
  readonly retries: number;
  readonly plugins: readonly string[];
}

export const serviceConfig = {
  mode: 'production',
  retries: 3,
  plugins: ['metrics', 'tracing'],
} as const satisfies ServiceConfig;

export type ConfiguredServiceMode = typeof serviceConfig.mode;
```

Derive a core type from its constructor when the constructor is authoritative:

```ts
export function createSession(userId: string, issuedAt = new Date()) {
  return {
    userId,
    issuedAt,
    expiresAt: new Date(issuedAt.getTime() + 3_600_000),
  };
}

export type Session = ReturnType<typeof createSession>;
```

Keep related domain shapes linked:

```ts
interface User {
  id: string;
  email: string;
  displayName: string;
  createdAt: Date;
}

interface UserSummary extends Pick<User, 'id' | 'displayName'> {
  label: string;
}

type UserDraft = Omit<User, 'id'> & Partial<Pick<User, 'id'>>;

interface SearchHit {
  userId: User['id'];
  title: string;
}
```

Do not duplicate fields merely to make a new type look self-contained. Do not stack utilities until the relationship becomes harder to understand than a small named interface.

### 5. Contain uncertainty at runtime boundaries

Prefer a project-standard runtime schema when one already exists. Otherwise, narrow with runtime checks that prove every property used:

```ts
interface WidgetPayload {
  id: string;
  enabled: boolean;
}

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === 'object' && value !== null && !Array.isArray(value);
}

function isWidgetPayload(value: unknown): value is WidgetPayload {
  return (
    isRecord(value) &&
    typeof value['id'] === 'string' &&
    typeof value['enabled'] === 'boolean'
  );
}

export function parseWidgetPayload(text: string) {
  const value: unknown = JSON.parse(text);
  if (!isWidgetPayload(value)) {
    throw new TypeError('Invalid widget payload');
  }
  return value;
}
```

Prefer inferred type predicates when TypeScript can prove them. Treat explicit `value is T` and `asserts value is T` signatures as handwritten promises and test both their true and false cases.

If `any` is unavoidable, require all of the following:

- place it in an adapter at an untyped dependency or migration boundary;
- prevent it from appearing in exported types or normal business logic;
- validate or convert it before returning;
- add a concise comment naming the external limitation;
- keep `no-explicit-any` and the `no-unsafe-*` lint family enabled elsewhere when compatible tooling is available.

### 6. Design generics for inference

- Make each type parameter relate at least two meaningful positions. Remove decorative generics.
- Push type parameters down to the value shape; constrain only capabilities actually used.
- Use the fewest type parameters that preserve the relationship.
- Accept `readonly` collections when the function does not mutate them.
- Use `const` type parameters in library/builders APIs that should capture caller literals without requiring `as const` at every call.
- For schema, route, command, and similar definition APIs, capture the whole definition in one generic and thread it into the returned model/instance type. Do not widen it in an intermediate variable or non-generic helper.
- Use `NoInfer<T>` to stop a fallback/default argument from widening the type inferred from the authoritative argument.
- Make conditional-type distribution intentional; wrap both sides in tuples when the union must be treated as a whole.
- Name and test complex mapped, recursive, or conditional types. Stop if a direct interface, discriminated union, or overload is clearer.

### 7. Model, narrow, and exhaust states

Prefer discriminated unions for mutually exclusive states. Avoid optional-property bags that permit impossible combinations.

Narrow a known union with ordinary JavaScript control flow first: a discriminant comparison, equality, `typeof`, `instanceof`, `Array.isArray`, or `in`. TypeScript follows `if`/`else`, early returns, assignments, and `switch`; do not extract a one-use type guard merely to restate a check the compiler already understands. Use a named predicate when the runtime test is complex or reused, and remember that an explicit predicate promises both its true and false branches.

For every finite decision, handle each member explicitly and terminate the impossible remainder with a project-standard function accepting `never` and returning `never`. Its explicit return type is justified because it declares a control-flow contract and preserves a runtime error for unvalidated or version-skewed values. Do not use a permissive `default` that silently handles future members.

Prefer a complete `Record` literal checked with `satisfies` when the operation is a static mapping rather than behavior. Adding a union member should fail at the lookup definition just as it should fail at an exhaustive branch.

Do not use truthiness to narrow when `0`, `''`, or `false` is a valid value. Prefer equality, `typeof`, `instanceof`, `Array.isArray`, `in`, or a discriminant.

### 8. Verify behavior and types

- Run the repository's narrowest relevant typecheck, lint, and test commands, then broader checks in proportion to the change.
- When changing the development loop, verify both a type error and a runtime edit: confirm the intended process restarts once, failed builds behave deliberately, source maps point to TypeScript, and shutdown releases ports and resources.
- Run a production-equivalent build and entry point in CI or before release. Do not let `tsx`, path aliases, or a development loader hide module-resolution or emit problems in the deployed runtime.
- With a `tsconfig.json` present, do not pass source file paths directly to TypeScript 7 unless intentionally using `--ignoreConfig`; prefer the project script or `tsc -p`.
- For public libraries, inspect emitted declarations or API reports. Ensure inferred exports did not expose private implementation details, giant anonymous types, or accidental literals.
- Add type-level regression checks for important inference: positive assignments/`satisfies` checks and focused `@ts-expect-error` negative cases with explanations.
- For a changed finite union, enum, event map, or key set, verify that adding a temporary member fails every exhaustive branch and complete lookup that requires an update.
- Use a type-aware switch-exhaustiveness lint rule as a supplement when the project already supports typed linting; configure it so a generic `default` does not hide newly added union members.
- For augmentation, typecheck a consumer that imports the real public entry point, verify the augmentation file appears in `--explainFiles` or equivalent output, and test the runtime initialization path independently.
- Search the diff for new `any`, `unknown`, `as`, `!`, `@ts-ignore`, duplicated fields, broad index signatures, explicit return types, and unusual `never` uses. Justify each exception.
- Do not weaken compiler or lint settings to make a local error disappear.

### 9. Report the result

Lead with the implemented behavior. Name the source of truth for derived types, any deliberate public annotations, every remaining escape hatch, and the verification commands run. If tooling prevented full verification, state the exact gap and its impact.
