# TypeScript Development Loop

Use this reference to design watch, execution, and restart scripts. Treat type-checking, JavaScript emission, execution, and process supervision as separate jobs even when one command coordinates them.

## Choose one loop

| Goal | Default tool | Important behavior |
| --- | --- | --- |
| Report type errors only | `tsc --noEmit --watch` | Uses the project compiler without executing code. |
| Run source with minimum latency | `tsx watch` | Transforms and restarts quickly but does not type-check. |
| Run only successfully emitted JavaScript | `tsc-watch --onSuccess` | Restarts the child only after a successful compilation. |
| Build a library or referenced workspace | `tsc -b --watch` | Preserves declaration and project-reference build semantics. |

Assign process restart ownership to exactly one tool. Do not let `tsx watch`, `tsc-watch`, and another supervisor all restart the same program. If `tsx` owns execution, run `tsc --noEmit --watch` separately for diagnostics and keep the non-watch typecheck mandatory in CI. Use the project's existing multi-process runner rather than adding one by default.

Keep scripts explicit. This single example presents alternatives; choose one `dev:*` owner rather than running both:

```json
{
  "scripts": {
    "dev:source": "tsx watch --clear-screen=false src/server.ts",
    "dev:compiled": "tsc-watch --noClear --onSuccess \"node --enable-source-maps dist/server.js\"",
    "typecheck": "tsc --noEmit",
    "typecheck:watch": "tsc --noEmit --watch --preserveWatchOutput"
  }
}
```

Use project-local dependencies and the repository's package manager. Do not depend on globally installed `typescript`, `tsx`, or `tsc-watch` versions.

## Use `tsx` deliberately

- Use `tsx` for applications, scripts, tests, and CLIs where direct source execution improves iteration speed.
- Pair it with `tsc --noEmit`; `tsx` uses esbuild and intentionally performs no type-checking.
- Use `tsx watch entry.ts` for imported source dependencies. Add `--include` only for relevant non-imported inputs such as configuration or templates, and `--exclude` generated or noisy paths. Quote globs so the shell does not expand them.
- Use `--clear-screen=false` when persistent diagnostics and application logs matter.
- Pass `--tsconfig` when the runtime entry needs a non-default project configuration.
- Prefer `node --import tsx entry.ts` when another Node flag or Node's CLI is the primary interface.
- Do not use `tsx` for compiler-specific emit requirements that esbuild cannot reproduce, such as `emitDecoratorMetadata`. Confirm supported `tsconfig` behavior before relying on a compiler option at runtime.

## Use `tsc-watch` deliberately

- Use plain `tsc --watch` when only compiler diagnostics or emit are needed. Add `tsc-watch` when compilation lifecycle hooks must control another command.
- Use `--onSuccess` to gate a runtime process on a clean compilation. Do not substitute `--onEmit`: emission can occur even when diagnostics fail unless the project prevents it.
- Set `noEmitOnError: true`, a deliberate `rootDir`/`outDir`, and `sourceMap: true` when running emitted JavaScript. Enable Node source maps for useful TypeScript stack traces.
- Keep hook commands single-purpose. Put multi-step behavior in a named package script instead of embedding shell command chains in the hook.
- Let `tsc-watch` terminate its previous child before starting the replacement. Make the application handle `SIGTERM` and close servers, database clients, workers, and telemetry promptly.
- Decide whether the last successful process should remain active while a later compilation has errors; that is the normal `--onSuccess` development model. Use a different supervisor policy if stale-but-working code must stop on failure.
- Use `--compileCommand` only when intentionally selecting a compiler executable other than the normal local `tsc`; do not copy obsolete `--compiler` examples.

## Tune watch mode only when needed

Prefer native file-system events. Configure TypeScript `watchOptions` only when containers, network volumes, editor write patterns, or resource usage justify it. Polling is often more reliable on unusual file systems but costs more CPU.

Exclude emitted output, dependency directories, caches, coverage, and large generated trees from compiler watches. Keep `.tsbuildinfo` in an intentional cache/output location and out of version control unless the project has a specific reason to commit it. Ensure an output write cannot trigger another build or restart loop.

## Prevent development and production drift

- Match `module` and `moduleResolution` to the deployed Node or bundler runtime. A development loader must not conceal invalid production import extensions, package exports, or ESM/CJS behavior.
- Remember that TypeScript `paths` checks imports but does not rewrite emitted specifiers. Verify the production runtime or bundler resolves every alias.
- For libraries, run the declaration-producing build and inspect its output; `tsx` execution is not a substitute for declaration emit.
- Test the actual production entry after a clean build. In CI, run at least the non-watch typecheck, tests, build, and a representative production-runtime check.

## Verify the loop

Introduce a temporary type error and confirm the chosen policy. Then make a valid runtime edit and confirm exactly one restart. Finally, trigger rapid consecutive edits, inspect TypeScript-mapped stack traces, and stop the watcher to verify that no child remains and no port stays occupied.

## Primary sources

- [TypeScript watch configuration](https://www.typescriptlang.org/docs/handbook/configuring-watch.html)
- [`tsx` TypeScript and type-checking guidance](https://github.com/privatenumber/tsx/blob/master/docs/typescript.md)
- [`tsx` watch mode](https://github.com/privatenumber/tsx/blob/master/docs/watch-mode.md)
- [`tsc-watch` usage and lifecycle hooks](https://github.com/gilamran/tsc-watch)
