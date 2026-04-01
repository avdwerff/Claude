# Code Hygiene Reference

Final review checklist. Apply after functional review is complete.

---

## Secrets and Security

- Never include API keys, tokens, or secrets in the repository — use `.xcconfig`, environment variables, or a secrets manager
- Never use `@AppStorage` for usernames, passwords, or sensitive data — use the Keychain
- Check for accidental `print()` statements that log sensitive data

---

## Access Control

- Default to the most restrictive access level (`private` > `fileprivate` > `internal`)
- Mark `final` on classes that are not designed for subclassing
- ViewModel state exposed to views should be `private(set)`

---

## Swift Correctness

- No force unwraps (`!`) except where a crash is the correct failure mode — add a comment explaining why
- No force casts (`as!`) — use `as?` with graceful fallback
- Avoid `@discardableResult` unless the return value is genuinely optional to handle
- Mark async entry points with `@MainActor` where UI updates occur
- Check for retain cycles in closures — use `[weak self]` where appropriate

---

## Code Clarity

- Comments and documentation (`///`) should be present where logic is not self-evident
- No magic numbers or strings — extract to named constants or enums
- TODO/FIXME comments must reference an issue number: `// TODO: #42 — remove after migration`

---

## Testing

- Unit tests must exist for core application logic
- UI tests only where unit tests are not possible
- Test method names should read as sentences: `func test_loadItems_setsIsLoadingFalse()`

---

## Localization

- If the project uses `Localizable.xcstrings`, add user-facing strings using symbol keys (e.g. `"helloWorld"`) with `extractionState` set to `"manual"`
- Access strings via generated symbols: `Text(.helloWorld)`, never via raw string literals
- Offer to translate new keys into all languages supported by the project

---

## Tooling

- If SwiftLint is configured, resolve all warnings and errors before finishing
- If the Xcode MCP is available, prefer its tools:
  - `RenderPreview` — capture SwiftUI preview screenshots for visual verification
  - `BuildProject` — verify the build is clean after changes
  - `RunAllTests` — confirm no regressions
  - `DocumentationSearch` — look up latest Apple API usage before assuming behavior
