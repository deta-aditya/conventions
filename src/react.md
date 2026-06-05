# React

This foundation serves as a guideline for writing consistent and maintainable React code.

---

## Naming

Components and Contexts should be in `PascalCase`. Hooks should be in `camelCase` with a
`use` prefix.

```tsx
// ✅ DO

const PrimaryButton = () => { ... }

const UserContext = createContext(...)
const UserProvider = ({ children }: PropsWithChildren) => { ... }
const useUser = () => { ... }

const useCountdown = (targetDate: string) => { ... }

// ❌ DON'T

const primaryButton = () => { ... }     // not PascalCase
const userContext = createContext(...)   // not PascalCase
const UseCountdown = () => { ... }      // not camelCase
const countdown = () => { ... }         // missing use prefix
```

---

## Centralize state

Put state in a centralized location. This simplifies extracting reducer and selector
functions, which are more testable and modular than logic blended inside components.

The state manager used is up to the project. What matters is that state lives in one place,
not scattered across many local `useState` calls.

Local state from `useState` is allowed when the component is fully isolated and the state
does not need to be shared. Use it sparingly. Too many local states lead to
synchronization problems between local and central state.

Also prefer other state sources where appropriate. URL query parameters, routes, and
local storage are valid state containers. Read from them directly instead of mirroring them
into React state, to avoid synchronization issues.

```tsx
// ✅ DO

const SomeComponent = () => {
  const { dispatch, select } = useCentralizedState()

  const data = select(getSomeComponentData)

  const handleClick = () => {
    dispatch(doSomething())
  }

  return (
    // JSX...
  )
}

// ❌ DON'T

const SomeComponent = () => {
  const [data, setData] = useState(...)
  const [otherData, setOtherData] = useState(...)
  const [moreData, setMoreData] = useState(...)

  const handleClick = () => {
    setData(...)
    setOtherData(...)
    setMoreData(...)
  }

  return (
    // JSX...
  )
}
```

---

## Context should have a provider and a usage hook

Never expose a raw Context object for consumers to use with `useContext` directly. Always
wrap it with a dedicated provider component and a usage hook. This keeps the Context
implementation internal and gives consumers a clean, stable API.

The naming pattern is:
- Context object: `XContext` (internal, not exported)
- Provider component: `XProvider` (exported)
- Usage hook: `useX` (exported)

```tsx
// ✅ DO

type UserContextValue = {
  user: User
  logout: () => void
}

const UserContext = createContext<UserContextValue | null>(null)

export const UserProvider = ({ children }: PropsWithChildren) => {
  const [user, setUser] = useState<User>(...)

  const logout = () => { ... }

  return (
    <UserContext.Provider value={{ user, logout }}>
      {children}
    </UserContext.Provider>
  )
}

export const useUser = (): UserContextValue => {
  const context = useContext(UserContext)
  if (context === null) throw new Error('useUser must be used within UserProvider')
  return context
}

// ❌ DON'T

export const UserContext = createContext<UserContextValue | null>(null)

// Consumer now has to use useContext(UserContext) directly
const SomeComponent = () => {
  const context = useContext(UserContext)
  // ...
}
```

---

## Use React Context for dependency injection

External dependencies such as API clients, services, and third-party integrations should be
injected via React Context rather than imported directly inside components or hooks. This
keeps components decoupled from infrastructure and makes them easier to test.

```tsx
// ✅ DO

type ApiClientContextValue = {
  apiClient: ApiClient
}

const ApiClientContext = createContext<ApiClientContextValue | null>(null)

export const ApiClientProvider = ({ client, children }: PropsWithChildren<{ client: ApiClient }>) => (
  <ApiClientContext.Provider value={{ apiClient: client }}>
    {children}
  </ApiClientContext.Provider>
)

export const useApiClient = (): ApiClientContextValue => {
  const context = useContext(ApiClientContext)
  if (context === null) throw new Error('useApiClient must be used within ApiClientProvider')
  return context
}

// ❌ DON'T

import { apiClient } from '@/infrastructure/apiClient'

const useUserData = () => {
  // Directly coupled to the infrastructure layer
  return apiClient.getUser()
}
```

---

## Discourage `useEffect`

`useEffect` is often reached for when state needs to react to other state. This is usually a
sign that the state model is wrong. The fix is derived state, moving logic upstream, or
restructuring the data flow.

`useEffect` should only be used to synchronize with a truly external system: a DOM API, a
WebSocket, a third-party library that lives outside React.

When you find yourself writing `useEffect` to sync state with state, stop and ask: can this
be derived? Can this be moved upstream?

See [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) for
further reading.

```tsx
// ✅ DO

// Derived value, no effect needed
const fullName = `${firstName} ${lastName}`

// ✅ DO

// Syncing with an external system is a valid use of useEffect
useEffect(() => {
  const socket = new WebSocket(url)
  socket.onmessage = (event) => { ... }
  return () => socket.close()
}, [url])

// ❌ DON'T

// Syncing state with state
useEffect(() => {
  setFullName(`${firstName} ${lastName}`)
}, [firstName, lastName])
```

