# Architecture: React UI Shell

This document covers the structure and rules for the React UI Shell layer. The shell
orchestrates the core via use cases, implements data gateways, and renders the UI. It
contains no business logic.

---

## Overview

The UI shell is responsible for three things:

1. Implementing data gateways defined by the core
2. Wiring use case implementations at the composition root
3. Rendering UI and handling user interactions through the state-action-command model

The shell never calls unit operators directly. All interactions with the core go through
use cases.

---

## Directory structure

```
/shell-ui
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
        /components    # page-scoped components
        index.tsx      # composition root, wires providers
        login.tsx      # page view component
        login-store.ts # createStore call for this page
        login-styles.ts
    /components        # module-scoped reusable components
      /auth-error-panel
        index.ts
        auth-error-panel.tsx
        auth-error-panel-styles.ts
    /contexts          # module-scoped contexts and use case providers
    /data-gateways     # shell implementations of core data gateway contracts
      /find-user-by-id
        index.ts
        find-user-by-id.ts
        find-user-by-id-deps.ts
    /units             # module-scoped units
    /extensions        # module-scoped extensions
    /hooks             # module-scoped hooks
    module-name-routes.ts
    module-name-store.ts  # only when necessary
    index.ts
```

Each level (page, module, global) has its own scope for units, extensions, hooks, and
components. A page-scoped unit is not accessible outside that page. A module-scoped unit
is not accessible outside that module.

---

## Page-level units

Pages have their own units following the same one-unit-per-file rule and block structure
as core units. These units model the UI state, actions, and commands for that page.

The reducer is a transformer on `LoginState` and lives in `login-state.ts`, not in the
store file.

```ts
// login-state.ts

// --- Unit ---

type LoginState = {
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

type LoginAction =
  | { kind: 'emailChanged'; value: string }
  | { kind: 'passwordChanged'; value: string }
  | { kind: 'submitPressed' }
  | { kind: 'loginSucceeded'; user: User }
  | { kind: 'loginFailed'; error: AuthError }
```

```ts
// login-command.ts

// --- Unit ---

type LoginCommand =
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

`index.tsx` is the composition root for the page. It wires all providers and wraps the
page view. No rendering logic lives here.

```tsx
// index.tsx

const LoginPage = () => (
  <AuthUseCaseProvider>
    <LoginStoreProvider initialState={createLoginState()} config={storeConfig}>
      <LoginView />
    </LoginStoreProvider>
  </AuthUseCaseProvider>
)

export default LoginPage
```

---

## Page view

`login.tsx` is the actual page view component. It reads state via selectors, dispatches
actions, and calls `useXEffect` when the page has commands to handle.

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

## Use case providers

Use case implementations are wired at the module level via a context provider. One provider
covers all use cases for a domain module. The context object itself is internal and never
exported directly. Consumers use the provided hook.

```tsx
// /contexts/auth-use-case-provider.tsx

type AuthUseCaseContextValue = {
  login: Login
  logout: Logout
}

const AuthUseCaseContext = createContext<AuthUseCaseContextValue | null>(null)

export const AuthUseCaseProvider = ({ children }: PropsWithChildren) => {
  const findUserById = useFindUserById()

  const login = createLogin({ findUserById })
  const logout = createLogout()

  return (
    <AuthUseCaseContext.Provider value={{ login, logout }}>
      {children}
    </AuthUseCaseContext.Provider>
  )
}

export const useAuthUseCase = (): AuthUseCaseContextValue => {
  const context = useContext(AuthUseCaseContext)
  if (context === null) throw new Error('useAuthUseCase must be used within AuthUseCaseProvider')
  return context
}
```

---

## Data gateway implementations

Data gateway implementations live in `/data-gateways` at the module level. Each implementation
fulfills a contract defined by the core. Implementations receive their own dependencies via
a constructor function, following the same convention as core use cases.

```ts
// /data-gateways/find-user-by-id/find-user-by-id-deps.ts

type FindUserByIdDeps = {
  apiClient: ApiClient
}
```

```ts
// /data-gateways/find-user-by-id/find-user-by-id.ts

export const createFindUserById = (deps: FindUserByIdDeps): FindUserById =>
  async (request) => {
    const response = await deps.apiClient.get(`/users/${request.id}`)
    if (!response.ok) return Option.none()
    return Option.some(parseUser(response.data))
  }
```

Data gateway implementations are injected into use case providers, never imported directly
inside components or hooks.

---

## Components

Components are thin. They read state via selectors, dispatch actions, and render JSX.
They contain no business logic, no use case calls, and no direct data fetching.

Page-scoped components live inside the page directory. Module-scoped components that are
reused across pages live in `/components` at the module level.

Each component follows the same one-unit-per-file convention. Styles live in a separate
`component-name-styles.ts` file co-located with the component.

---

## Rules summary

| Rule | Description |
|---|---|
| No direct unit operator calls | All core interactions go through use cases |
| Reducer is a transformer | Lives in the state unit file, not in the store file |
| Store exports are aliased | Destructure and alias createStore results for named exports |
| Store is page-level by default | Module or app-level only when state is genuinely shared |
| Pages are lazy loaded | Always use lazy imports in the routes file |
| Commands are handled in `useXEffect` | Only created when the page has commands |
| Use cases are injected via context | Never imported directly inside components or hooks |
| Data gateways use constructor injection | Never call infrastructure directly without deps |
| Components are thin | No business logic, no use case calls, no data fetching |
| One unit per file | Applies to page-level and module-level units equally |
