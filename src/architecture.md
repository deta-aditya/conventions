# Architecture

This document serves as an overview of the architectural principles and structure used
across the codebase. Each section links to a more detailed document covering its specific
concerns.

---

## Functional Core, Imperative Shell

The codebase is divided into two layers.

**Functional Core** is where pure domain logic lives. No side effects, no framework imports,
no IO. Everything here is deterministic: same input always produces same output.

**Imperative Shell** is the outer layer that talks to the real world. React components, hooks,
API calls, storage, timers, consumers, and cron jobs all live here. It orchestrates the core
but contains no business logic itself.

The shell depends on the core. The core never depends on the shell.

---

## Repository

Every repository uses `apps/` and `packages/` at the root. The core always lives in
packages. Deployable shells are apps. Code shared between shells also goes in packages.

```
/apps
  /web
  /api
/packages
  /lang               # polyfill
  /core               # core
  /dto                # shell
  /design-system      # client
  /database           # server
  /eslint-config      # tooling
```

Every package belongs to one category: `polyfill`, `core`, `shell`, `client`, `server`, or
`tooling`. The category is declared in the package's `package.json` and decides who may
import it. An app is never imported, not even by
another app. See [architecture-repository.md](./architecture-repository.md) for the
categories and the full import rules.

---

## Modules

The core is organized into modules, kept in core packages. Each module ideally corresponds
to a single bounded context or domain space. Keeping a module aligned to one bounded
context makes it easier to isolate or extract in the future.

```
/common        # optional shared kernel
/module-a
/module-b
/module-c
```

### Common module

`common` is an optional shared kernel for primitives that are used across multiple modules,
such as `Email`, `Money`, or `Pagination`. It sits at the bottom of the dependency chain.
Any module can import from `common` directly.

Not every application needs a `common` module. It should only be introduced when there are
genuinely shared primitives across multiple bounded contexts.

### Inter-module interaction

Modules interact with each other through contracts, not through direct unit imports.
A module exposes contracts as its public interface. Another module that needs data from
it consumes those contracts. The shell is responsible for wiring the actual implementations.

Direct unit imports across module boundaries are not allowed. The one exception is `common`,
which is designed to be a shared dependency and can always be imported directly.

```
module-a                    module-b
  /units                      /units
  /contracts      <------     /use-cases
                         |      (depends on module-a's
                         |       contract type,
                         |       shell wires the impl)
```

---

## Units

A unit is a single exported type and all the operators that work on it. Units are the
building block of every layer, not only the core. The core uses them for domain objects.
The shell uses them for things like page state, actions, and commands.

Core units must be pure and immutable. Shell units follow the same rules by default but may
deviate when needed, such as when interacting with a third-party API. See
[architecture-unit.md](./architecture-unit.md) for the full rules, including extensions.

---

## Core

The core contains all pure domain logic organized into four strata. See
[architecture-core.md](./architecture-core.md) for the full breakdown.

```
/module-name
  /extensions
  /units
  /contracts
  /use-cases
```

---

## Shell variants

The shell has multiple variants depending on the runtime context. Each has its own
structural conventions and concerns, and each is deployed as one or more apps.

| Variant | Context |
|---|---|
| [UI Shell](./architecture-shell-ui.md) | React applications, browser |
| [Server Shell](./architecture-shell-server.md) | HTTP servers, REST or RPC APIs |
| [Consumer Shell](./architecture-shell-consumer.md) | Message queue consumers |
| [Cron Shell](./architecture-shell-cron.md) | Scheduled jobs |

All shell variants share the same core principles. They implement contracts defined by the
core, reach the outside world through use cases, keep orchestration thin with no
business logic in the shell, and wire dependencies at the entry point.

---

## Testing

Core unit tests should ideally reach 100% coverage without mocks. Shell unit tests should
reach at least 70%, with mocking used sparingly. See [testing.md](./testing.md) for how
tests are written.

---

## Dependency direction

```
polyfill packages
  ^
  |
common
  ^
  |
module (core)
  /extensions
  /units
  /contracts
  /use-cases
  ^
  |
shell
  (implements contracts)
  (calls use cases)
  (wires dependencies)
```

From lowest to highest:

1. Polyfill packages fill TS/stdlib gaps
2. `common` contains shared primitives
3. Core extensions add operators to units the module does not own
4. Core units are domain objects and their operators
5. Core contracts describe external data the domain needs
6. Core use cases are entry points for shell operations
7. Shell orchestrates everything and talks to the real world
