# Swift Redux Patterns

## Contents

- Mental Model
- Core Types
- Store Delivery Order
- Subscription Pipeline
- Subscription And Select
- Store Subscriber Bridge
- App State Composition
- Screen State Reducer
- Actions
- Middleware
- Store Setup
- UI Subscription
- Select Patterns
- Subscription Lifetime
- New State Handling
- Tests

## Mental Model

The store owns one `StateType & Equatable` app state. `dispatch(_:)` queues actions, reduces the current state, runs each middleware once with the reduced state and original action, then publishes the new state. Middleware does not form a transform chain; it is an ordered list of side-effect handlers that can observe state/action and dispatch follow-up actions.

Subscriptions are observer-style. `Subscription.select` maps app state to a smaller equatable substate, so `newState` only fires when that selected value changes.

## Core Types

```swift
public typealias WindowUUID = String

public protocol ActionType {}

public protocol Action {
    var windowUUID: WindowUUID { get }
    var actionType: any ActionType { get }
    var debugDescription: String { get }
}

public extension Action {
    var debugDescription: String {
        let className = String(describing: Self.self)
        return "<\(className)> Type: \(actionType) Window: \(windowUUID)"
    }
}

@MainActor
protocol StateType: Equatable {
    static func defaultState(from state: Self) -> Self
}

public typealias DispatchFunction = @MainActor (Action) -> Void
public typealias Middleware<State> = @MainActor (State, Action) -> Void
typealias Reducer<State> = @MainActor (State, Action) -> State
```

When a caller only needs to dispatch actions, depend on the minimal dispatch interface instead of the concrete store.

```swift
protocol DispatchStore {
    @MainActor
    func dispatch(_ action: Action)
}

protocol DefaultDispatchStore<State>: DispatchStore where State: StateType {
    associatedtype State

    @MainActor
    var state: State { get }

    @MainActor
    func subscribe<S: StoreSubscriber>(_ subscriber: S) where S.SubscriberStateType == State

    @MainActor
    func subscribe<SubState: Equatable, S: StoreSubscriber>(
        _ subscriber: S,
        transform: ((Subscription<State>) -> Subscription<SubState>)?
    ) where S.SubscriberStateType == SubState

    @MainActor
    func unsubscribe<S: StoreSubscriber>(_ subscriber: S) where S.SubscriberStateType == State

    @MainActor
    func unsubscribe(_ subscriber: any StoreSubscriber)
}
```

## Store Delivery Order

The store is the only mutator of app state. A dispatch goes through this sequence:

1. Append the action to an internal queue.
2. If no action is currently being processed, drain the queue.
3. For each action, compute `newState = reducer(state, action)`.
4. Run each middleware once with `(newState, action)`.
5. Assign `state = newState`.
6. `state.didSet` removes dead weak subscribers and calls each subscription wrapper with `(oldState, newState)`.

The queue prevents a middleware-dispatched action from interleaving with the current reducer/middleware pass. The follow-up action runs after the current action has completed reduction, middleware observation, and publication.

## Subscription Pipeline

This Redux implementation uses an observer pattern with a wrapper around each subscriber. The wrapper owns the original app-state subscription and holds the subscriber weakly. When a transformed substate subscription is used, it is intentionally not stored as a wrapper property; it stays alive through the `originalSubscription.observer -> sink -> transformedSubscription` retain chain created by `Subscription.select`.

```swift
@MainActor
final class SubscriptionWrapper<State: Equatable>: Hashable {
    private let originalSubscription: Subscription<State>
    weak var subscriber: AnyStoreSubscriber?
    private let objectIdentifier: ObjectIdentifier

    init<T: Equatable>(
        originalSubscription: Subscription<State>,
        transformedSubscription: Subscription<T>?,
        subscriber: AnyStoreSubscriber
    ) {
        self.originalSubscription = originalSubscription
        self.subscriber = subscriber
        self.objectIdentifier = ObjectIdentifier(subscriber)

        if let transformedSubscription {
            transformedSubscription.observer = { [unowned self] _, newState in
                self.subscriber?.newState(state: newState as Any)
            }
        } else {
            originalSubscription.observer = { [unowned self] _, newState in
                self.subscriber?.newState(state: newState as Any)
            }
        }
    }

    func newValues(oldState: State, newState: State) {
        originalSubscription.newValues(oldState: oldState, newState: newState)
    }
}
```

`objectIdentifier` makes one subscriber instance map to one wrapper in the store's `Set`; resubscribing the same object updates the wrapper instead of adding duplicates. The weak subscriber means explicit unsubscribe is useful for removing active screen state, but dead subscribers can still be cleaned during publication.

Do not make the `Subscription.init(sink:)` callback capture `self` weakly. The transformed subscription returned from `select` is retained by the sink callback. Using `[weak self]` there breaks the chain, lets the transformed subscription deallocate immediately after wrapper setup, and prevents selected-substate subscribers from receiving updates.

## Subscription And Select

`Subscription` suppresses notifications when the observed value is unchanged. `select` wires the original app-state observer to a derived `Subscription<Substate>`.

