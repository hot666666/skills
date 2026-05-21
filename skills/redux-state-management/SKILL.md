---
name: redux-state-management
description: Apply Firefox iOS-style Swift Redux state management patterns. Use when adding, refactoring, or reviewing Swift code that defines app or screen state, actions/action types, reducers, middleware side effects, global Store setup, StoreSubscriber view/controller subscriptions, selected substate subscriptions, or unit tests around reducers and middleware.
metadata:
  short-description: Redux state, reducer, middleware patterns
---

# Swift Redux State Management

## Workflow

1. Inspect the existing Redux primitives before editing: `Action`, `ActionType`, `StateType`, `Reducer`, `Middleware`, `Store`, `Subscription`, and `StoreSubscriber`.
2. Model state as `Equatable` value types. Use one app state that delegates to feature or screen substates, and keep transient defaults behind `defaultState(from:)`.
3. Define actions as small structs with `windowUUID`, `actionType`, and only the payload needed by reducers or middleware. Define user/view actions separately from middleware-result actions when that improves traceability.
4. Keep reducers pure and synchronous. Reducers receive `(state, action)` and return a new state. They gate by `windowUUID` where the app can have multiple windows or multiple active screen states.
5. Put side effects in middleware. Middleware is an array of `(State, Action) -> Void` handlers run after reduction for each dispatched action. Treat it as action/state-triggered side-effect handlers, not as a chain that transforms the action.
6. Subscribe UI using `StoreSubscriber` and `store.subscribe(_:transform:)`. Select the narrow substate needed by the screen or view so unchanged unrelated app state does not trigger updates.
7. On screen/component lifetime, dispatch add/remove component actions if the app tracks active screen state.
8. Test reducers with direct reducer calls. Test middleware with injected dependencies and a replaceable or mock store.

## Implementation Rules

- Prefer local feature folders with `FeatureAction.swift`, `FeatureState.swift`, and `FeatureMiddleware.swift` where the codebase already uses that shape.
- Prefer immutable state plus copy/update helpers over mutating state in place.
- Keep reducer switch statements exhaustive over meaningful action types and return a passthrough/default state for unrelated actions.
- In reducers, validate action payload casts before reading optional payload fields.
- In middleware, inject dependencies through the initializer and provide production defaults. Dispatch follow-up actions only after side effects complete.
- Keep store dispatches on the main actor when the store is main-actor isolated.
- Avoid putting network, storage, telemetry, or service calls in reducers.

## Reference

Read `references/patterns.md` when implementing or reviewing a concrete feature. It contains compact examples for Store execution order, selected subscriptions, screen state, middleware, and tests.
