# Code Documentation and Comments

Use this reference to document TypeScript APIs, types, and non-obvious implementation logic. Treat documentation as part of the design: it should make contracts and decisions easier to understand without restating information already proved by names and types.

## Contents

1. [Separate API documentation from implementation comments](#separate-api-documentation-from-implementation-comments)
2. [Choose documentation coverage deliberately](#choose-documentation-coverage-deliberately)
3. [Write semantic TSDoc](#write-semantic-tsdoc)
4. [Comment implementation intent](#comment-implementation-intent)
5. [Refactor before narrating](#refactor-before-narrating)
6. [Name values instead of documenting abbreviations](#name-values-instead-of-documenting-abbreviations)
7. [Document interfaces and related fields](#document-interfaces-and-related-fields)
8. [Keep documentation correct](#keep-documentation-correct)
9. [Enforce useful documentation](#enforce-useful-documentation)
10. [Recognize common failure modes](#recognize-common-failure-modes)
11. [Primary references](#primary-references)

## Separate API documentation from implementation comments

Use `/** ... */` TSDoc comments for information a consumer should see in editor hovers or generated API documentation. Attach them directly to declarations such as exported functions, classes, interfaces, type aliases, enums, properties, methods, and constants.

Use `//` comments for information needed only while reading the implementation: why an unusual order is required, which invariant is being maintained, why an apparent simplification is unsafe, or what a coherent phase accomplishes. Do not use TSDoc blocks as visual separators inside a function; tools may associate them with a nearby declaration.

Prefer consecutive `//` lines for a multi-line implementation note. Use blank lines and short phase comments for visual structure; do not draw banners or boxes with punctuation.

## Choose documentation coverage deliberately

Use this default coverage policy:

| Declaration | Documentation default |
| --- | --- |
| Published or public top-level export | Require a TSDoc summary |
| Exported interface, type, class, enum, constant, property, or method | Document its consumer-facing role; document members whose semantics are not obvious |
| Application export crossing a subsystem boundary | Document the contract when usage, side effects, lifecycle, or failure behavior is not completely clear from its name and type |
| Module-local function or type | Document when it expresses a reusable concept, non-obvious contract, algorithm, invariant, or workaround |
| Small callback, obvious wrapper, or short local helper | Let its name, contextual types, and surrounding code explain it |
| Re-export or generated declaration | Keep documentation at the canonical source instead of copying it |

A project may require a one-sentence summary on every named function for consistency, but reject comments that merely repeat the name or signature. Blanket coverage is useful only when comments remain semantic and maintained; otherwise it creates noise that hides the documentation readers actually need.

Document every public declaration in a published library unless the project deliberately excludes tooling-only or generated symbols. For application internals, optimize for information value rather than raw comment count.

## Write semantic TSDoc

Start with a brief summary of the problem solved or observable outcome. Add only sections that contribute information:

- use `@remarks` for invariants, lifecycle, side effects, mutation or ownership, ordering, concurrency, caching, security, performance characteristics, or non-obvious design constraints;
- use `@param name - ...` for units, ranges, formats, defaults, sentinel values, mutation, ownership, or a role that the parameter name and type do not already explain;
- use `@returns` for semantic guarantees such as ordering, identity, ownership, caching, or the meaning of an empty result—not merely to restate the return type;
- use `@typeParam` to explain the relationship a generic represents when its role is not obvious;
- use one `@throws` block per stable error condition callers can reasonably handle, remembering that the tag is informational rather than enforced by TypeScript;
- use `@example` for a non-obvious calling pattern and keep the example minimal and executable where practical;
- use `@defaultValue` on class or interface properties when omission has meaningful runtime behavior;
- use `@deprecated` with a concrete migration path, not only a warning.

Do not repeat TypeScript types in TSDoc tags. The signature is the source of truth for types. Keep the summary brief because documentation tools often display it in member lists and reserve detail for `@remarks`.

```ts
/**
 * Formats a byte count as kibibytes.
 *
 * @param bytes - A non-negative number of bytes.
 * @returns A value rounded to one decimal place with the `KiB` unit.
 * @throws RangeError
 * Thrown when `bytes` is negative.
 */
export function formatKibibytes(bytes: number) {
  if (bytes < 0) {
    throw new RangeError('bytes must be non-negative');
  }
  return `${(bytes / 1024).toFixed(1)} KiB`;
}
```

Place a declaration's TSDoc before its decorators. Use Markdown supported by the project's documentation tool, and verify optional or custom tags against that tool rather than assuming every TSDoc extension is rendered.

## Comment implementation intent

Write an implementation comment when it preserves knowledge that cannot be expressed more clearly with a name, type, helper, or control-flow structure. Good subjects include:

- the reason for an otherwise surprising operation or order;
- an invariant that must hold across several statements;
- a compatibility, security, or performance constraint;
- the relationship to an external protocol, defect, or specification;
- a deliberately non-obvious algorithmic phase;
- the runtime evidence, scoped test assumption, and containment behind a non-obvious assertion, suppression, or intentional sentinel.

Place the comment immediately before the code it governs. Explain why the code exists or what contract the phase maintains; do not translate each statement into English.

Use phase comments only when a function contains multiple meaningful stages that remain clearer together:

```ts
// Normalize aliases first so validation errors use canonical field names.
const normalized = normalizeAliases(input);

// Validate the complete object because constraints span multiple fields.
const validated = validateConfiguration(normalized);
```

Do not number phase comments unless the order is itself a stable protocol or algorithm requirement. Numbered outlines become incorrect when steps are inserted or reordered.

When an escape hatch is not self-explanatory in its immediate context, place a comment directly above the exact expression. For a double assertion, `null!` sentinel, `as any`, lint suppression, or production `@ts-expect-error`, name the runtime fact, deliberately partial/invalid test substitute, inaccurate declaration, narrower construction contract, or compiler limitation involved. Add a removal condition or upstream issue when one exists. Do not write only “TypeScript is wrong” or “safe cast.” A focused invalid-input test may make a one-line `as any` obvious without a separate comment; a repeated explanation is evidence that the source declaration, module augmentation, or boundary adapter should be repaired once instead.

## Refactor before narrating

When a function cannot be understood at a glance, first ask whether it has too many responsibilities, excessive nesting, repeated conditions, or unnamed intermediate concepts. Prefer:

- extracting a well-named helper around a coherent operation;
- introducing a domain-relevant intermediate value;
- replacing flags with a discriminated union;
- using early returns to flatten control flow;
- moving a cohesive group of inputs or state into a named interface;
- encoding an invariant in a type instead of explaining it repeatedly.

Retain a longer function when its steps form one cohesive algorithm and extraction would obscure data flow. In that case, separate meaningful phases with blank lines and concise `//` comments. Function length alone is not the criterion; responsibility count and cognitive load are.

Comments should not serve as an outline that excuses a function with several unrelated jobs. If every small block needs a heading, the blocks probably need names and boundaries in code.

## Name values instead of documenting abbreviations

Rename an ambiguous shortened variable instead of adding a TSDoc comment to explain it. A descriptive identifier stays visible at every use; a declaration comment does not.

Allow short names when their meaning is conventional and local:

- loop indices and mathematical notation in a very small scope;
- ubiquitous forms such as `id`, `url`, or `http`;
- a domain abbreviation already understood by the intended audience and defined at the domain boundary;
- an external protocol field whose spelling must be preserved.

Do not create abbreviations by deleting letters from words. When an external shape forces an unfamiliar short field, document the property on its interface and translate it to a descriptive internal name when practical. Use TSDoc on exported constants or public properties; use a rename or, rarely, a nearby `//` explanation for a local value.

## Document interfaces and related fields

Give an exported interface or type a summary describing what it represents, who creates it, and where it is valid when those facts are not obvious. Document properties when callers need semantics beyond their TypeScript type, including:

- units, ranges, encodings, formats, or identifier namespaces;
- the difference between absence, `undefined`, `null`, and an empty value;
- defaults and when they are applied;
- mutability, ownership, lifecycle, or lazy initialization;
- privacy or security sensitivity;
- cross-field requirements and mutually exclusive combinations.

Attach property documentation directly to the property. Do not repeat property types with `@property` tags in TypeScript source.

When fields form a real domain concept or share invariants, prefer extracting a nested named interface. This gives the group a reusable name and lets its documentation travel with the type. If a flat external shape must be preserved, use blank lines and restrained `//` headings for source navigation while documenting each non-obvious property individually. A TSDoc block labeled as a “group” immediately before a property normally documents that property, not an abstract section.

Document a discriminated union as a whole and add member documentation where a variant's lifecycle or payload semantics are non-obvious. Document cross-member invariants once at the nearest shared declaration instead of copying them into every member.

## Keep documentation correct

Treat public TSDoc as part of the API contract:

- update documentation in the same change as behavior, parameters, return semantics, defaults, or errors;
- remove comments that no longer add information;
- keep examples typechecked or executable when the project can support it;
- link to an issue or external specification for a workaround, and state the condition for removing it;
- remove or revise an escape-hatch comment when the declaration, compiler, or runtime contract changes;
- write TODOs with a concrete reason and tracking reference when the project has an issue system;
- use version control for history instead of narrating past implementations in current comments;
- review comments for claims that types or tests could enforce more reliably.

Do not depend on arbitrary implementation comments surviving JavaScript emission. Documentation needed by package consumers belongs in the public declarations and generated documentation pipeline.

## Enforce useful documentation

Follow the project's established TSDoc/JSDoc dialect and documentation generator. Do not introduce incompatible tags casually.

Use tooling in layers when the project benefits from generated API documentation:

1. Validate TSDoc syntax so malformed comments do not silently render incorrectly. `eslint-plugin-tsdoc` provides syntax validation but does not prove coverage or semantic quality.
2. Validate generated-document links and exported references.
3. For published libraries, opt into missing-documentation checks for the intended public reflection kinds. TypeDoc provides `validation.notDocumented` and `requiredToBeDocumented`; missing-documentation validation is not enabled by default.
4. Treat validation warnings as errors in CI only after the scope is tuned to avoid forcing meaningless comments on generated, tooling-only, or intentionally undocumented symbols.

Coverage rules can prove that a comment exists, not that it is useful or correct. Review semantic content and avoid generated placeholder prose.

## Recognize common failure modes

- Repeating the function, parameter, property, or return type in prose without adding semantics.
- Adding TSDoc to every trivial local declaration until meaningful documentation is difficult to find.
- Using `/** ... */` for implementation steps or visual separators.
- Keeping an ambiguous abbreviation and compensating with a comment instead of renaming it.
- Using comments to describe a type relationship that should be derived or enforced.
- Narrating every statement in a long function instead of extracting responsibilities.
- Drawing large comment banners that dominate the code or become landmarks for oversized functions.
- Copying the same contract into an interface, implementation, and wrapper so the versions drift.
- Documenting a field group with a comment that tooling attaches only to the following property.
- Leaving examples, defaults, units, links, error behavior, or TODOs stale after a refactor.
- Treating `@throws` as an enforced exception type or a TSDoc assertion as runtime validation.
- Leaving a non-obvious `as unknown as T`, `as any`, `null!`, or suppression unexplained, so reviewers cannot verify its runtime proof, test limitation, containment, or removal condition.
- Enforcing raw documentation coverage before deciding which API surface actually requires documentation.

## Primary references

- [TSDoc approach and standardized syntax](https://tsdoc.org/pages/intro/approach/)
- [TSDoc summary and `@remarks`](https://tsdoc.org/pages/tags/remarks/)
- [TSDoc `@param`](https://tsdoc.org/pages/tags/param/), [`@returns`](https://tsdoc.org/pages/tags/returns/), [`@throws`](https://tsdoc.org/pages/tags/throws/), [`@typeParam`](https://tsdoc.org/pages/tags/typeparam/), [`@example`](https://tsdoc.org/pages/tags/example/), and [`@defaultValue`](https://tsdoc.org/pages/tags/defaultvalue/)
- [Google TypeScript style guide: comments, documentation, and descriptive names](https://google.github.io/styleguide/tsguide.html#comments-documentation)
- [TypeDoc comment discovery](https://typedoc.org/documents/Doc_Comments.html)
- [TypeDoc documentation validation](https://typedoc.org/documents/Options.Validation.html)
- [`eslint-plugin-tsdoc` syntax validation](https://tsdoc.org/pages/packages/eslint-plugin-tsdoc/)
