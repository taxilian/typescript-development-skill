# TypeScript Development Skill

An agentic skill for developing, reviewing, configuring, and migrating TypeScript 7 projects with runtime-faithful, readable, inference-first type design.

The complete skill is in [`typescript-development/`](typescript-development/). Add or link that directory to the skills location used by your agentic development environment.

The skill covers:

- advanced inference and generic API design;
- definition-driven and schema-driven types;
- global, module, and framework type augmentation;
- `satisfies`, `as const`, type guards, and safe runtime boundaries;
- control-flow narrowing, `never`, and refactor-safe exhaustiveness;
- TSDoc, public API documentation, implementation comments, and descriptive naming;
- DRY domain-model composition;
- TypeScript 7 configuration and migration;
- `tsc`, `tsc-watch`, and `tsx` development loops;
- type-level and runtime verification.

Its priorities are runtime truth, intuitive use, one source of truth, and compiler-enforced refactor safety. Sophisticated type machinery is welcome when it is localized, named, documented, and tested—and makes the code using it meaningfully simpler.

Start with [`typescript-development/SKILL.md`](typescript-development/SKILL.md). Supporting references are loaded as needed from [`typescript-development/references/`](typescript-development/references/).
