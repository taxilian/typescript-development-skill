# TypeScript 7 Guide

Use this reference for TypeScript 7 installation, migration, compiler configuration, performance, and ecosystem compatibility. TypeScript 7 is fully released and production-ready; do not describe it as a preview. Version-sensitive statements below reflect TypeScript 7.0.2 and documentation available on 2026-08-04. Verify current primary documentation before changing dependencies or declaring a tool incompatible.

## Contents

1. [Establish the installed toolchain](#establish-the-installed-toolchain)
2. [Understand the TypeScript 7 contract](#understand-the-typescript-7-contract)
3. [Adopt TypeScript 7 defaults deliberately](#adopt-typescript-7-defaults-deliberately)
4. [Remove unsupported legacy configuration](#remove-unsupported-legacy-configuration)
5. [Account for compiler API and framework gaps](#account-for-compiler-api-and-framework-gaps)
6. [Run TypeScript 6 and 7 side by side](#run-typescript-6-and-7-side-by-side)
7. [Configure strict inference-first projects](#configure-strict-inference-first-projects)
8. [Migrate in a controlled sequence](#migrate-in-a-controlled-sequence)
9. [Use native parallelism after measuring](#use-native-parallelism-after-measuring)
10. [Handle TypeScript 7-specific inference behavior](#handle-typescript-7-specific-inference-behavior)
11. [Consult primary sources](#consult-primary-sources)

## Establish the installed toolchain

Before making a TypeScript 7-specific change:

1. Inspect `package.json`, lockfile, workspace manifests, and package-manager overrides/aliases.
2. Run the local package script or package-manager executable for `tsc --version`; do not trust a global compiler.
3. Resolve the full `tsconfig` extension chain with the compiler's `--showConfig` when useful.
4. Identify tools that import the compiler API: typed ESLint parsers, custom transformers, webpack loaders, API extractors, framework language plugins, test transformers, and code generators.
5. Identify editors or embedded languages that use a separate TypeScript language service.
6. Read current TypeScript 7 release notes and each integration's supported-version range.

Do not update a compiler major and unrelated framework/build dependencies in the same step unless the integration requires it.

## Understand the TypeScript 7 contract

TypeScript 7.0 is the fully released native Go implementation and is installed from the standard `typescript` package. Do not infer release status from stale native-port documentation. The former package named `@typescript/native-preview` was only the pre-release distribution channel; nightly builds return to `typescript@next`.

TypeScript 7.0 targets compatibility with TypeScript 6.0 type-checking and CLI behavior when TypeScript 6 uses `stableTypeOrdering` and does not suppress deprecations. TypeScript 7 also turns TypeScript 6 deprecations into hard errors.

TypeScript 7.0 does not expose the old programmatic compiler API. The TypeScript team planned a new API for TypeScript 7.1, so never assume packages importing `typescript` work merely because `tsc` works.

## Adopt TypeScript 7 defaults deliberately

TypeScript 7.0 adopts these notable defaults:

- `strict: true`;
- `module: "esnext"`;
- `target`: the stable ECMAScript version immediately before `esnext`;
- `noUncheckedSideEffectImports: true`;
- `libReplacement: false`;
- `stableTypeOrdering: true`, with no opt-out;
- `rootDir: "./"`;
- `types: []`.

Do not rely blindly on defaults in shared configs. State important intent explicitly when multiple compilers/tools consume the config.

The `rootDir` and `types` defaults commonly surface migration issues:

```json
{
  "compilerOptions": {
    "rootDir": "./src",
    "types": ["node", "jest"]
  },
  "include": ["./src"]
}
```

List only ambient `@types` packages needed in the global scope. Imported package declarations remain available even when omitted from `types`.

## Remove unsupported legacy configuration

TypeScript 7 rejects or no longer supports these TypeScript 6 deprecations:

- `target: "es5"` and `downlevelIteration`;
- `moduleResolution: "node"`, `"node10"`, or `"classic"`;
- `module: "amd"`, `"umd"`, `"system"`, or `"none"`;
- `baseUrl`;
- `esModuleInterop: false` and `allowSyntheticDefaultImports: false`;
- `alwaysStrict: false`;
- the `module` keyword for namespace declarations;
- import attributes written with `asserts` instead of `with`;
- some legacy triple-slash/default-lib behavior.

Prefer `moduleResolution: "nodenext"` for Node's runtime resolution or `"bundler"` for a bundler that implements its own resolution. Pair it with a compatible `module` setting based on the actual runtime/build pipeline. Do not choose between them as a style preference.

Convert `baseUrl`-dependent `paths` entries to project-root-relative paths. Confirm the runtime/bundler resolves the same aliases; TypeScript `paths` does not rewrite emitted imports.

When a directory contains a `tsconfig.json`, TypeScript 7 command-line builds cannot take file paths unless `--ignoreConfig` is explicitly passed. Prefer project scripts or `tsc -p path/to/tsconfig.json`.

## Account for compiler API and framework gaps

As of TypeScript 7.0:

- the compiler CLI and language server are production releases;
- the programmatic compiler API is absent;
- tools that import the compiler API may need TypeScript 6 side by side;
- Vue, MDX, Astro, Svelte, and similar embedded-language editor workflows may need TypeScript 6;
- Angular projects can use TypeScript 7 for CLI project-wide checking while retaining TypeScript 6 for specialized template/editor support;
- the current `typescript-eslint` parser package declares TypeScript support below 6.1, so use its documented supported compiler for typed linting until its range changes.

Check current compatibility rather than hard-coding these restrictions into project logic. If the editor and CLI use different compiler generations, document that fact and run both relevant checks in CI.

## Run TypeScript 6 and 7 side by side

The official transition package is `@typescript/typescript6`, which exposes `tsc6` and the TypeScript 6 API. The TypeScript team documents npm aliases for tools that require an import named `typescript`:

```json
{
  "devDependencies": {
    "@typescript/native": "npm:typescript@^7.0.2",
    "typescript": "npm:@typescript/typescript6@^6.0.2"
  }
}
```

In that arrangement, verify which executable each package script runs; package aliases and `.bin` links can be non-obvious. Name scripts clearly, for example `typecheck:ts7`, `typecheck:ts6`, and `lint`.

Do not introduce side-by-side compilers preemptively. Use them only for a concrete API or language-plugin dependency and remove the bridge when the ecosystem supports TypeScript 7's API.

## Configure strict inference-first projects

Start from the framework/runtime's supported module settings, then consider this correctness profile:

```json
{
  "compilerOptions": {
    "strict": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noPropertyAccessFromIndexSignature": true,
    "noUncheckedSideEffectImports": true,
    "forceConsistentCasingInFileNames": true,
    "verbatimModuleSyntax": true
  }
}
```

Adopt extra checks based on project behavior:

- `exactOptionalPropertyTypes` distinguishes absence from explicit `undefined`.
- `noUncheckedIndexedAccess` adds `undefined` to unproven indexed reads.
- `noPropertyAccessFromIndexSignature` makes uncertain dictionary access visually explicit.
- `noImplicitOverride` keeps class overrides linked to base members.
- `verbatimModuleSyntax` makes `import type`/`export type` behavior predictable, but must match the module pipeline.
- `isolatedModules` is important when another tool transpiles one file at a time.
- `isolatedDeclarations` is for declaration-producing pipelines; it intentionally requires sufficient export annotations and is a valid exception to inferred public returns.
- `erasableSyntaxOnly` is useful when runtimes strip types directly; it excludes TypeScript syntax with runtime transforms and is not a universal default.

Do not switch on a large flag bundle without fixing and testing the semantic effects. Never disable `strict` to complete a migration.

For linting, prefer type-aware strict presets when their parser supports the compiler version. Useful rules include:

- `no-explicit-any` and the `no-unsafe-*` family;
- `no-unnecessary-type-assertion` and `no-unsafe-type-assertion`;
- `prefer-as-const`;
- `switch-exhaustiveness-check`;
- `consistent-type-imports`;
- `no-floating-promises` and `no-misused-promises`;
- `no-unnecessary-type-parameters` and unnecessary type-argument checks where available.

Avoid a blanket `explicit-function-return-type` rule for inference-first application code. Configure exceptions for public APIs if that is the project's chosen contract policy.

## Migrate in a controlled sequence

1. Make the project clean on TypeScript 6.0 with `stableTypeOrdering` and without `ignoreDeprecations`.
2. Inventory compiler-API consumers and embedded-language plugins.
3. Replace unsupported targets, modules, resolution modes, `baseUrl`, and legacy import attributes.
4. Set `rootDir` and ambient `types` explicitly where needed.
5. Install TypeScript 7 using the existing package manager and lockfile policy.
6. Run TypeScript 7's project typecheck and build.
7. Run TypeScript 6-dependent lint/editor/framework checks if a compatibility bridge is required.
8. Inspect emitted JavaScript and declarations for libraries.
9. Run unit, integration, and package-consumer tests.
10. Document temporary dual-compiler tooling and an exit condition.

Do not respond to a changed diagnostic by casting. Determine whether it reveals a bug, a changed default, stable type ordering, or an ecosystem mismatch.

## Use native parallelism after measuring

TypeScript 7 parallelizes parsing, checking, emitting, and project-reference builds. It provides experimental tuning flags:

- `--checkers N` controls type-checker workers; the default is 4 in TypeScript 7.0.
- `--builders N` controls parallel project-reference builders.
- `--singleThreaded` disables parallelism for constrained environments or debugging.

More workers can reduce time while increasing memory. CI may need fewer workers than developer machines. Benchmark representative clean/incremental builds before pinning values, and keep a fixed value across environments if order-dependent diagnostics appear.

Continue to write easy-to-compile types: prefer interface extension, name complex types, avoid huge union intersections, and use explicit exported return types at measured hot spots. A faster compiler does not make pathological type-level programs maintainable.

## Handle TypeScript 7-specific inference behavior

TypeScript 7 template-literal inference splits Unicode code points rather than UTF-16 surrogate halves:

```ts
type HeadTail<S> = S extends `${infer Head}${infer Tail}` ? [Head, Tail] : never;

type Result = HeadTail<'😀abc'>;
// TypeScript 7: ['😀', 'abc']
```

Audit type-level string length, splitting, parser, and case utilities that intentionally modeled UTF-16 code units. Add non-BMP Unicode tests when the distinction matters.

TypeScript 7 also aligns JavaScript/JSDoc analysis more closely with TypeScript. In checked JavaScript, use `typeof value` in type positions and current TypeScript-style JSDoc forms; do not rely on removed Closure-specific patterns.

## Consult primary sources

- [Announcing TypeScript 7.0](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/)
- [TypeScript 7 native compiler repository](https://github.com/microsoft/typescript-go)
- [TypeScript compiler performance guidance](https://github.com/microsoft/TypeScript/wiki/Performance)
- [TypeScript 6.0 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-6-0.html)
- [TSConfig reference](https://www.typescriptlang.org/tsconfig/)
- [typescript-eslint typed linting](https://typescript-eslint.io/getting-started/typed-linting/)
- [typescript-eslint compiler peer dependency](https://github.com/typescript-eslint/typescript-eslint/blob/main/packages/typescript-estree/package.json)

Use current official documentation for every version/tooling claim. Prefer Context7's official TypeScript and typescript-eslint sources where available; use official TypeScript websites and repositories as the fallback.