---

## Prefer new component over nested sub-component

Avoid writing internal render functions that return JSX inside a component body. JSX is
already less readable than plain TypeScript. Spreading render functions throughout the
component body makes it worse.

If a piece of JSX is complex enough to extract, always make it a proper component, not an
inline function.

This is also noted as a pitfall in the [React official documentation](https://react.dev/learn/your-first-component#nesting-and-organizing-components).

```tsx
// ✅ DO

const SomeComponent = () => (
  <div>
    <SubComponent />
  </div>
)

const SubComponent = () => (
  <p>SubComponent</p>
)

// ❌ DON'T

const SomeComponent = () => {
  const renderSubComponent = () => (
    <p>SubComponent</p>
  )

  return (
    <div>
      {renderSubComponent()}
    </div>
  )
}
```

---

## Keep components and hooks shallow

Components and hooks should not contain too many branches, loops, or state transformations
inline. When the logic inside a component or hook becomes complex, extract it into plain
functions.

The React layer should be responsible for rendering and wiring, not for heavy computation or
business logic.

```tsx
// ✅ DO

// Logic extracted to a plain function
const buildChartData = (transactions: Transaction[]): ChartData => {
  // complex transformation here
}

const TransactionChart = ({ transactions }: Props) => {
  const chartData = buildChartData(transactions)
  return <Chart data={chartData} />
}

// ❌ DON'T

const TransactionChart = ({ transactions }: Props) => {
  const chartData = transactions
    .filter(t => ...)
    .reduce((acc, t) => {
      // many lines of transformation...
    }, {})

  return <Chart data={chartData} />
}
```

---

## Prefix event handler props with `on`

Event handler props should be prefixed with `on`. This mirrors how HTML exposes event
attributes (`onClick`, `onChange`) and makes it easy to distinguish data props from event
props at a glance.

```tsx
// ✅ DO

type SpecialInputProps = {
  value: string
  onChange: (value: string) => void
  onSubmit: (value: string) => void
  onBlur: () => void
}

// ❌ DON'T

type SpecialInputProps = {
  setValue: (value: string) => void    // not prefixed with on
  whenSubmitted: (value: string) => void
  blur: () => void
}
```

---

## Prefix event handler functions with `handle`

Event handler function implementations should be prefixed with `handle`. This distinguishes
the handler implementation from the prop it is passed to, making it clear which side is the
contract and which is the implementation.

```tsx
// ✅ DO

const SpecialInput = ({ value, onChange, onSubmit, onBlur }: SpecialInputProps) => {
  const handleChange = (e: ChangeEvent<HTMLInputElement>) => onChange(e.target.value)
  const handleSubmit = () => onSubmit(value)

  return <input value={value} onChange={handleChange} onBlur={onBlur} />
}

// ❌ DON'T

const SpecialInput = ({ value, onChange, onSubmit, onBlur }: SpecialInputProps) => {
  const change = (e: ChangeEvent<HTMLInputElement>) => onChange(e.target.value)
  const submit = () => onSubmit(value)

  return <input value={value} onChange={change} onBlur={onBlur} />
}
```

---

## Do not use `FC<Props>`

Avoid typing components with `React.FC`, `React.FunctionComponent`, or `FunctionComponent`.
These add an unnecessary layer of indirection and are inconsistent with how any other
function is typed in the codebase.

Type the props parameter directly instead. This is more explicit, more consistent, and avoids
the implicit `children` prop that `FC` used to include before React 18.

```tsx
// ✅ DO

type ButtonProps = {
  label: string
  onClick: () => void
}

const Button = ({ label, onClick }: ButtonProps) => (
  <button onClick={onClick}>{label}</button>
)

// ❌ DON'T

const Button: React.FC<ButtonProps> = ({ label, onClick }) => (
  <button onClick={onClick}>{label}</button>
)
```

---

## Edge cases

These rules may be broken when a third-party library requires special patterns. For example,
some animation or form libraries have their own component or hook conventions. Only deviate
at the integration boundary. Once the code moves inward, the rules apply normally.

---

## Summary

| Rule | Do | Don't |
|---|---|---|
| Component naming | `PascalCase` | `camelCase` or `MACRO_CASE` |
| Hook naming | `camelCase` with `use` prefix | Missing prefix or `PascalCase` |
| State | Centralize, use local state sparingly | Many scattered `useState` calls |
| Context | Provider + usage hook, internal Context object | Expose raw Context to consumers |
| Dependency injection | Inject via React Context | Import infrastructure directly |
| `useEffect` | Sync with external systems only | Sync state with state |
| Sub-components | Extract as proper components | Inline render functions |
| Component complexity | Extract logic to plain functions | Heavy logic inline |
| Event handler props | Prefix with `on` | Other prefixes or no prefix |
| Event handler functions | Prefix with `handle` | Other prefixes or no prefix |
| Component typing | Type props parameter directly | Use `FC<Props>` or `FunctionComponent<Props>` |
