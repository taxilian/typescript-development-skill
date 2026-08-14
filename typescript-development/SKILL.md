---
name: typescript-development
description: Develop, refactor, review, debug, document, configure, run, and migrate TypeScript 7 code with runtime-faithful, readable, inference-first, strict, and DRY type design. Use for .ts, .tsx, .mts, .cts, .d.ts, tsconfig files, TypeScript package APIs, TSDoc or JSDoc, TypeDoc, public API documentation, implementation comments, descriptive naming, type-heavy application or library code, advanced generics, schema-driven or definition-driven inference, control-flow narrowing, discriminated unions, never and exhaustiveness, declaration merging, global or module augmentation, ambient modules, globalThis, Node and Express extension points, inferred configuration, repairing type mismatches, safe nullish defaults, controlling assertions and escape hatches, eliminating any or unnecessary unknown, shared domain models, TypeScript 6-to-7 migration, compiler and type errors, or development loops involving tsc watch mode, tsc-watch, tsx, process restarts, and package scripts.
---

# TypeScript Development

Build code whose runtime behavior and domain relationships the compiler describes truthfully and whose types work intuitively at the point of use. Spend type-level complexity deliberately when it removes greater complexity from application code, strengthens inference, or catches real mistakes.

## Apply priorities in order

Use this order to resolve tradeoffs:

1. **Runtime truth and domain correctness.** Model values the program can actually receive, validate untrusted inputs, and never let an assertion or predicate make an unchecked mismatch escape its deliberate containment boundary.
2. **Clarity and intuitive use.** Optimize the whole abstraction, especially its call sites. Accept sophisticated internal types when they make consuming code simpler, safer, and more natural; localize them behind clear names, concise documentation, focused type tests, and actionable diagnostics.
3. **One source of truth and reuse.** Derive linked values and types instead of copying them, but name or simplify a derivation when readers would otherwise have to decode it repeatedly.
4. **Refactor safety.** Make new union members, keys, states, and schema fields produce useful errors at every decision or lookup that must change.
5. **Measured compiler and tooling health.** Keep diagnostics actionable and optimize type-checking or declaration size only from evidence.

Treat inference-first as an operating default across every priority, not as a lower-ranked objective. Infer local values and implementation return types unless an annotation communicates a deliberate contract or solves a specific inference, declaration, or measured performance constraint. Preserve literal and relational information rather than restating or widening it.

Judge these priorities across the abstraction and all of its consumers, not one type expression in isolation. A simple type that misrepresents runtime data is still wrong. A complex type can be the clearest design when it concentrates complexity once and lets many call sites work naturally without annotations, assertions, duplicated relationships, or defensive branches. Do not add complexity that merely moves work around or produces harder usage and diagnostics.

## Load the right reference

- Read [type-system-playbook.md](references/type-system-playbook.md) when designing or refactoring configs, exported types, generics, guards, discriminated unions, exhaustive decisions, deliberate `never` usage, conditional or mapped types, assertions, or `any`/`unknown` usage.
- Read [code-documentation.md](references/code-documentation.md) when adding or reviewing TSDoc/JSDoc, public API documentation, interface or property descriptions, implementation comments, phase comments, TODOs, examples, or abbreviated names.
- Read [type-augmentation.md](references/type-augmentation.md) when typing existing globals, adding properties supplied by middleware or plugins, extending third-party declarations, declaring untyped modules or assets, or designing an augmentable library API.
- Read [typescript-7.md](references/typescript-7.md) when installing, configuring, migrating, compiling, linting, integrating editor/build tooling, or relying on TypeScript 7 behavior. Re-verify its time-sensitive compatibility notes against current primary documentation before changing dependencies.
- Read [development-loop.md](references/development-loop.md) when choosing or changing watch mode, source execution, emitted execution, process restart hooks, source maps, or `package.json` development scripts.
- Read [review-checklist.md](references/review-checklist.md) before finishing a material TypeScript implementation, migration, or review.

## Apply the working contract

