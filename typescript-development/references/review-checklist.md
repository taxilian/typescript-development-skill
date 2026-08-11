# TypeScript Review Checklist

Use this checklist before finishing a material implementation, refactor, migration, or code review. Apply only relevant items, but do not skip boundary safety or verification.

## Contents

1. [Priority gate](#priority-gate)
2. [Project context](#project-context)
3. [Inference and contracts](#inference-and-contracts)
4. [Objects and configuration](#objects-and-configuration)
5. [DRY type composition](#dry-type-composition)
6. [Generics and advanced types](#generics-and-advanced-types)
7. [Boundary safety](#boundary-safety)
8. [State, control flow, and mutability](#state-control-flow-and-mutability)
9. [TypeScript 7 configuration and tooling](#typescript-7-configuration-and-tooling)
10. [Augmentation and ambient declarations](#augmentation-and-ambient-declarations)
11. [Development loop](#development-loop)
12. [Compiler health and readability](#compiler-health-and-readability)
13. [Verification](#verification)
14. [Escape-hatch ledger](#escape-hatch-ledger)

## Priority gate

- [ ] Types match values and behavior that can occur at runtime; external data is validated before trust.
- [ ] Local values and implementation return types are inferred by default; every explicit annotation communicates a deliberate contract or documented constraint.
- [ ] The design minimizes total complexity across the abstraction and its consumers, not merely the syntax of the type definition.
- [ ] Any sophisticated type machinery is localized, clearly named, concisely documented, and protected by focused positive and negative type tests.
- [ ] Common call sites work intuitively without unnecessary annotations, assertions, duplicated logic, or knowledge of the internal type machinery.
- [ ] Linked facts have one authoritative source; derivations are named or simplified when reuse would otherwise become opaque.
- [ ] New finite states, variants, keys, and schema fields fail at every branch or lookup that must be updated.
- [ ] Inference preserves useful literals and relationships without hiding a deliberate public contract.
- [ ] Compiler-performance or declaration-size changes are supported by measurements.

## Project context

- [ ] Read repository instructions and adjacent code.
- [ ] Confirm the local TypeScript version and complete `tsconfig` chain.
- [ ] Identify application versus internal/published package boundaries.
- [ ] Preserve the package manager, framework conventions, module system, and existing scripts.
- [ ] Check current TypeScript 7 compatibility for compiler-API consumers and embedded-language tools.

## Inference and contracts

- [ ] Infer local function returns and local values.
- [ ] Give each explicit return annotation a concrete reason: public contract, recursion, intentional abstraction, `isolatedDeclarations`, overload/assertion signature, or measured performance.
- [ ] Explicitly annotate a shared failure or exhaustiveness helper with `never`; verify that every path truly cannot complete normally.
- [ ] Use contextual typing for callbacks rather than repeating parameter types.
- [ ] Prevent inferred public declarations from exposing private names, accidental literals, or giant anonymous structures.
- [ ] Prefer `ReturnType`, `Parameters`, and `Awaited` when a function is the source of truth.
- [ ] Prefer `typeof`, `keyof typeof`, and indexed access when a value/property is the source of truth.

## Objects and configuration

- [ ] Use `satisfies` instead of a broad object annotation when inferred detail matters.
- [ ] Use `as const` only when literal capture and readonly intent are correct.
- [ ] Define config contracts with readonly properties/collections when configs are immutable.
- [ ] Remember that `satisfies` and `as const` provide no runtime validation or freezing.
- [ ] Check excess keys and missing keys at the literal construction site.
- [ ] Annotate empty mutable collections that have no inference evidence; do not expect `satisfies` to prevent `never[]`.
- [ ] Preserve schema/definition literals through the generic call; avoid broad annotations and non-generic intermediate helpers.

## DRY type composition

- [ ] Search for an existing domain type before adding one.
- [ ] Extend the parent interface for a real object hierarchy.
- [ ] Use `Pick`, `Omit`, `Partial`, `Required`, or indexed access instead of copying linked fields.
- [ ] Keep coincidentally same-shaped fields independent when they may evolve separately.
- [ ] Replace hard-to-read utility stacks with a small named interface/type.
- [ ] Check utility behavior over unions; distribute only when intended.
- [ ] Avoid generic deep transforms unless special runtime types have defined semantics.

## Generics and advanced types

- [ ] Make every type parameter relate meaningful positions; remove decorative parameters.
- [ ] Push type parameters down and use the fewest needed.
- [ ] Use `const` type parameters only when callers need literal preservation.
- [ ] Use `NoInfer` only to select an authoritative inference source.
- [ ] Preserve argument lists with variadic tuples instead of `Function` or `any[]`.
- [ ] Keep correlated keys, payloads, and callbacks in a discriminated or mapped-indexed union.
- [ ] Make conditional distribution explicit.
- [ ] Name and test complex conditional, mapped, template-literal, or recursive types.
- [ ] Use `Exclude`/`Extract` before writing a custom `never`-based union filter.
- [ ] Use key remapping to `never` only when the runtime API really filters those keys.
- [ ] Tuple-wrap an `IsNever`-style check so distributive conditional behavior cannot erase the test.
- [ ] Use `(...args: never[]) => unknown` only to inspect or constrain a callable that will never be invoked; use variadic tuples when calling or wrapping it.
- [ ] Prefer a direct union/interface/overload when it communicates better.
- [ ] For definition-driven APIs, capture the whole definition, interpret it with mapped/conditional types, and thread it into the returned model or instance.
- [ ] Use the library's supported `Infer...` helper and distinguish raw data from hydrated/wrapped values where applicable.

## Boundary safety

- [ ] Assign untrusted raw data to `unknown` and narrow immediately.
- [ ] Use the project's existing runtime schema/validator when available.
- [ ] Make handwritten guards prove every required field and test true/false cases.
- [ ] Treat a predicate claiming `value is never` as a likely type lie; normal guards should narrow an impossible remainder to `never`.
- [ ] Avoid truthiness narrowing when valid values can be `0`, `''`, or `false`.
- [ ] Treat caught errors as `unknown` until narrowed.
- [ ] Prevent `any` from leaking out of external adapters.
- [ ] Treat assertions, brands, and non-null assertions as documented proof obligations.
- [ ] Reject double assertions and `@ts-ignore`.
- [ ] Use `@ts-expect-error` only for focused negative tests or documented external defects.

## State, control flow, and mutability

- [ ] Use discriminated unions for mutually exclusive states.
- [ ] Narrow a known union inline with its discriminant or other ordinary control flow before creating a one-use guard.
- [ ] Use a named guard only for a reused semantic rule or multi-step boundary validation.
- [ ] Require exhaustive `switch` statements and `if` chains with one project-standard helper that accepts and returns `never`.
- [ ] Keep a runtime throw in the exhaustive helper when unvalidated or version-skewed values can arrive.
- [ ] Prefer a complete `satisfies Record<Keys, Value>` lookup when the operation is static data mapping.
- [ ] Do not let a broad `default` silently absorb future members of a closed union or enum.
- [ ] Prefer a discriminant over optional `never` properties; use the exclusive-property pattern only when the runtime shape cannot carry a tag.
- [ ] Accept readonly inputs when mutation is unnecessary.
- [ ] Distinguish optional absence from explicit `undefined`, especially with `exactOptionalPropertyTypes`.
- [ ] Avoid optional-property bags that admit impossible states.
- [ ] Verify handler parameter variance where narrower callbacks could be unsafe.

## TypeScript 7 configuration and tooling

- [ ] Use `nodenext` versus `bundler` according to the actual runtime/build resolver.
- [ ] Remove unsupported legacy target/module/resolution options and `baseUrl` assumptions.
- [ ] Set `rootDir` and ambient `types` intentionally.
- [ ] Keep `strict` enabled; consider `exactOptionalPropertyTypes` and `noUncheckedIndexedAccess`.
- [ ] If using typescript-eslint switch exhaustiveness, use typed linting and keep `considerDefaultExhaustiveForUnions: false` when every new member must receive an explicit case.
- [ ] Reject redundant default cases only when runtime values cannot exceed the compile-time union; otherwise retain a throwing default deliberately.
- [ ] Use `verbatimModuleSyntax`, `isolatedModules`, `isolatedDeclarations`, and `erasableSyntaxOnly` only when the build model calls for them.
- [ ] Do not pass source paths to `tsc` beside a `tsconfig` unless intentionally using `--ignoreConfig`.
- [ ] Keep temporary TypeScript 6/7 dual-tooling isolated, documented, and removable.
- [ ] Tune `--checkers`/`--builders` only from measurements.

## Augmentation and ambient declarations

- [ ] Confirm the augmented property or method exists at runtime and identify the code that installs it.
- [ ] Prefer a generic, refined local type, or wrapper when the extension is not universal.
- [ ] Target the dependency's documented open interface and exact public module specifier.
- [ ] Distinguish module augmentation from a script-file ambient module declaration.
- [ ] Make global augmentations module-scoped with `export {}` or a deliberate top-level import.
- [ ] Use `var` only for values intentionally exposed on `globalThis`; do not expect top-level Node module variables to become global.
- [ ] Keep augmented members optional unless every represented runtime value is initialized before observation.
- [ ] Avoid augmenting type aliases, default exports, private names, or undeclared top-level module exports.
- [ ] For Express, prefer the open global `Express` interfaces for universal additions and handler/response generics for route-local data.
- [ ] Treat `NodeJS.ProcessEnv` augmentation as autocomplete, not validation; consume a validated config object in application code.
- [ ] Confirm the augmentation `.d.ts` is included and that `types`, `typeRoots`, `moduleDetection`, and package subpaths do not hide or change it.
- [ ] Typecheck declarations with `skipLibCheck: false` and test the runtime initialization, missing, and reordered paths.

## Development loop

- [ ] Separate type-checking, emission, execution, and restart responsibilities.
- [ ] Give process restarts to exactly one watcher or supervisor.
- [ ] Pair `tsx` with a mandatory `tsc --noEmit` check in development or CI; do not describe `tsx` as a type-checker.
- [ ] Use `tsc-watch --onSuccess` rather than `--onEmit` when only clean compilations may restart the process.
- [ ] Use project-local compiler and runner versions rather than global installations.
- [ ] Exclude emitted, generated, dependency, cache, and coverage paths from watches as appropriate.
- [ ] Prevent emitted writes from creating build or restart loops.
- [ ] Enable source maps and graceful `SIGTERM` handling for a restarted Node process.
- [ ] Verify that the development runner does not hide production module-resolution, path-alias, ESM/CJS, or emit failures.
- [ ] Test a type failure, one valid restart, rapid edits, and watcher shutdown.

## Compiler health and readability

- [ ] Evaluate an advanced type by its downstream benefit: simpler call sites, stronger inference, less repetition, or a valuable enforced invariant.
- [ ] Prefer interfaces/extends over named object intersections.
- [ ] Name repeated complex types for caching and diagnostics.
- [ ] Avoid huge unions, union intersections, unbounded recursion, and type-level parsers without a demonstrated need.
- [ ] Do not trade readability for hypothetical compiler speed; measure first.
- [ ] Ensure error messages remain understandable to callers.
- [ ] Do not replace localized advanced machinery with a simpler-looking definition that pushes complexity into every consumer.

## Verification

- [ ] Run the narrowest relevant project typecheck, then the broader typecheck as appropriate.
- [ ] Run compatible typed lint rules, tests, and the relevant build.
- [ ] Inspect declaration output/API reports for published packages.
- [ ] Add positive inference assertions and focused negative `@ts-expect-error` cases.
- [ ] Temporarily add a union member or lookup key and confirm every required exhaustive branch/table fails to typecheck.
- [ ] Verify non-returning helpers narrow following expressions and still throw or halt at runtime.
- [ ] Test runtime validation independently from compile-time types.
- [ ] Review the final diff for behavior changes and unrelated config churn.

## Escape-hatch ledger

Search the final diff for each item and record the reason for every new occurrence:

- [ ] explicit function return annotation;
- [ ] `never` return outside a failure/exhaustiveness contract;
- [ ] `as never` or an explicit `value is never` predicate;
- [ ] `unknown`;
- [ ] `any`;
- [ ] `as` assertion;
- [ ] non-null `!`;
- [ ] `@ts-expect-error`;
- [ ] broad string/number index signature;
- [ ] duplicated domain field;
- [ ] disabled compiler/lint rule.

If the reason cannot be stated in one precise sentence, redesign the code.
