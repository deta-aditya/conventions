# Architecture: Repository

This document covers how a repository is laid out and which parts of it may import which.

---

## Overview

Every repository uses `apps/` and `packages/` at the root, conceptually similar to a
Turborepo monorepo. This applies even when the repository holds a single app.

- **Apps** are deployable shells.
- **Packages** are code shared by apps or by other packages. The core always lives in
  packages.

```
/apps
  /web                # client app, UI shell
  /api                # server app, server shell
  /worker             # server app, consumer shell
/packages
  /lang               # polyfill
  /core               # core
  /dto                # shell
  /design-system      # client
  /database           # server
  /eslint-config      # tooling
```

---

## Apps

An app is a deployable shell. Each shell variant, such as the
[UI shell](./architecture-shell-ui.md) or the [server shell](./architecture-shell-server.md),
is deployed as one or more apps. The app is the
composition root: it wires contract implementations into use cases and owns the entry
point.

An app is never imported by anything, including another app. Code that more than one app
needs goes in a package.

Each app runs in one of two runtimes:

- **Client apps** run in the browser or another client, such as the UI shell.
- **Server apps** run on a server, such as the server, consumer, and cron shells.

A fullstack app that runs in both follows the client rules in its client code and the
server rules in its server code.

An app declares its runtime in its `package.json`, the same way a package declares its
category:

```json
{
  "name": "@acme/web",
  "conventions": {
    "category": "client"
  }
}
```

---

## Packages

A package is code shared across the repository. Every package belongs to exactly one
category, and the category decides who may import it.

The category is declared in the package's `package.json`, under a `conventions` field. npm
and other package managers ignore fields they do not know, so this is safe. Keeping it
under its own `conventions` field avoids clashing with any field a tool may add in the
future. The category is not part of the package's name or directory.

```json
{
  "name": "@acme/billing",
  "conventions": {
    "category": "core"
  }
}
```

| Category | Holds | Example |
|---|---|---|
| `polyfill` | Fills gaps in the TypeScript language and standard library | `Option`, `Result`, union constructors |
| `core` | Business domain logic, the functional core | Units, contracts, and use cases of a bounded context |
| `shell` | Reusable shell code that stays isomorphic, running on both client and server | DTOs shared by the client and server of a fullstack app |
| `client` | Shell code specific to client apps | Shared React components, browser contract implementations |
| `server` | Shell code specific to server apps | Database clients, server contract implementations |
| `tooling` | Repository-level tools, outside the apps' business flow | ESLint and TypeScript configs, code generators, scripts |

### Polyfill

Polyfill packages fill gaps in TypeScript, such as `Option<T>`, `Result<T, E>`, and
discriminated union constructors. They can be imported by anyone in the repository,
including the core. They import nothing but other polyfill packages.

They are temporary by nature. Once a library covers the need (`gado` for `Option`/`Result`,
`yamin` for unions, `zod` for validation schemas), the polyfill package is replaced by the
library. The rest of the codebase does not need to change since the interface stays the
same.

### Core

Core packages hold the business domain logic, organized into modules as described in
[architecture-core.md](./architecture-core.md). They can be imported by anyone in the
repository.

A core package may hold one module or several. Start with one, and move a module into its
own package when it needs to be isolated. The module rules still apply across packages: a
core package imports another core package only for `common` or for contract types, never
for units. When `common` is needed, it is its own core package, such as `packages/common`.

### Shell

Shell packages hold shell code that more than one shell needs, such as DTOs shared by the
client and server of a fullstack app. They must stay isomorphic: no browser-only and no
server-only code.

Only shells may import them: apps, and shell, client, or server packages. The core never
imports a shell package.

### Client and server

Client packages hold shell code shared by client apps. Server packages hold shell code
shared by server apps. A client package never imports a server package, and a server
package never imports a client package.

### Tooling

Tooling packages support developing, building, and maintaining the repository. They are
outside the apps' business flow, so source code never imports them. Apps and packages use
them only from config files and scripts, for example a `tsconfig.json` that extends a
shared config.

---

## Import rules

Each row shows what that kind of code may import.

| Importer | `polyfill` | `core` | `shell` | `client` | `server` | `tooling` | App |
|---|---|---|---|---|---|---|---|
| `polyfill` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `core` | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `shell` | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `client` | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| `server` | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| Client app | ✅ | ✅ | ✅ | ✅ | ❌ | Config only | ❌ |
| Server app | ✅ | ✅ | ✅ | ❌ | ✅ | Config only | ❌ |
| `tooling` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |

In short:

1. Nothing imports an app.
2. Polyfill and core packages can be imported by anyone.
3. Shell packages can be imported only by shells.
4. Client packages stay on the client and server packages stay on the server.
5. Tooling is used only from config files and scripts.

These rules should be enforced by a lint rule kept in a tooling package, not by review
alone. The lint rule reads each package's category from its `package.json`.

---

## Package exports

Every package declares its public entry points in the `exports` field of its
`package.json`. Anything not listed there cannot be imported from outside the package, so
deep imports into its internals fail at build time instead of being caught in review.

- List each entry point explicitly. Never use a wildcard such as `"./*"`, which exposes
  every file.
- Most packages have a single entry point, `"."`, pointing to their root `index.ts`.
- Add another entry point only when importers genuinely need a narrower or separate part of
  the package.

```json
{
  "name": "@acme/lang",
  "exports": {
    ".": "./src/index.ts"
  }
}
```

Core packages follow the same rule. The module rules, such as a module importing only the
contract types of another module, are enforced by the lint rule, not by `exports`.

---

## Conventions version

The repository declares the version of these conventions it follows, in the `conventions`
field of the root `package.json`.

```json
{
  "name": "@acme/root",
  "conventions": {
    "version": "0.2.0"
  }
}
```

A package or app may declare its own version, which overrides the root one for that package
or app. This lets a repository move to a new version one package at a time, instead of
breaking everything at once when a version changes a rule. Once every package has moved,
the overrides are removed and the root version is raised.

```json
{
  "name": "@acme/billing",
  "conventions": {
    "category": "core",
    "version": "0.1.0"
  }
}
```

Lint rules and reviewers check each package against the version it declares, or the root
version when it declares none.

---

## Summary

| Rule | Description |
|---|---|
| Root layout | Always `apps/` and `packages/` |
| Apps | Deployable shells, never imported by anything |
| Core location | Always in `core` packages |
| Shared shell code | Goes in a `shell`, `client`, or `server` package |
| Category declaration | `conventions.category` in `package.json`, for packages and apps |
| Package categories | `polyfill`, `core`, `shell`, `client`, `server`, `tooling` |
| Import rules | Follow the import rules table, enforced by a lint rule |
| Package exports | Explicit entry points in `exports`, no wildcards |
| Conventions version | `conventions.version` in the root `package.json`, overridable per package or app |