1. Make every type match the values and behavior that can occur at runtime. TypeScript types are erased; validate external data.
2. Infer local values and function returns by default.
3. Annotate only where the annotation expresses a real contract, breaks a cycle, enables an intentional abstraction or assertion signature, marks a deliberately non-returning function, satisfies `isolatedDeclarations`, or fixes a measured compiler/declaration performance problem.
4. Minimize total complexity across definitions and consumers. Accept localized, named, documented, and tested type machinery when it makes common usage intuitive and meaningfully simpler.
5. Keep one authoritative definition. Derive related types with `typeof`, `keyof`, indexed access, `ReturnType`, `Parameters`, `Awaited`, `Pick`, `Omit`, or interface extension.
6. Validate object literals with `satisfies`; do not erase their useful inferred detail with a broad annotation.
7. Use `as const` for configuration and lookup data when literal preservation and readonly intent are correct. Do not use it as decoration or claim runtime immutability.
8. Keep `unknown` at genuinely untrusted or universal boundaries and narrow it immediately. Use `any` rarely, as the smallest clear suspension of checking for one understood relationship; resume typed code immediately and prevent it from leaking through declarations, inferred returns, generics, overloads, callbacks, or stored values.
9. Treat type assertions, explicit predicates, assertion signatures, and non-null assertions as proof obligations. Repair the rejected type relationship before considering an escape hatch. Apply the same justification threshold to `as any` and `as unknown as T`, then use the shortest form that preserves useful downstream types. Make the runtime invariant or deliberately limited test substitute evident in context and comment only when it is not obvious.
10. Prefer interfaces and `extends` for named object hierarchies. Prefer type aliases for unions, tuples, primitives, mapped types, conditional types, and other transformations.
11. Model finite states with discriminated unions and make decisions over them exhaustive. Treat an unexpected `never` as a diagnostic to investigate, not a type to cast away.
12. Document public contracts with TSDoc and implementation intent with `//` comments. Add semantic information rather than restating names and types; rename or refactor before compensating with narration.
13. Separate type-checking, emission, execution, and restart responsibilities. Give process restarts to exactly one tool, and never mistake fast transpilation for type-checking.
14. Treat augmentation as a runtime proof obligation. Target the exact existing declaration, ensure the declaration file is loaded, and create or validate the augmented member before use.

Use this escalation order when the compiler needs help:

`correct source type` → `inference and narrowing` → `satisfies or annotation` → `typed helper or adapter` → localized `assertion` → isolated `any`

## Follow the agentic workflow

### 1. Discover the actual project

- Read repository instructions and inspect `package.json`, the full `tsconfig` extension chain, lockfile, source layout, framework/build tooling, lint config, and existing tests.
- Run the local compiler version command or package-manager equivalent. Do not assume the project is on TypeScript 7 because the request mentions it.
- Determine whether the code is an application, an internal package, or a published library. Public API and declaration requirements change annotation decisions.
- Identify existing typecheck, lint, test, and build scripts. Preserve the project's package manager and conventions.
- Identify the production runtime and the current owner of type-checking, emission, source execution, and process restarts. Avoid adding a second watcher for the same responsibility.
- Inspect existing `.d.ts` files and dependency declarations before augmenting a global or module. Find the documented open interface and the exact module specifier that owns it.
- Search existing project utilities and dependency APIs before adding a fallback, coercion, normalization, or assertion helper. Reuse the established runtime behavior and naming when it is sound.
- Inspect existing TSDoc/JSDoc style, generated-documentation tooling, lint rules, declaration output, and documentation coverage expectations before adding comments or tags.
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
- which declarations form a public or subsystem contract, what their consumers cannot learn from the signature alone, and which comments are implementation-only.

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
| Nullable value needs a valid default | Direct `??` with a genuinely assignable fallback | The pattern repeats or an exact subtype needs a boundary-specific factory/helper |
| Compiler rejects a proposed conversion | Correct the source type, narrow, validate, or preserve the lost generic relationship | A proven runtime invariant or deliberately limited test/implementation boundary makes a localized escape clearer without weakening surrounding types |
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

Permit `any` only when it disables one understood check rather than an area of the program. Pin the result immediately with a trustworthy annotation, non-generic parameter, explicit return contract, or equally clear typed boundary. Confirm that it cannot alter an inferred return, generic argument, overload choice, callback, stored property, or emitted declaration. Require a proven runtime invariant, a staged construction guarantee completed before escape, or a deliberately partial/invalid test substitute whose mismatch is irrelevant to the behavior under test. Prefer a precise model when it recovers meaningful checking, but do not add elaborate types or a longer double assertion that changes no useful type behavior. Keep `no-explicit-any` and the `no-unsafe-*` lint family enabled where compatible, with narrow reviewed exceptions.

When the compiler rejects a conversion, read the diagnostic as evidence of a broken or missing relationship. Inspect the source declaration, lost generic, mutability, tuple or branded subtype, library wrapper, and nullability before writing `as`. Prefer direct `??` for a valid nullish default. If the pattern repeats, use an existing helper or add a typed helper whose fallback is actually assignable to its result; never hide `[] as T` or `{} as T` inside a generic utility.

Permit `null!` only as a deliberately documented way to emit a real `null` sentinel where the immediate consumer's verified runtime contract accepts it but the destination type intentionally cannot represent that construction-only value or has an inaccurate declaration. It is not null-safety: under strict checking the expression is assignable because it becomes `never`, while the JavaScript value remains `null`. Keep it on the exact expression, explain the mismatch, and cover the runtime path. Do not let the sentinel escape to code promised a non-null value; repair the type, declaration, or boundary when the exception repeats.

### 6. Design generics for inference

