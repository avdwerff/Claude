---
name: swift-macos-ios-dev
description: Comprehensively reviews Swift and SwiftUI code for best practices on modern APIs, maintainability, and performance. Use when reading, writing, or reviewing MacOS and iOS Swift projects.
license: MIT
metadata:
  author: Alexander van der Werff
  version: "1.0"
---

Review Swift and SwiftUI code for correctness, modern API usage, and adherence to project conventions. Report only genuine problems - do not nitpick or invent issues.

## Reference Loading Guide

Load reference files when the task matches. Always load before writing or reviewing relevant code.

| Reference | Load When |
|-----------|-----------|
| **[Views](references/views.md)** | Reading, writing, or reviewing any SwiftUI view, layout, modifier, list, sheet, or lifecycle code |
| **[MVVM](references/arch-mvvm.md)** | Project uses MVVM, creating or reviewing ViewModels, navigation, dependency injection |
| **[TCA](references/arch-tca.md)** | Project uses TCA, creating or reviewing Reducers, Features, Effects, TestStore |
| **[Hygiene](references/hygiene.md)** | Final review pass, checking code style, naming and Swift conventions |

## Review Process

1. Determine architecture from project context (MVVM or TCA) and load the relevant reference
2. Check views, modifiers, and animations against `references/views.md`
3. Final code hygiene check using `references/hygiene.md`
