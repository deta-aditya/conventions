# Conventions

Coding and architecture conventions for TypeScript and React codebases.

Current version: **0.2.0**. See the [changelog](CHANGELOG.md) for what changed between
versions.

---

## Documents

Start with [Architecture](src/architecture.md), then read the rest as needed.

| Document | Covers |
|---|---|
| [Architecture](src/architecture.md) | Functional core and imperative shell, modules, dependency direction |
| [Architecture: Repository](src/architecture-repository.md) | `apps/` and `packages/`, package categories, import rules, package exports, and the conventions version |
| [Architecture: Unit](src/architecture-unit.md) | Units, operators, and extensions, the building blocks used in every layer |
| [Architecture: Core](src/architecture-core.md) | Extensions, units, contracts, and use cases in the core |
| [Architecture: React UI Shell](src/architecture-shell-ui.md) | Pages, page error boundaries, stores, commands, initialization, use case providers, and contract implementations |
| [Architecture: Server Shell](src/architecture-shell-server.md) | Configuration, initialization, routes, handlers, DTOs, error mapping, and authentication |
| [Testing](src/testing.md) | Coverage targets, test structure, test data, mocking, and testing framework units |
| [Naming](src/naming.md) | Casing, booleans, type parameters, discriminated union variants, use cases, contracts, and files |
| [TypeScript](src/typescript.md) | Type definitions, functions, classes, parsing, and inference |
| [React](src/react.md) | Components, hooks, state, context, and effects |

The consumer and cron shell documents are planned but not written yet.

---

## Key terms

| Term | Meaning |
|---|---|
| Core | Pure, deterministic domain logic with no IO or framework imports |
| Shell | The outer layer that talks to the real world and orchestrates the core |
| App | A deployable shell in `apps/`, never imported by anything |
| Package | Shared code in `packages/`, with a category declared in its `package.json`: `polyfill`, `core`, `shell`, `client`, `server`, or `tooling` |
| Module | A slice of the core aligned to one bounded context |
| Unit | A single exported type and all the operators that work on it, in any layer |
| Operator | A constructor, selector, or transformer of a unit |
| Framework unit | A building block defined by a framework, such as a React component or hook, which follows the framework instead of the unit rules |
| Extension | Operators added to a unit the current scope does not own |
| Contract | A function type in the core describing external data the domain needs, implemented by the shell |
| Use case | A function type the shell calls to perform an operation that needs the outside world |
| Page error boundary | The mandatory React error boundary at the root of every page in a client app |

---

## Versioning

The conventions are versioned as a whole using [Semantic Versioning](https://semver.org).
Each release is tagged `vX.Y.Z` in git and listed in the [changelog](CHANGELOG.md).

- **Major**: code that followed the previous version no longer complies, such as a renamed
  concept, a reversed rule, or a newly required structure.
- **Minor**: new guidance that does not make compliant code non-compliant, such as a new
  document, a new optional pattern, or a relaxed rule.
- **Patch**: wording, typo, and example fixes that do not change any rule.

While the version is below 1.0.0, breaking changes bump the minor version instead of the
major version.

Projects that adopt these conventions declare which version they follow in the
`conventions.version` field of their root `package.json`. A package or app may override it
while it migrates to a newer version. See
[architecture-repository.md](src/architecture-repository.md#conventions-version).
