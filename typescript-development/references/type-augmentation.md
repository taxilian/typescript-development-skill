# Type Augmentation and Ambient Declarations

Use augmentation only to describe a runtime contract that already exists or is installed by known initialization code. Declaration merging changes the compiler's view; it creates no property, executes no plugin, validates no value, and enforces no middleware order. Express details were verified against `@types/express` 5.0.6 and `@types/express-serve-static-core` 5.1.3 on 2026-08-05; re-check the installed declarations when versions change.

## Contents

1. [Choose augmentation only when the scope is real](#choose-augmentation-only-when-the-scope-is-real)
2. [Distinguish the declaration mechanisms](#distinguish-the-declaration-mechanisms)
3. [Keep runtime behavior and requiredness honest](#keep-runtime-behavior-and-requiredness-honest)
4. [Augment the global scope](#augment-the-global-scope)
5. [Augment an existing module](#augment-an-existing-module)
6. [Declare a missing module instead of augmenting one](#declare-a-missing-module-instead-of-augmenting-one)
7. [Extend Express through its intended contracts](#extend-express-through-its-intended-contracts)
8. [Treat Node globals and environment variables carefully](#treat-node-globals-and-environment-variables-carefully)
9. [Design deliberate extension points for library consumers](#design-deliberate-extension-points-for-library-consumers)
10. [Place and load declaration files intentionally](#place-and-load-declaration-files-intentionally)
11. [Verify types and runtime behavior separately](#verify-types-and-runtime-behavior-separately)
12. [Consult primary sources](#consult-primary-sources)

## Choose augmentation only when the scope is real

Start with the narrowest mechanism that matches runtime behavior:

| Runtime situation | Prefer |
| --- | --- |
| A property exists on every instance after a documented plugin or bootstrap step | Augment the documented open interface and test the runtime step |
| A property exists only in one route, middleware branch, or subsystem | A framework generic, local intersection, refined type, or wrapper |
| External input may or may not contain a property | Runtime validation followed by an inferred domain type |
| A module exists but has no declarations | A complete ambient module declaration |
| A module has declarations but the desired target is a type alias, default export, or private name | A wrapper/adapter or an upstream type fix |
| A library intentionally supports third-party plugins | A named exported interface or registry designed for module augmentation |

Do not augment merely to avoid narrowing, to make an optional value required, or to silence a dependency error. Prefer an upstream correction when the published declaration is objectively wrong and the fix benefits every consumer.

## Distinguish the declaration mechanisms

Use the right mechanism; several forms share similar syntax but have different meanings.

- **Interface merging** combines same-named interfaces in the same declaration scope. Duplicate data properties must have identical types. Duplicate methods become overloads, whose ordering can change resolution.
- **Global augmentation** uses `declare global` inside an external module to merge declarations into global scope.
- **Module augmentation** uses `declare module 'specifier'` inside an external module to patch existing named declarations resolved from that exact specifier.
- **Ambient module declaration** uses `declare module 'specifier'` in a script `.d.ts` file to describe a module that otherwise has no declarations.
- **Namespace merging** models JavaScript values with attached members, but is rarely the right consumer-side extension mechanism.

A file becomes an external module when it has a top-level import/export or when `moduleDetection` classifies it as one. Therefore, this sentinel is semantic, not decorative:

```ts
export {};
```

In a module file, `declare module 'pkg'` is augmentation. In a script file, the same syntax is an ambient declaration and can replace the compiler's understanding of the package instead of extending it.

Module augmentation has hard limits:

- patch an existing named, exported, mergeable declaration;
- do not add a new top-level export;
- do not augment a default export by the name `default`;
- do not reach a non-exported declaration;
- do not expect a type alias to merge like an interface;
- use the exact module specifier or package subpath resolved by consumers.

If the target does not meet those rules, introduce a wrapper or propose a named extension interface upstream.

## Keep runtime behavior and requiredness honest

For every augmented member, identify:

1. Which runtime code creates it?
2. Which instances receive it?
3. Can code observe the object before initialization?
4. Can initialization fail or be skipped in tests, workers, or alternate entry points?
5. Does importing a plugin execute the required side effect, or is explicit registration required?

Declare a member as required only if every value represented by the augmented type has it before any typed consumer can observe it. Otherwise, make it optional and narrow it with a guard or assertion function at the point where runtime control flow proves presence.

Keep runtime patches and their declarations close together when practical. A `.ts` plugin module can contain both `declare module` and the prototype/registration code. A `.d.ts` import loads declarations only; it does not execute the package at runtime. Likewise, `import type` never activates a side-effect plugin.

Avoid modifying built-in prototypes and broad globals. When an existing dependency genuinely does so, mirror its runtime behavior precisely and test collision, load-order, and missing-import cases.

## Augment the global scope

Put project global declarations in an included module-shaped declaration file:

```ts
import type { AppServices } from '../app-services.js';

export {};

declare global {
  var appServices: AppServices | undefined;
}
```

Then perform the runtime initialization explicitly:

```ts
globalThis.appServices ??= createAppServices();
```

Use ambient `var` for a value that is a property of `globalThis`. TypeScript models global `var` declarations on `globalThis`; global `let` and `const` declarations do not become its properties. The declaration emits nothing.

In Node, top-level `var` inside either a CommonJS or ECMAScript module remains module-local. Assign to `globalThis` when the runtime state is intentionally global. Prefer `globalThis` to Node's legacy `global` alias.

Augment `Window`, `WorkerGlobalScope`, or another environment interface only when that environment is the real contract. Do not add browser globals to a universal or Node-only project merely because DOM libraries happen to be included.

Choose collision-resistant names and keep mutable global state minimal. Remember that workers and isolated test realms do not necessarily share the same global object or initialization lifecycle.

## Augment an existing module

Import the original module, then merge an existing named interface:

```ts
import 'request-context';

declare module 'request-context' {
  interface Context {
    traceId: string;
  }
}
```

This is correct only if `request-context` already exports a mergeable `Context` and runtime code supplies `traceId` on every represented context. The import makes the file a module and ensures the original declarations participate in resolution; in a `.d.ts` file, it does not run JavaScript.

Target the public extension point documented by the dependency. A facade package, internal implementation package, and exported subpath can resolve to different declarations. Augmenting `'pkg'` does not necessarily affect `'pkg/subpath'`, and augmenting a re-exporting facade may not reach the interface used internally.

Keep third-party patches version-sensitive and small. Recheck them on dependency upgrades because an upstream property with an incompatible type will turn the merge into an error. Do not hide such conflicts with `skipLibCheck`.

## Declare a missing module instead of augmenting one

Use an ambient module declaration when the runtime or loader accepts an import for which no declaration exists:

```ts
declare module '*.svg' {
  const url: string;
  export default url;
}
```

Keep the containing `.d.ts` file a script: no top-level import or export. Imports needed only for types may appear inside the `declare module` block without converting the containing file into a module.

For an untyped JavaScript package, describe its actual exports precisely. Avoid the empty shorthand `declare module 'pkg';`, which makes imports effectively `any` and can conceal a package that already ships usable declarations. Mirror package subpaths when consumers import them separately.

## Extend Express through its intended contracts

The current Express core declarations deliberately expose open global interfaces named `Express.Request`, `Express.Response`, `Express.Locals`, and `Express.Application`. The exported core `Request` and `Response` interfaces extend those global interfaces, so application-wide properties should normally use that documented extension point:

```ts
import type { User } from '../auth/user.js';

export {};

declare global {
  namespace Express {
    interface Request {
      user?: User;
    }
  }
}
```

The authentication middleware must still assign `req.user`. Keep it optional when unauthenticated requests, earlier middleware, tests, or alternate routers can observe a request without a user. Narrow after the middleware with a guard/assertion helper rather than declaring a globally required property that Express cannot guarantee.

Prefer Express's existing generics for route- or pipeline-specific data:

- `Request<Params, ResBody, ReqBody, ReqQuery, Locals>`;
- `Response<ResBody, Locals>`;
- `RequestHandler<Params, ResBody, ReqBody, ReqQuery, Locals>`;
- the `Locals` parameter for typed `res.locals` values.

Use global `Express.Locals` augmentation only for locals that truly exist throughout the application. Use a route's `Locals` generic when middleware state belongs only to that route chain.

Some code augments `'express-serve-static-core'` directly. That can merge with its exported interfaces, but prefer the global `Express` interfaces above for application-wide Request/Response/Locals/Application additions because the declaration source marks them as the intended extension points. Do not augment the `'express'` facade without confirming that the handler types in use actually flow through that declaration.

Middleware packages should avoid claiming required properties on every consumer request. Export a named value type, middleware, and narrowing helper; use optional augmentation only when the package's documented contract requires shared property syntax.

## Treat Node globals and environment variables carefully

The current Node declarations expose `NodeJS.ProcessEnv` as an open interface extending a string dictionary, so it can be augmented:

```ts
export {};

declare global {
  namespace NodeJS {
    interface ProcessEnv {
      APP_MODE?: 'development' | 'production';
    }
  }
}
```

This improves discovery but does not validate the operating-system value. At runtime the variable can still be absent or contain another string. Prefer a bootstrap function that validates `process.env`, returns a precise config object, and makes `ReturnType<typeof loadConfig>` the application contract. Do not make environment keys required globally merely because one deployment is expected to provide them.

Keep augmented `ProcessEnv` properties compatible with `string | undefined`; parse numbers, booleans, URLs, and structured data into separate domain values. Node process environment mutations affect only the process, and worker threads normally receive their own copy, so do not treat augmentation as evidence of cross-worker synchronization.

Apply the same rule to Node globals, request objects, and caches: augment only when the runtime value is intentionally shared, initialize it per process/realm, and expose a function when lifecycle or teardown matters.

## Design deliberate extension points for library consumers

Prefer generics when each instance or call may have a different extension. Use augmentation when independently authored plugins must contribute to one shared registry or interface.

Expose a named interface rather than a type alias or default-only contract:

```ts
export interface ContextExtensions {}

export interface Context extends ContextExtensions {
  requestId: string;
}
```

Consumers can augment `ContextExtensions` through the package's public module specifier, while the library retains its base fields. A similar empty registry interface can map plugin names to payloads and support derived `keyof` unions.

Document all of the following for an extension point:

- the exact module specifier to augment;
- the interface name and whether fields should be optional;
- the runtime registration/import that installs the behavior;
- which package subpath loads the augmentation;
- version and collision expectations;
- a consumer-level type and runtime test.

Do not place declarations in global scope unless the runtime API is actually global. For optional plugins, expose an explicit plugin subpath so importing the base package does not silently widen types for behavior that may never be installed.

## Place and load declaration files intentionally

- Put application augmentations under an included source path such as `src/types/` and confirm the `.d.ts` file matches `files`/`include`.
- Prefer normal project inclusion or an explicit imported augmentation entry over adding `typeRoots`. When `typeRoots` is set, only packages under those roots are auto-included.
- Remember that `compilerOptions.types` limits which visible `@types` packages contribute globals automatically; it does not block types reached through imports.
- Ensure an augmentation package is actually referenced. Publishing a `.d.ts` file does not make it visible unless the package's `types`/`exports` layout or a consumer import reaches it.
- Keep the declaration layout aligned with package export subpaths. Each independently imported subpath may require its own declaration entry.
- Use `.d.ts` for declaration-only patches. Use `.ts` when the same module must also execute a runtime patch or registration.
- Avoid triple-slash references in application code unless package-format constraints require them; use normal inclusion and imports.
- Check `moduleDetection` before assuming a declaration file is a script. Module classification changes ambient `declare module` into augmentation.

Do not respond to a missing augmentation by restarting the editor repeatedly. Use compiler evidence such as `tsc --explainFiles`, `--listFiles`, `--traceResolution`, and `--showConfig` to determine whether the file and target declaration are in the program.

## Verify types and runtime behavior separately

For every material augmentation:

1. Typecheck a minimal consumer through the same public import specifier used by production code.
2. Add a positive check for the intended member and a focused `@ts-expect-error` for an invalid value or unavailable scope.
3. Run a declaration check with `skipLibCheck: false` so merge conflicts are visible.
4. Test the runtime path that installs or initializes the member.
5. Test the missing, failed, reordered, and alternate-entry-point paths.
6. Inspect emitted JavaScript when a prototype patch or side-effect import is required.
7. Re-run the tests when the augmented dependency version changes.
8. Check that the augmentation did not leak into a separate client, server, test, or worker TypeScript program unintentionally.

Treat a type-only success with no runtime test as incomplete. Treat a runtime property accessed through `as` because the augmentation did not load as a configuration defect, not a reason to add another assertion.

## Consult primary sources

- [TypeScript declaration merging and augmentation](https://www.typescriptlang.org/docs/handbook/declaration-merging.html)
- [TypeScript module reference: ambient modules versus augmentation](https://www.typescriptlang.org/docs/handbook/modules/reference.html#ambient-modules)
- [TypeScript global-modifying module template](https://www.typescriptlang.org/docs/handbook/declaration-files/templates/global-modifying-module-d-ts.html)
- [TypeScript module plugin template](https://www.typescriptlang.org/docs/handbook/declaration-files/templates/module-plugin-d-ts.html)
- [TypeScript `globalThis` behavior](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-4.html#type-checking-for-globalthis)
- [TypeScript `types`](https://www.typescriptlang.org/tsconfig/types.html), [`typeRoots`](https://www.typescriptlang.org/tsconfig/typeRoots.html), and [`skipLibCheck`](https://www.typescriptlang.org/tsconfig/skipLibCheck.html)
- [Node.js globals](https://nodejs.org/api/globals.html#global) and [`process.env`](https://nodejs.org/api/process.html#processenv)
- [Current Express core declaration source](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/express-serve-static-core/index.d.ts)

Re-check dependency-owned extension points against the installed declaration version before editing them.
