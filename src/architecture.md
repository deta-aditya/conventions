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

## Modules

The codebase is organized into modules. Each module ideally corresponds to a single bounded
context or domain space. Keeping a module aligned to one bounded context makes it easier to
isolate or extract in the future.

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

Modules interact with each other through data gateways, not through direct unit imports.
A module exposes data gateways as its public contract. Another module that needs data from
it consumes those gateways. The shell is responsible for wiring the actual implementations.

Direct unit imports across module boundaries are not allowed. The one exception is `common`,
which is designed to be a shared dependency and can always be imported directly.

```
module-a                    module-b
  /units                      /units
  /data-gateways  <------     /use-cases
                         |      (depends on module-a's
                         |       data gateway type,
                         |       shell wires the impl)
```

---

## Core

The core contains all pure domain logic organized into five strata. See
[architecture-core.md](./architecture-core.md) for the full breakdown.

```
/module-name
  /polyfills
  /extensions
  /units
  /data-gateways
  /use-cases
```

---

## Shell variants

The shell has multiple variants depending on the runtime context. Each has its own
structural conventions and concerns.

| Variant | Context |
|---|---|
| [UI Shell](./architecture-shell-ui.md) | React applications, browser |
| [Server Shell](./architecture-shell-server.md) | HTTP servers, REST or RPC APIs |
| [Consumer Shell](./architecture-shell-consumer.md) | Message queue consumers |
| [Cron Shell](./architecture-shell-cron.md) | Scheduled jobs |

All shell variants share the same core principles. They implement data gateways defined by
the core, call use cases instead of unit operators directly, keep orchestration thin with no
business logic in the shell, and wire dependencies at the entry point.

---

## Dependency direction

```
common
  ^
  |
module (core)
  /polyfills
  /extensions
  /units
  /data-gateways
  /use-cases
  ^
  |
shell
  (implements data gateways)
  (calls use cases)
  (wires dependencies)
```

From lowest to highest:

1. `common` contains shared primitives with no dependencies
2. Core polyfills fill TS/stdlib gaps
3. Core extensions add operators on language types
4. Core units are domain objects and their operators
5. Core data gateways are contracts for external data
6. Core use cases are entry points for shell operations
7. Shell orchestrates everything and talks to the real world
