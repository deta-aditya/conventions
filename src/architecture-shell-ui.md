# Architecture: UI Shell

This document covers the structure and rules for the UI Shell layer. The shell
orchestrates the core via use cases, implements contracts, and renders the UI. It
contains no business logic.

The UI shell is not tied to any UI framework. The examples use React, but every idea here,
such as state, actions, commands, stores, use case providers, and containers, works in any
UI framework, such as Vue, Svelte, or Solid. Use that framework's own tools for each idea:

| Idea | In React | In other frameworks |
|---|---|---|
| Use case provider | React Context | The framework's way of passing values down, such as Vue `provide`/`inject` or Svelte context |
| Page error boundary | An error boundary component | The framework's error catching, such as Vue `onErrorCaptured` or Svelte `<svelte:boundary>` |
| Hooks | Hooks | The framework's reusable logic, such as Vue composables |

Components, hooks, and the like follow the best practices of the framework in use. For
React, see [react.md](./react.md).

---

## Overview

The UI shell is responsible for three things:

1. Implementing contracts defined by the core
2. Wiring contracts and use cases at initialization, outside the UI framework
3. Rendering UI and handling user interactions through the state-action-command model

The shell may call core unit operators directly, including constructors and transformers.
They are pure and immutable, so calling them has no side effects. Operations that need the
outside world, such as fetching or saving data, go through use cases.

---

## Opinionated frameworks

Some UI frameworks decide where files go and how the app is put together. For example,
Next.js, Nuxt, and SvelteKit turn folders into routes, and some frameworks have no `main.tsx`
for you to write. When you use such a framework, follow it. But keep the framework's files
thin, so the architecture still holds:

- Framework files only connect the framework to our code. They contain no logic. For
  example, a Next.js `page.tsx` only renders the page from our module.
- The core never changes for a framework. Use cases and contracts stay plain functions,
  without framework imports.
- When the framework does a job this document describes, let the framework do it. The rule
  behind the job still applies:
  - Routing by folders replaces `module-name-routes.ts`, and the framework's own code
    splitting replaces lazy loading.
  - A built-in error page per route, such as Next.js `error.tsx`, can be the page error
    boundary. Every page still needs one.
  - Without a `main.tsx`, contracts and use cases are still built once, in a single setup
    file that the framework files import.

The rest of this document describes the setup for frameworks that do not decide these
things, including the directory structure.

---

## Directory structure

A UI shell is a client app in `apps/`. Code shared with other client apps goes in a
`client` package, and code shared with server apps goes in a `shell` package. See
[architecture-repository.md](./architecture-repository.md).

```
/apps/web/src
  main.tsx             # entry point, builds contracts and use cases, renders the app
  /common              # optional, shared across all modules
  /module-name
    /pages
      /login
        /units         # page-scoped units
          login-state.ts
          login-action.ts
          login-command.ts
        /extensions    # page-scoped extensions
        /hooks         # page-scoped hooks
          use-login-effect.ts
        /components    # page-scoped presentational components
        /containers    # page-scoped container components
        index.tsx      # sets up the page: providers, error boundary, anything else needed
        login.tsx      # page view component
        login-store.ts # createStore call for this page
        login-styles.ts
    /components        # module-scoped presentational components
      /auth-error-panel
        index.ts
        auth-error-panel.tsx
        auth-error-panel-styles.ts
    /contexts          # module-scoped contexts and use case providers
    /contracts         # shell implementations of core contracts
      /find-user-by-id
        index.ts
        find-user-by-id.http.ts
        find-user-by-id-deps.http.ts
    /units             # module-scoped units
    /extensions        # module-scoped extensions
    /hooks             # module-scoped hooks
    module-name-contracts.ts   # all contract implementations of the module
    module-name-use-cases.ts   # all use cases of the module
    module-name-routes.ts
    module-name-store.ts  # only when necessary
    index.ts
```