- Make each type parameter relate at least two meaningful positions. Remove decorative generics.
- Push type parameters down to the value shape; constrain only capabilities actually used.
- Use the fewest type parameters that preserve the relationship.
- Accept `readonly` collections when the function does not mutate them.
- Use `const` type parameters in library/builders APIs that should capture caller literals without requiring `as const` at every call.
- For schema, route, command, and similar definition APIs, capture the whole definition in one generic and thread it into the returned model/instance type. Do not widen it in an intermediate variable or non-generic helper.
- Use `NoInfer<T>` to stop a fallback/default argument from widening the type inferred from the authoritative argument.
- Make conditional-type distribution intentional; wrap both sides in tuples when the union must be treated as a whole.
- Name, document, and test complex mapped, recursive, or conditional types. Keep them when they improve inference, remove repeated consumer logic, or enforce a valuable invariant; choose a direct interface, discriminated union, or overload when the advanced form does not provide enough downstream benefit.

### 7. Model, narrow, and exhaust states

Prefer discriminated unions for mutually exclusive states. Avoid optional-property bags that permit impossible combinations.

Narrow a known union with ordinary JavaScript control flow first: a discriminant comparison, equality, `typeof`, `instanceof`, `Array.isArray`, or `in`. TypeScript follows `if`/`else`, early returns, assignments, and `switch`; do not extract a one-use type guard merely to restate a check the compiler already understands. Use a named predicate when the runtime test is complex or reused, and remember that an explicit predicate promises both its true and false branches.

For every finite decision, handle each member explicitly and terminate the impossible remainder with a project-standard function accepting `never` and returning `never`. Its explicit return type is justified because it declares a control-flow contract and preserves a runtime error for unvalidated or version-skewed values. Do not use a permissive `default` that silently handles future members.

Prefer a complete `Record` literal checked with `satisfies` when the operation is a static mapping rather than behavior. Adding a union member should fail at the lookup definition just as it should fail at an exhaustive branch.

Do not use truthiness to narrow when `0`, `''`, or `false` is a valid value. Prefer equality, `typeof`, `instanceof`, `Array.isArray`, `in`, or a discriminant.

### 8. Document contracts and implementation intent

- Give published/public top-level exports TSDoc summaries. Document internal declarations when they carry a non-obvious contract, invariant, algorithm, or workaround; do not force placeholder prose onto trivial local helpers.
- Use summaries and tags to explain semantic behavior: constraints, units, formats, defaults, ownership, mutation, ordering, side effects, stable failure conditions, and generic relationships. Omit `@param` or `@returns` when it would only repeat the name and TypeScript signature.
- Use `//` comments for implementation rationale and meaningful algorithmic phases. If a long function needs narration for many small blocks, first extract responsibilities, name intermediate concepts, flatten control flow, or strengthen its types.
- Rename unfamiliar shortened variables instead of adding comments to decode them. Permit conventional or tiny-scope names; document forced external abbreviations at the nearest reusable interface or protocol boundary.
- Document exported interfaces/types and every property whose units, optional semantics, default, lifecycle, sensitivity, or cross-field relationship is not obvious. Extract a nested interface for a true field group; use restrained `//` headings only when a required flat shape must remain flat.

### 9. Verify behavior, types, and documentation

- Run the repository's narrowest relevant typecheck, lint, and test commands, then broader checks in proportion to the change.
- When changing the development loop, verify both a type error and a runtime edit: confirm the intended process restarts once, failed builds behave deliberately, source maps point to TypeScript, and shutdown releases ports and resources.
- Run a production-equivalent build and entry point in CI or before release. Do not let `tsx`, path aliases, or a development loader hide module-resolution or emit problems in the deployed runtime.
- With a `tsconfig.json` present, do not pass source file paths directly to TypeScript 7 unless intentionally using `--ignoreConfig`; prefer the project script or `tsc -p`.
- For public libraries, inspect emitted declarations or API reports. Ensure inferred exports did not expose private implementation details, giant anonymous types, or accidental literals.
- Add type-level regression checks for important inference: positive assignments/`satisfies` checks and focused `@ts-expect-error` negative cases with explanations.
- For a changed finite union, enum, event map, or key set, verify that adding a temporary member fails every exhaustive branch and complete lookup that requires an update.
- Use a type-aware switch-exhaustiveness lint rule as a supplement when the project already supports typed linting; configure it so a generic `default` does not hide newly added union members.
- For augmentation, typecheck a consumer that imports the real public entry point, verify the augmentation file appears in `--explainFiles` or equivalent output, and test the runtime initialization path independently.
- When documentation tooling exists, validate TSDoc syntax, links, intended public coverage, and generated output. Keep code examples typechecked or executable when practical.
- Review changed comments beside changed behavior. Remove stale narration and verify defaults, units, optional semantics, side effects, errors, deprecations, and TODO references.
- Search the diff for new `any`, `unknown`, `as`, `!`, `@ts-ignore`, duplicated fields, broad index signatures, explicit return types, and unusual `never` uses. Justify each exception.
- Search specifically for `as unknown as`, `as any`, `null!`, and generic helpers that return an asserted `T`. Verify typed containment and a locally evident reason; add comments and focused type/runtime coverage in proportion to the non-obvious invariant and its blast radius.
- Do not weaken compiler or lint settings to make a local error disappear.

### 10. Report the result

Lead with the implemented behavior. Name the source of truth for derived types, any deliberate public annotations, every remaining escape hatch, and the verification commands run. If tooling prevented full verification, state the exact gap and its impact.
