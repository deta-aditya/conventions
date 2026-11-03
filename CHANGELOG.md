# Changelog

All notable changes to these conventions are listed here. Versioning rules are described in
the [README](README.md#versioning).

---

## 0.2.0 - 2026-11-02

### Changed (breaking)

**Repository**

- Repositories always use `apps/` and `packages/` at the root. Deployable shells are apps,
  and the core lives in `core` packages.
- Every package and app declares its category in `conventions.category` in its
  `package.json`, and follows the import rules between categories.
- Every package lists explicit entry points in the `exports` field of its `package.json`,
  with no wildcards.
- The root `package.json` declares the conventions version it follows in
  `conventions.version`.

**Core**

- Renamed "data gateway" to "contract". The `/data-gateways` directory is now `/contracts`,
  in both the core and the shell.
- Polyfills moved out of core modules into `polyfill` packages. Core modules now have four
  strata: extensions, units, contracts, and use cases.
- Use cases are required only for operations that need the outside world. The shell may
  call core unit operators directly, including constructors and transformers.

**Units**

- A unit file exports only the unit's type and its operators.
- Selectors and transformers take the unit as their first parameter, and the unit parameter
  is never wrapped.
- Extensions extend a unit the current scope does not own, such as a language type, a
  polyfill or library type, a `common` unit, or, in the shell, a core unit. A unit the scope
  owns gets new operators in its own file instead.
- Extension files are named `{unit-name}.ext.ts` instead of `{unit-name}.ts`.

**UI shell**

- The UI shell lives in `apps/{app-name}/src`.
- Each module centralizes its contract implementations in `module-name-contracts.ts` and its
  use cases in `module-name-use-cases.ts`. Both are built once at initialization in
  `main.tsx`, outside React.
- Use case providers receive the use cases built at initialization instead of constructing
  them, and are set up in `main.tsx` instead of in each page.
- Contract implementation files name the technology they bind to, as
  `{contract-name}.{tech}.ts`, such as `find-user-by-id.http.ts`. Their deps files follow
  the same pattern, such as `find-user-by-id-deps.http.ts`.
- Every page must have an error boundary as the outermost component of its `index.tsx`.

**React**

- Infrastructure such as API clients stays out of React and reaches it only through use
  cases. When it absolutely has to enter React, it is wrapped in a context first. This
  replaces injecting infrastructure via React Context.

**Testing**

- Core and polyfill packages should ideally reach 100% unit test coverage without mocks.
  Apps and shell, client, and server packages must reach at least 70%, with mocking used
  sparingly.

**Naming**

- Use cases are named after the business operation, with a domain verb and an object, and
  never after a contract they call.

### Added

**New documents**

- [architecture-unit.md](src/architecture-unit.md): units as a concept used in every
  layer, not only the core. The unit rules moved here from
  [architecture-core.md](src/architecture-core.md).
- [architecture-repository.md](src/architecture-repository.md): apps, the six package
  categories (`polyfill`, `core`, `shell`, `client`, `server`, `tooling`), import rules,
  package exports, and the conventions version.
- [testing.md](src/testing.md): coverage targets, test placement and structure, test names,
  arrange-act-assert, test data and fixtures, assertions, determinism, and mocking.
- [architecture-shell-server.md](src/architecture-shell-server.md): configuration,
  contract implementations, initialization, routes, handlers, DTOs, error mapping,
  authentication and authorization, unexpected errors, and testing for server apps. When a
  server framework decides its own structure, follow it, but keep its files thin so the
  architecture still holds.
- README and changelog, with a versioning policy.

**Units**

- A unit and its operators form an abstraction barrier.
- A Type section covering custom types, discriminated unions, branded types, classes, and
  schemas. A schema with its inferred type counts as a unit, and the schema may be exported
  alongside the type.
- An Operators section, with a table of the parameters, return value, type signature, and
  usage of constructors, selectors, and transformers.
- `Record<U, V>` is a valid selector.
- Constructors and transformers may wrap the unit in a generic type, and a tuple counts as
  one. A reducer returning `[State, Command[]]` is a transformer.
- Class units: methods are operators, with `this` as the implicit first parameter.
- Framework units: components, hooks, contexts, providers, and stores follow the framework's
  best practices instead of the unit rules.
- Core units must be pure and immutable. Shell units may deviate when needed, such as when
  wrapping a third-party SDK, as long as the deviation stays contained.
- An expanded stratified design section: use cases sit at the top as very specific units,
  code is generic and brittle at the bottom and specific and fluid at the top, unit tests
  are more valuable at the bottom, and reaching too deep may signal a missing unit.

**Core**

- Contract implementations bind a core contract to something in the shell. They are not use
  cases.

**UI shell**

- Initialization in `main.tsx`, which is also where modules are wired together.
- Presentational components live at the common, module, or page scope, starting at the page
  scope and moving up when shared.
- Container components: page views may be broken down into leaf containers in a page's
  `/containers` directory, which select state close to where it is rendered.

**Testing**

- Spend the most testing effort at the bottom strata, while keeping the same coverage
  target everywhere.
- Test rules for presentational components, container components, hooks, and server
  handlers.

**Naming**

- Contracts are named after the operation they perform on the outside world, with any verb
  that describes it clearly.
- File naming: `kebab-case`, with hyphen suffixes for related things and dot suffixes for
  roles, and a table of every file pattern, including server handlers, middlewares, and
  DTOs.

---

## 0.1.0

- Initial conventions: architecture overview, core, React UI shell, naming, TypeScript, and
  React.
