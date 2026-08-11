---
name: uikit-coding-style
description: Apply consistent UIKit and AppKit view/controller coding patterns. Use when creating, refactoring, or reviewing Swift UIView, NSView, UIViewController, NSViewController, table or collection cells, themed components, delegate callbacks, coordinator flows, accessibility identifiers, Auto Layout, configure/setup/applyTheme lifecycles, animations, or view-model-driven Apple platform UI code. Do not use for pure SwiftUI views.
---

# UIKit and AppKit View/Controller Style

## Workflow

1. Identify whether the target uses UIKit or AppKit, then inspect nearby files in the same feature area. Match naming, dependency injection, theming, accessibility, lifecycle, and layout helpers already present.
2. Put layout constants in a private `UX` namespace near the top of the type. Keep strings, asset names, and accessibility identifiers in existing centralized constants when available.
3. Declare dependencies and non-UI state before UI elements. Inject services, managers, stores, and callbacks through initializers with production defaults.
4. Build views with lazy private properties and `.build { ... }` when the project provides that helper. Configure static framework properties in the builder block.
5. Use `configure(...)` for applying view models or external state to a reusable view. Use `setup()`, `setupLayout()`, or more specific `setupX` methods for one-time hierarchy and constraint creation.
6. Keep `viewDidLoad` short: call setup/configuration, register listeners, and apply theme. For AppKit, keep `loadView` limited to root-view construction. Keep action handlers and delegate methods small and route business behavior through injected dependencies or actions.
7. Apply theming through `Themeable` for owners that observe theme changes and `ThemeApplicable` for leaf views/cells that receive a theme.
8. Use delegates for reusable child views that report multiple user events, and coordinators/router boundaries for screen transitions.
9. Keep animations local to view state unless they present/dismiss screens; route screen transitions through the existing coordinator/router layer.
10. Add accessibility identifiers when UI test code may need stable selectors. Use UIKit `accessibilityIdentifier` or AppKit `setAccessibilityIdentifier(_:)` according to the framework and prefer existing namespaces.
11. Add or update tests when refactoring behavior, view state, accessibility ids, or view-model driven configuration.

## Implementation Rules

- Prefer `final` for views and view controllers unless subclassing is intentional.
- Prefer `private lazy var` for UI elements that capture `self`; use `private let` when construction does not need `self`.
- Keep Auto Layout programmatic and explicit. Use `addSubview` or `addSubviews`, then one `NSLayoutConstraint.activate` block per logical constraint group.
- In AppKit, set `translatesAutoresizingMaskIntoConstraints = false` explicitly unless the project helper does so, and use `addSubview` rather than UIKit-only hierarchy helpers.
- Store mutable constraints as properties only when they are changed after setup.
- Put `required init?(coder:) { fatalError(...) }` in programmatic UIKit types.
- Split large layout variants into named helpers such as `adjustConstraints()`, `showViewForCurrentOrientation()`, or `setupAccessibility()`.
- Keep comments rare. Preserve existing comments unless directly made stale by the change.

## Reference

Read `references/patterns.md` when implementing a concrete UIKit or AppKit change. Start with its framework-selection notes, then use only the matching framework examples.