Each level (page, module, global) has its own scope for units, extensions, hooks, and
components. A page-scoped unit is not accessible outside that page. A module-scoped unit
is not accessible outside that module.

---

## Shell units

Units in the shell follow the rules in [architecture-unit.md](./architecture-unit.md). Unlike
core units, they may deviate from purity and immutability when needed, for example a unit
that wraps a stateful third-party SDK. Keep the deviation contained to the operators that
need it.

Shell extensions may extend core units, for example to add display formatting to the core
`User` unit.

---

## Page-level units

Pages have their own units following the same one-unit-per-file rule and block structure
as core units. These units model the UI state, actions, and commands for that page. They
have no reason to deviate from purity and immutability, so they stay pure.

The reducer is a transformer on `LoginState` and lives in `login-state.ts`, not in the
store file. Its tuple return type counts as a wrapper of the unit.

```ts
// login-state.ts

// --- Unit ---

export type LoginState = {
  email: string
  password: string
  status: LoginStatus
}

// --- Constructors ---

export const createLoginState = (): LoginState => ({
  email: '',
  password: '',
  status: { kind: 'idle' },
})

// --- Selectors ---

export const isSubmitting = (state: LoginState): boolean =>
  state.status.kind === 'submitting'

// --- Transformers ---

export const loginReducer = (
  state: LoginState,
  action: LoginAction,
): [LoginState, LoginCommand[]] => {
  switch (action.kind) {
    case 'emailChanged':
      return [{ ...state, email: action.value }, []]
    case 'submitPressed':
      return [
        { ...state, status: { kind: 'submitting' } },
        [{ kind: 'submitLogin', email: state.email, password: state.password }],
      ]
    // ...
  }
}
```

```ts
// login-action.ts

// --- Unit ---

export type LoginAction =
  | { kind: 'emailChanged'; value: string }
  | { kind: 'passwordChanged'; value: string }
  | { kind: 'submitPressed' }
  | { kind: 'loginSucceeded'; user: User }
  | { kind: 'loginFailed'; error: AuthError }
```

```ts
// login-command.ts

// --- Unit ---

export type LoginCommand =
  | { kind: 'submitLogin'; email: string; password: string }
```

---

## Store

A store is created per page using `createStore`, which takes the page reducer and returns
the full React integration. The result is destructured and aliased so consumers import
named exports directly instead of accessing them via dot notation.

Stores are page-level by default. A module-level or app-level store is allowed when state
genuinely needs to be shared across pages.

```ts
// login-store.ts

const {
  StoreProvider: LoginStoreProvider,
  useStoreState: useLoginState,
  useStoreDispatcher: useLoginDispatcher,
  useCommandListener: useLoginCommandListener,
} = createStore(loginReducer)

export {
  LoginStoreProvider,
  useLoginState,
  useLoginDispatcher,
  useLoginCommandListener,
}
```

---

## Routes

Each module exposes a routes file that lazy loads pages. This keeps the initial bundle
small by only loading a page when it is navigated to.

```ts
// auth-routes.ts

const LoginPage = lazy(() => import('./pages/login'))
const RegisterPage = lazy(() => import('./pages/register'))

export const authRoutes = [
  { path: '/login', element: <LoginPage /> },
  { path: '/register', element: <RegisterPage /> },
]
```

---

## Page composition root

`index.tsx` is the composition root for the page. It sets up the page with its providers,
error boundary, and anything else the page needs, and wraps the page view. No rendering
logic lives here.

```tsx
// index.tsx

const LoginPage = () => (
  <PageErrorBoundary>
    <LoginStoreProvider initialState={createLoginState()} config={storeConfig}>
      <LoginView />
    </LoginStoreProvider>
  </PageErrorBoundary>
)

export default LoginPage
```

---

## Page error boundary

Every page must have an error boundary at its root, as the outermost component of its
composition root. This is mandatory.

