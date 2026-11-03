# Testing

This document covers how unit tests are written and how much of the code they must cover.

Examples use the Vitest and Jest API (`describe`, `it`, `expect`). Any test runner with the
same shape works.

---

## Coverage

| Code | Target |
|---|---|
| `core` and `polyfill` packages | Ideally 100% |
| Apps, and `shell`, `client`, and `server` packages | At least 70% |
| `tooling` packages | No target |

Reaching 100% in the core is easy because everything there is pure and immutable. A test
gives an operator its input and asserts on its output. No mocks, no setup, no teardown.

The target is the same across all strata, but the effort is not. Spend the most testing
effort at the bottom strata, where code is generic and a bug spreads to every unit above it.
Test the bottom thoroughly first, with edge cases, then cover the top. See
[stratified design](./architecture-unit.md#bottom-versus-top).

---

## File placement and naming

A test file lives next to the file it tests and is named after it, with `.test` before the
extension.

```
/units
  /user
    index.ts
    user.ts
    user.test.ts
/extensions
  /array
    index.ts
    array.ext.ts
    array.ext.test.ts
/components
  /auth-error-panel
    index.ts
    auth-error-panel.tsx
    auth-error-panel.test.tsx
```

---

## Test through the abstraction barrier

Test a unit only through its exported operators. Private items are covered through the
operators that use them. Never export something only so that it can be tested.

If a private item is hard to cover through the operators, that is a signal it should be
extracted into its own unit.

---

## Structure

Each test file has one top-level `describe` named after the unit, then one nested
`describe` per operator, named after the operator. Each `it` covers one behavior.

```ts
describe('User', () => {
  describe('updateEmail', () => {
    it('returns the user with the new email', () => { ... })
    it('fails when the user is suspended', () => { ... })
  })

  describe('getFullName', () => {
    it('joins the first and last name with a space', () => { ... })
  })
})
```

---

## Test names

An `it` name describes the behavior in the present tense, so that `it` and the name read
as a sentence. Do not start with "should".

```ts
// ✅ DO
it('returns none when the list is empty', () => { ... })
it('fails when the email has no domain', () => { ... })

// ❌ DON'T
it('should return none', () => { ... })   // "should" adds nothing
it('works', () => { ... })                // does not say what behavior
it('updateEmail', () => { ... })          // names the operator, not the behavior
```

---

## Arrange, act, assert

Each test has three parts in this order, separated by a blank line:

1. **Arrange**: build the input.
2. **Act**: call the operator under test, once.
3. **Assert**: check the output.

```ts
it('returns the user with the new email', () => {
  const user = makeUser()
  const email = makeEmail('new@example.com')

  const result = updateEmail(user, email)

  expect(result).toEqual(Result.ok({ ...user, email }))
})
```

---

## No logic in tests

Tests contain no `if`, loops, or `try`/`catch`. When the same behavior needs several
inputs, use `it.each`.

```ts
// ✅ DO
it.each([
  ['missing domain', 'user@'],
  ['missing at sign', 'user.example.com'],
])('fails when the email has a %s', (_, value) => {
  expect(createEmail(value).isErr()).toBe(true)
})

// ❌ DON'T
it('fails for invalid emails', () => {
  for (const value of ['user@', 'user.example.com']) {
    expect(createEmail(value).isErr()).toBe(true)
  }
})
```

---

## Test data

Build test data with the unit's own constructors where possible, so tests never hold a
unit in an invalid state.

When a valid unit is needed in many tests, write a fixture function named `make{Unit}` that
returns a valid unit and accepts overrides. Keep it in the test file. When fixtures are
shared across test files, put them in `{unit-name}.fixture.ts` next to the unit.

```ts
const makeUser = (overrides: Partial<CreateUserParams> = {}): User =>
  createUser({ id: 'user-1', email: 'user@example.com', name: 'Deta', ...overrides }).unwrap()
```

---

## Assert on values

Assert on what an operator returns, not on how it got there. Prefer comparing whole values
with `toEqual` over checking fields one by one. For `Result` and `Option`, assert on the
variant and its contents.

---

## Deterministic

Tests never depend on the current time, randomness, the network, or the order they run in.
In the core, these already arrive as parameters. In the shell, inject them through deps so
tests can pass fixed values.

---

## Mocking

### In the core

Never mock in the core. Use cases receive contracts through their deps, so a test passes
plain functions in their place. No mocking library or module mocking is needed.

```ts
describe('SendWelcomeEmail', () => {
  it('fails when the user does not exist', async () => {
    const sendWelcomeEmail = createSendWelcomeEmail({
      findUserById: async () => Option.none(),
      // ...
    })

    const result = await sendWelcomeEmail({ userId })

    expect(result.isErr()).toBe(true)
  })
})
```

### In the shell

Mocking is allowed when code talks to something that cannot run in a test, such as a
third-party SDK or a browser API, but use it sparingly.

- Prefer passing plain functions through deps, as in the core.
- Mock only at the boundary with the outside world. Never mock the code under test, the
  core, or other units of your own.

---

## Testing framework units

Framework units, such as components and hooks, have their own test rules. Each test file
has one top-level `describe` named after the component or hook.

Reducers are not framework units. They are pure transformers, so test them like any other
unit: give a state and an action, and assert on the returned state and commands.

### Presentational components

Presentational components receive everything through props and render it. Test them like
pure functions: give props as input, and assert on the rendered output and on the `on`
props they call. Do not assert on internal state or on which hooks were called.

```tsx
describe('AuthErrorPanel', () => {
  it('shows the error message', () => {
    render(<AuthErrorPanel error={{ kind: 'invalidCredentials' }} onRetry={() => {}} />)

    expect(screen.getByText('Invalid email or password')).toBeVisible()
  })
})
```

### Container components

Container components connect presentational components to the store and use cases, such as
a page view. They should be almost devoid of logic, so one test case is enough: render it
with its providers and assert that it renders.

### Hooks

Hooks can get complicated, so keep them thin. Extract logic into units as much as possible,
where it is tested as pure operators. What remains in the hook is wiring, which needs only a
few tests.

Effect hooks (`useXEffect`) are tested by passing use cases built from plain functions to
the use case provider, and asserting on the dispatched actions.

If a hook needs many test cases, that is a signal it holds logic that belongs in a unit.

### Server handlers

Handlers are like container components, almost devoid of logic. Test each one by passing
use cases built from plain functions, with one test per kind of result: a parse failure, a
success, and an error. Parsing and error mapping live in DTO units and error extensions,
which are tested as pure operators. See
[architecture-shell-server.md](./architecture-shell-server.md#testing).

---

## Summary

| Rule | Description |
|---|---|
| Coverage | Core and polyfill ideally 100%, shell at least 70% |
| Testing effort | Most at the bottom strata, same target everywhere |
| Placement | `{file-name}.test.ts` next to the file under test |
| Abstraction barrier | Test through exported operators only |
| Structure | `describe` per unit, nested `describe` per operator, `it` per behavior |
| Test names | Present tense behavior, no "should" |
| Arrange, act, assert | Three parts, one act per test |
| No logic | No `if`, loops, or `try`/`catch`, use `it.each` |
| Test data | Built with constructors, shared through `make{Unit}` fixtures |
| Assertions | On returned values, whole values with `toEqual` |
| Deterministic | No real time, randomness, network, or order dependence |
| Mocking | Never in the core, sparingly and only at the boundary in the shell |
| Presentational components | Tested like pure functions, props in, rendered output out |
| Container components | One test case that renders them with their providers |
| Hooks | Kept thin, logic extracted into units, few tests for the wiring |
| Server handlers | One test per kind of result, use cases built from plain functions |