```swift
@MainActor
final class Subscription<State: Equatable> {
    var observer: (@MainActor (State?, State) -> Void)?

    init() {}

    init(sink: @escaping (@MainActor @escaping (State?, State) -> Void) -> Void) {
        sink { oldState, newState in
            self.newValues(oldState: oldState, newState: newState)
        }
    }

    func newValues(oldState: State?, newState: State) {
        guard newState != oldState else { return }
        observer?(oldState, newState)
    }

    func select<Substate: Equatable>(
        _ selector: @escaping (State) -> Substate
    ) -> Subscription<Substate> {
        Subscription<Substate> { sink in
            self.observer = { oldState, newState in
                sink(oldState.map(selector), selector(newState))
            }
        }
    }
}
```

The `Substate` must be `Equatable` because `Subscription<Substate>.newValues` compares old and new selected values. This is the optimization point: reducers can run for many active substates, but a subscriber only receives `newState` when its selected value changes.

## Store Subscriber Bridge

The typed subscriber protocol is bridged through `AnyStoreSubscriber` so a wrapper can store heterogeneous subscribers and still call typed `newState`.

```swift
public protocol AnyStoreSubscriber: AnyObject {
    @MainActor
    func subscribeToRedux()

    @MainActor
    func newState(state: Any)
}

public protocol StoreSubscriber: AnyStoreSubscriber {
    associatedtype SubscriberStateType

    @MainActor
    func newState(state: SubscriberStateType)
}

extension StoreSubscriber {
    @MainActor
    public func newState(state: Any) {
        if let typedState = state as? SubscriberStateType {
            newState(state: typedState)
        }
    }
}
```

The selected substate type from `store.subscribe(_:transform:)` must match `SubscriberStateType`; otherwise the bridge drops the update.

## App State Composition

```swift
struct AppState: StateType, Sendable {
    let presentedComponents: PresentedComponentsState

    static let reducer: Reducer<Self> = { state, action in
        AppState(
            presentedComponents: PresentedComponentsState.reducer(
                state.presentedComponents,
                action
            )
        )
    }

    static func defaultState(from state: AppState) -> AppState {
        AppState(presentedComponents: state.presentedComponents)
    }
}
```

Use a helper such as `componentState(_:for:window:)` when a screen state is stored inside a heterogeneous active-component collection. Filter by `windowUUID` unless the action is intentionally global.

## Screen State Reducer

```swift
struct FeatureState: ScreenState, Equatable {
    var windowUUID: WindowUUID
    let childState: ChildState
    let isLoading: Bool

    init(windowUUID: WindowUUID) {
        self.windowUUID = windowUUID
        self.childState = ChildState(windowUUID: windowUUID)
        self.isLoading = false
    }

    static let reducer: Reducer<Self> = { state, action in
        guard action.windowUUID == .unavailable || action.windowUUID == state.windowUUID else {
            return passthroughState(from: state, action: action)
        }

        switch action.actionType {
        case FeatureActionType.initialize:
            return state.copyWithUpdates(
                childState: ChildState.reducer(state.childState, action),
                isLoading: true
            )
        case FeatureMiddlewareActionType.didLoad:
            guard let action = action as? FeatureAction else {
                return defaultState(from: state)
            }
            return state.copyWithUpdates(
                childState: ChildState.reducer(state.childState, action),
                isLoading: false
            )
        default:
            return passthroughState(from: state, action: action)
        }
    }

    private static func passthroughState(from state: FeatureState, action: Action) -> FeatureState {
        state.copyWithUpdates(
            childState: ChildState.reducer(state.childState, action)
        )
    }

    static func defaultState(from state: FeatureState) -> FeatureState {
        state.copyWithUpdates(
            childState: ChildState.defaultState(from: state.childState),
            isLoading: false
        )
    }
}
```

## Actions

```swift
struct FeatureAction: Action {
    let windowUUID: WindowUUID
    let actionType: ActionType
    let items: [FeatureItem]?

    init(
        items: [FeatureItem]? = nil,
        windowUUID: WindowUUID,
        actionType: ActionType
    ) {
        self.items = items
        self.windowUUID = windowUUID
        self.actionType = actionType
    }
}

enum FeatureActionType: ActionType {
    case initialize
    case didTapItem
}

enum FeatureMiddlewareActionType: ActionType {
    case didLoad
}
```

## Middleware

```swift
@MainActor
final class FeatureMiddleware {
    private let service: FeatureServiceProtocol
    private let logger: Logger

    init(
        service: FeatureServiceProtocol = AppContainer.shared.resolve(),
        logger: Logger = DefaultLogger.shared
    ) {
        self.service = service
        self.logger = logger
    }

    lazy var featureProvider: Middleware<AppState> = { state, action in
        switch action.actionType {
        case FeatureActionType.initialize:
            Task { @MainActor in
                await self.loadFeatureData(windowUUID: action.windowUUID)
            }
        default:
            break
        }
    }

    private func loadFeatureData(windowUUID: WindowUUID) async {
        do {
            let items = try await service.loadItems()
            store.dispatch(
                FeatureAction(
                    items: items,
                    windowUUID: windowUUID,
                    actionType: FeatureMiddlewareActionType.didLoad
                )
            )
        } catch {
            logger.log("Unable to load feature data", level: .warning, category: .redux)
        }
    }
}
```

