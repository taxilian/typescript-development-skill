---
name: typescript-development
description: Develop, refactor, review, debug, configure, run, and migrate TypeScript 7 code with inference-first, strict, DRY type design and runtime-safe boundaries. Use for .ts, .tsx, .mts, .cts, .d.ts, tsconfig files, TypeScript package APIs, type-heavy application or library code, advanced generics, schema-driven or definition-driven inference, declaration merging, global or module augmentation, ambient modules, globalThis, Node and Express extension points, inferred configuration, eliminating any or unnecessary unknown, shared domain models, TypeScript 6-to-7 migration, compiler and type errors, or development loops involving tsc watch mode, tsc-watch, tsx, process restarts, and package scripts.
---

# TypeScript Development

Build code whose relationships the compiler can prove and humans can still understand. Optimize for precise inference, one source of truth, safe boundaries, and simple public contracts—not maximum type cleverness.

## Load the right reference

- Read [type-system-playbook.md](references/type-system-playbook.md) when designing or refactoring configs, exported types, generics, guards, conditional or mapped types, assertions, or `any`/`unknown` usage.
- Read [type-augmentation.md](references/type-augmentation.md) when typing existing globals, adding properties supplied by middleware or plugins, extending third-party declarations, declaring untyped modules or assets, or designing an augmentable library API.
- Read [typescript-7.md](references/typescript-7.md) when installing, configuring, migrating, compiling, linting, integrating editor/build tooling, or relying on TypeScript 7 behavior. Re-verify its time-sensitive compatibility notes against current primary documentation before changing dependencies.
- Read [development-loop.md](references/development-loop.md) when choosing or changing watch mode, source execution, emitted execution, process restart hooks, source maps, or `package.json` development scripts.
- Read [review-checklist.md](references/review-checklist.md) before finishing a material TypeScript implementation, migration, or review.

## Apply the working contract

1. Infer local values and function returns by default.
2. Annotate only where the annotation expresses a real contract, breaks a cycle, enables an intentional abstraction, satisfies `isolatedDeclarations`, or fixes a measured compiler/declaration performance problem.
3. Keep one authoritative definition. Derive related types with `typeof`, `keyof`, indexed access, `ReturnType`, `Parameters`, `Awaited`, `Pick`, `Omit`, or interface extension.
4. Validate object literals with `satisfies`; do not erase their useful inferred detail with a broad annotation.
5. Use `as const` for configuration and lookup data when literal preservation and readonly intent are correct. Do not use it as decoration or claim runtime immutability.
6. Keep `unknown` at genuinely untrusted or universal boundaries and narrow it immediately. Keep `any` inside the smallest documented interoperability adapter when no precise alternative exists.
7. Treat type assertions and non-null assertions as proof obligations. Never use them merely to silence the compiler.
8. Prefer interfaces and `extends` for named object hierarchies. Prefer type aliases for unions, tuples, primitives, mapped types, conditional types, and other transformations.
9. Prefer the simplest readable representation that preserves the required relationships. A clever type that obscures behavior is a defect.
10. Remember that TypeScript types are erased. Validate external data and preserve runtime behavior.
11. Separate type-checking, emission, execution, and restart responsibilities. Give process restarts to exactly one tool, and never mistake fast transpilation for type-checking.
12. Treat augmentation as a runtime proof obligation. Target the exact existing declaration, ensure the declaration file is loaded, and create or validate the augmented member before use.

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

### 7. Model states, not flags

Prefer discriminated unions for mutually exclusive states. Narrow with runtime control flow and require exhaustiveness with `never`. Avoid optional-property bags that permit impossible combinations.

Do not use truthiness to narrow when `0`, `''`, or `false` is a valid value. Prefer equality, `typeof`, `instanceof`, `Array.isArray`, `in`, or a discriminant.

### 8. Verify behavior and types

- Run the repository's narrowest relevant typecheck, lint, and test commands, then broader checks in proportion to the change.
- When changing the development loop, verify both a type error and a runtime edit: confirm the intended process restarts once, failed builds behave deliberately, source maps point to TypeScript, and shutdown releases ports and resources.
- Run a production-equivalent build and entry point in CI or before release. Do not let `tsx`, path aliases, or a development loader hide module-resolution or emit problems in the deployed runtime.
- With a `tsconfig.json` present, do not pass source file paths directly to TypeScript 7 unless intentionally using `--ignoreConfig`; prefer the project script or `tsc -p`.
- For public libraries, inspect emitted declarations or API reports. Ensure inferred exports did not expose private implementation details, giant anonymous types, or accidental literals.
- Add type-level regression checks for important inference: positive assignments/`satisfies` checks and focused `@ts-expect-error` negative cases with explanations.
- For augmentation, typecheck a consumer that imports the real public entry point, verify the augmentation file appears in `--explainFiles` or equivalent output, and test the runtime initialization path independently.
- Search the diff for new `any`, `unknown`, `as`, `!`, `@ts-ignore`, duplicated fields, broad index signatures, and explicit return types. Justify each exception.
- Do not weaken compiler or lint settings to make a local error disappear.

### 9. Report the result

Lead with the implemented behavior. Name the source of truth for derived types, any deliberate public annotations, every remaining escape hatch, and the verification commands run. If tooling prevented full verification, state the exact gap and its impact.
