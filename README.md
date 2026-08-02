# Agent Skills

This repository collects skills for common iOS workflows:

- UIKit view and view controller coding style
- Swift Redux state management
- XCUITest page object patterns
- Privacy-aware Swift logging systems

## Skills

### `uikit-coding-style`

Use when creating, refactoring, or reviewing UIKit views, view controllers, themed components, layout code, and screen wiring.

Reference: `skills/uikit-coding-style/references/patterns.md`

### `redux-state-management`

Use when working on app state, actions, reducers, middleware, subscriptions, or state-management tests.

Reference: `skills/redux-state-management/references/patterns.md`

### `xcuitest-page-object-patterns`

Use when building or updating UI tests, selectors, page objects, accessibility identifiers, or screenshot flows.

Reference: `skills/xcuitest-page-object-patterns/references/patterns.md`

### `swift-secure-logging`

Use when designing, implementing, refactoring, or reviewing a centralized Swift logging package, log levels, domain categories, privacy redaction, Release policy, or logger tests.

Reference: `skills/swift-secure-logging/references/patterns.md`

## Install

Interactive install:

```bash
npx skills add hot666666/skills
```

Install one skill:

```bash
npx skills add hot666666/skills --skill uikit-coding-style
```

## References

### External background docs

- Redux: [Mozilla Firefox iOS Redux sources](https://github.com/mozilla-mobile/firefox-ios/tree/main/BrowserKit/Sources/Redux)
- POM: [Firefox iOS UI testing guide - building page object](https://github.com/mozilla-mobile/firefox-ios/wiki/Test-Automation-Efficiency-UI-Testing-Guide-for-Firefox-iOS#building-page-object)
- Logging: [bitchat BitLogger sources](https://github.com/permissionlesstech/bitchat/tree/main/localPackages/BitLogger)