Expected errors are values. They travel as `Result` and `Option` and are handled by the
page through actions. Throwing is reserved for truly unexpected, unrecoverable situations,
and the page error boundary is where those end up. Without it, one failing page takes down
the whole app.

The page error boundary must:

1. Catch every error that nothing inside the page handled
2. Report it, for example to logs or error monitoring
3. Render a fallback in place of the page, while the rest of the app keeps working

It never swallows an error silently.

In React, error boundaries can only be class components. Use a library such as
`react-error-boundary`, or keep a single shared class component at this edge, for example
in `/common/components`.

---

## Page view

`login.tsx` is the actual page view component. It is a container component: it reads state
via selectors, dispatches actions, and calls `useXEffect` when the page has commands to
handle. As the page grows, move state selection into smaller leaf containers. See
[Container components](#container-components).

```tsx
// login.tsx

const LoginView = () => {
  useLoginEffect()

  const email = useLoginState(state => state.email)
  const dispatch = useLoginDispatcher()

  const handleEmailChange = (value: string) => dispatch({ kind: 'emailChanged', value })

  return (
    // JSX...
  )
}
```

---

## Command handling

When a page has commands, a dedicated `useXEffect` hook handles the command-to-use-case
wiring. This hook calls the command listener and maps each command to the relevant use case.
Not every page needs this hook. It is only created when there are commands to handle.

```ts
// use-login-effect.ts

const useLoginEffect = () => {
  const { login } = useAuthUseCase()
  const dispatch = useLoginDispatcher()

  useLoginCommandListener(async (command) => {
    switch (command.kind) {
      case 'submitLogin': {
        const result = await login({ email: command.email, password: command.password })
        result.match({
          ok: user => dispatch({ kind: 'loginSucceeded', user }),
          err: error => dispatch({ kind: 'loginFailed', error }),
        })
        break
      }
    }
  })
}
```

---

## Contract implementations

A contract implementation binds a contract defined in the core to something in the shell,
such as an API client, so that use cases can use it. It is not a use case.

Contract implementations live in `/contracts` at the module level. Implementations receive
their own dependencies via a constructor function, following the same convention as core
use cases.

The filename names the technology the implementation binds to, as
`{contract-name}.{tech}.ts`. The same goes for its deps file. For example,
`find-user-by-id.http.ts` implements `FindUserById` by calling an HTTP client. The tech is
written in kebab-case, such as `http`, `graphql`, `local-storage`, or `indexed-db`. This
makes the binding visible at a glance, and lets one contract have several implementations
side by side.

```
/contracts
  /find-user-by-id
    index.ts
    find-user-by-id.http.ts
    find-user-by-id-deps.http.ts
    find-user-by-id.local-storage.ts
    find-user-by-id-deps.local-storage.ts
```

```ts
// /contracts/find-user-by-id/find-user-by-id-deps.http.ts

type FindUserByIdDeps = {
  apiClient: ApiClient
}
```

```ts
// /contracts/find-user-by-id/find-user-by-id.http.ts

export const createFindUserById = (deps: FindUserByIdDeps): FindUserById =>
  async (request) => {
    const response = await deps.apiClient.get(`/users/${request.id}`)
    if (!response.ok) return Option.none()
    return Option.some(parseUser(response.data))
  }
```

This example returns an `Option`, which only says that no user came back. A `Result` would
also say why, such as the user not existing or the network failing. Both are allowed, but a
`Result` gives clearer data.

Contract implementations are never imported inside components or hooks. They only reach
use cases through initialization.

---

## Parse at the edge

Every piece of data that enters the UI is parsed into a unit where it enters, including
errors. See [architecture.md](./architecture.md#parse-at-the-edge) for the rule. In the UI
shell, data enters here:

| Data | Where it is parsed |
|---|---|
| API responses | In the contract implementation, before it returns |
| API errors | In the contract implementation, into one of our error units |
| Local storage, cookies, and other browser storage | In the contract implementation that reads it |
| WebSocket messages and third-party SDK callbacks | In the contract implementation or effect that receives them |
| URL, route, and query parameters | Where they are read, with a parse operator such as `parseOrderFilters` |
| Form input | When the form is submitted, in the page reducer, with a schema unit |

The contract implementation is the only place that knows what the backend sends. It turns
both the success data and the errors into units:

```ts
// /contracts/verify-credentials/verify-credentials.http.ts

export const createVerifyCredentials = (deps: VerifyCredentialsDeps): VerifyCredentials =>
  async (request) => {
    const response = await deps.apiClient.post('/sessions', request)

    if (response.status === 401) return Result.err({ kind: 'invalidCredentials' })
    if (!response.ok) return Result.err({ kind: 'unavailable' })

    return parseUser(response.data)
  }
```

From there, the error travels as a value. The use case returns it, the reducer stores the
error unit in state, and only the presentational component decides what text to show and how
to show it. Nothing turns the error into text before the component.

Form input is parsed in the reducer, since parsing is pure. When the form is valid, the
reducer issues a command with the parsed request. When it is not, it stores the parse errors
in state for the form to show.

---

## Module contracts and use cases

Each module centralizes its contract implementations in `module-name-contracts.ts`, and its
use cases in `module-name-use-cases.ts`. Both are units: a type listing everything the
module provides, and a constructor that builds it.

```ts
// auth-contracts.ts

// --- Unit ---

export type AuthContracts = {
  findUserById: FindUserById
}

// --- Constructors ---

export const createAuthContracts = (deps: AuthContractsDeps): AuthContracts => ({
  findUserById: createFindUserById({ apiClient: deps.apiClient }),
})
```

```ts
// auth-use-cases.ts

// --- Unit ---

export type AuthUseCases = {
  login: Login
  logout: Logout
}

// --- Constructors ---

export const createAuthUseCases = (contracts: AuthContracts): AuthUseCases => ({
  login: createLogin({ findUserById: contracts.findUserById }),
  logout: createLogout(),
})
```

---

## Initialization

Contracts and use cases are built once, at initialization in `main.tsx`, outside the UI
framework. Components and hooks never build them, so infrastructure such as API clients
does not leak into them.

Initialization is also where modules are wired together. When a use case in one module
depends on a contract from another, it is passed in here.

```tsx
// main.tsx

const apiClient = createApiClient(config)

const authContracts = createAuthContracts({ apiClient })
const authUseCases = createAuthUseCases(authContracts)

createRoot(rootElement).render(
  <AuthUseCaseProvider useCases={authUseCases}>
    <RouterProvider router={router} />
  </AuthUseCaseProvider>
)
```

---

## Use case providers

Each module has one use case provider. It receives the use cases already built at
initialization and makes them available to components and hooks. It never builds anything
itself. The
context object is internal and never exported directly. Consumers use the provided hook.

```tsx
// /contexts/auth-use-case-provider.tsx

const AuthUseCaseContext = createContext<AuthUseCases | null>(null)

type AuthUseCaseProviderProps = PropsWithChildren<{ useCases: AuthUseCases }>

export const AuthUseCaseProvider = ({ useCases, children }: AuthUseCaseProviderProps) => (
  <AuthUseCaseContext.Provider value={useCases}>
    {children}
  </AuthUseCaseContext.Provider>
)

export const useAuthUseCase = (): AuthUseCases => {
  const context = useContext(AuthUseCaseContext)
  if (context === null) throw new Error('useAuthUseCase must be used within AuthUseCaseProvider')
  return context
}
```

In tests, pass use cases built from plain functions to the provider instead.

---

## Components

Components are thin. They contain no business logic, no use case calls, and no direct data
fetching. There are two kinds: presentational components and container components.

Components, hooks, contexts, and stores are framework units. They follow the UI
framework's best practices, such as [react.md](./react.md) for React, not the unit rules. Styles live in a separate
`component-name-styles.ts` file co-located with the component.

### Presentational components

A presentational component receives everything through props, renders it, and reports
events through `on` props. It never reads the store, use cases, or other contexts. This
makes it reusable anywhere and easy to test as a pure function.

Components in `/components` directories should ideally be presentational. They live at the
narrowest scope that covers everything using them:

| Scope | Location | Used by |
|---|---|---|
| Common | `/common/components` | Any module in the app |
| Module | `/module-name/components` | Any page in the module |
| Page | `/module-name/pages/page-name/components` | Only that page |

Start at the page scope. Move a component up a scope when a second page or module needs it.
When another app needs it too, move it into a `client` package.

```tsx
// /common/components/text-input/text-input.tsx

type TextInputProps = {
  label: string
  value: string
  onChange: (value: string) => void
}

export const TextInput = ({ label, value, onChange }: TextInputProps) => (
  // JSX...
)
```

### Container components

A container component connects presentational components to the page: it selects state,
dispatches actions, and passes them down as props. The page view is a container component.

A page view does not have to do all the selecting itself. It may be broken down into
smaller container components, which live in the page's `/containers` directory. The goal
is to:

- Select state in the "leaf" containers, as close as possible to the presentational
  components that need it. Each leaf selects only what it renders, so a state change
  re-renders only the containers that use that state.
- Keep every container small. The page view ends up as mostly layout, composing leaf
  containers.

Only the page view calls the page's `useXEffect` hook.

```tsx
// login.tsx

const LoginView = () => {
  useLoginEffect()

  return (
    <AuthLayout>
      <LoginEmailField />
      <LoginPasswordField />
      <LoginSubmitButton />
    </AuthLayout>
  )
}
```

```tsx
// /containers/login-email-field.tsx

export const LoginEmailField = () => {
  const email = useLoginState(state => state.email)
  const dispatch = useLoginDispatcher()

  const handleChange = (value: string) => dispatch({ kind: 'emailChanged', value })

  return <TextInput label="Email" value={email} onChange={handleChange} />
}
```

Containers are page-scoped, since they bind to the page store. A module-level container
exists only when it binds to a module-level store.

---

## Rules summary

| Rule | Description |
|---|---|
| Opinionated frameworks are followed thinly | Framework files only connect to our code, the core never changes |
| Parse at the edge | All data, including errors, is parsed into units where it enters |
| Errors stay values until the component | Only presentational components turn an error into text |
| Unit operators can be called directly | Core operators are pure and immutable, so the shell may call them |
| Outside world goes through use cases | Fetching or saving data always goes through a use case |
| Contracts and use cases are built at initialization | Once per module in `main.tsx`, never inside components or hooks |
| Reducer is a transformer | Lives in the state unit file, not in the store file |
| Store exports are aliased | Destructure and alias createStore results for named exports |
| Store is page-level by default | Module or app-level only when state is genuinely shared |
| Pages are lazy loaded | Always use lazy imports in the routes file |
| Commands are handled in `useXEffect` | Only created when the page has commands |
| Use cases are injected via context | Never imported directly inside components or hooks |
| Contract implementations use constructor injection | Never call infrastructure directly without deps |
| Contract implementations name their tech | `{contract-name}.{tech}.ts`, such as `find-user-by-id.http.ts` |
| Components are thin | No business logic, no use case calls, no data fetching |
| Components are presentational by default | Props in, JSX out, at the narrowest scope that covers their users |
| Containers select at the leaves | Page views are mostly layout, leaf containers select state |
| One unit per file | Applies to page-level and module-level units equally |
| UI framework agnostic | Any UI framework works, using its own tools for each idea |
| Framework units follow the framework | Components, hooks, contexts, and stores follow the framework's best practices, such as [react.md](./react.md) |
| Shell units may deviate | Purity and immutability can be relaxed when needed, kept contained |
| Page error boundary | Mandatory, the outermost component of every page's `index.tsx` |
| Testing | At least 70% coverage, mocking used sparingly, see [testing.md](./testing.md) |