## Store Setup

```swift
@MainActor
let middlewares: [Middleware<AppState>] = [
    FeatureMiddleware().featureProvider,
    OtherMiddleware().otherProvider
]

@MainActor
let store: any DefaultDispatchStore<AppState> = Store(
    initialState: AppState(),
    reducer: AppState.reducer,
    middlewares: middlewares
)
```

For test builds, keep the store replaceable so tests can inject an empty-middleware store or a mock dispatch store.

## UI Subscription

```swift
final class FeatureViewController: UIViewController, StoreSubscriber {
    typealias SubscriberStateType = FeatureState

    private let windowUUID: WindowUUID
    private var featureState: FeatureState

    func subscribeToRedux() {
        store.dispatch(ComponentAction(
            windowUUID: windowUUID,
            actionType: ComponentActionType.addComponent,
            component: .feature
        ))

        let uuid = windowUUID
        store.subscribe(self, transform: {
            $0.select { appState in
                FeatureState(appState: appState, uuid: uuid)
            }
        })
    }

    func newState(state: FeatureState) {
        guard featureState != state else { return }
        featureState = state
        configure(with: state)
    }

    func unsubscribeFromRedux() {
        store.dispatch(ComponentAction(
            windowUUID: windowUUID,
            actionType: ComponentActionType.removeComponent,
            component: .feature
        ))
    }
}
```

## Select Patterns

Select a screen state from app state:

```swift
typealias SubscriberStateType = FeatureState

func subscribeToRedux() {
    store.dispatch(ComponentAction(
        windowUUID: windowUUID,
        actionType: ComponentActionType.addComponent,
        component: .feature
    ))

    let uuid = windowUUID
    store.subscribe(self, transform: {
        $0.select { appState in
            FeatureState(appState: appState, uuid: uuid)
        }
    })
}
```

Select a nested UI substate when a view only needs one section:

```swift
typealias SubscriberStateType = FeatureHeaderState

func subscribeToRedux() {
    let uuid = windowUUID
    store.subscribe(self, transform: {
        $0.select { appState in
            FeatureState(appState: appState, uuid: uuid).headerState
        }
    })
}
```

Select a small composite state when the view depends on a few values:

```swift
struct FeatureToolbarViewState: Equatable {
    let title: String
    let isLoading: Bool
    let canSubmit: Bool
}

typealias SubscriberStateType = FeatureToolbarViewState

func subscribeToRedux() {
    let uuid = windowUUID
    store.subscribe(self, transform: {
        $0.select { appState in
            let state = FeatureState(appState: appState, uuid: uuid)
            return FeatureToolbarViewState(
                title: state.title,
                isLoading: state.isLoading,
                canSubmit: state.formState.canSubmit
            )
        }
    })
}
```

Subscribe to the full app state only for infrastructure-level observers that truly need it:

```swift
typealias SubscriberStateType = AppState

func subscribeToRedux() {
    store.subscribe(self)
}
```

## Subscription Lifetime

For screen/component state, subscribe and dispatch add-component together, then dispatch remove-component when the screen is done:

```swift
func unsubscribeFromRedux() {
    store.dispatch(ComponentAction(
        windowUUID: windowUUID,
        actionType: ComponentActionType.removeComponent,
        component: .feature
    ))
    store.unsubscribe(self)
}
```

If the local implementation documents weak subscribers and does not require explicit unsubscribe for memory cleanup, still prefer explicit unsubscribe when it changes app state or when the subscriber can be recreated frequently.

## New State Handling

Keep `newState` small and diff-aware. The subscription already filters by equality of the selected state, but a local guard is still useful when the method performs expensive UI work after also touching unrelated state.

```swift
func newState(state: FeatureState) {
    guard featureState != state else { return }
    featureState = state
    contentView.configure(state: state.contentState)
    updateActionButton(isEnabled: state.canSubmit)
}
```

When a selected state is intentionally broad, compare the expensive child state before reloading tables or collection snapshots.

## Tests

Reducer tests should call the reducer directly:

```swift
func test_didLoad_setsItemsAndStopsLoading() {
    let reducer = FeatureState.reducer
    let initialState = FeatureState(windowUUID: windowUUID)
    let action = FeatureAction(
        items: [.fixture()],
        windowUUID: windowUUID,
        actionType: FeatureMiddlewareActionType.didLoad
    )

    let result = reducer(initialState, action)

    XCTAssertEqual(result.items, [.fixture()])
    XCTAssertFalse(result.isLoading)
}
```

Middleware tests should inject mock services and a mock or replaceable store, then assert dispatched follow-up actions and side effects. Reset the global store after each test if the project uses a global store.
