# Flutter Project Collaboration Guidelines

## Purpose

This document guides agents collaborating on Flutter projects. The user is still becoming familiar with Flutter, so the priority is clear, executable, and verifiable code. Do not introduce complex architecture or a large dependency set while the requirements remain small.

## Language and Communication

- Respond, write documentation, code comments, log explanations, and commit messages in Traditional Chinese by default, using Taiwan terminology.
- Programming language keywords, Flutter widgets, package names, file names, and necessary technical terms may remain in English.
- Lead with the completed outcome. Then briefly explain important decisions, scope of impact, verification performed, and unresolved risks.
- When information is incomplete, explicitly label assumptions and identify what is missing. Do not invent requirements or data.
- Do not use emoji or dashes as emphasis.

## Before Starting Work

1. Read the project-root `README.md`, `pubspec.yaml`, this document, and code and tests directly related to the request.
2. Use `rg --files` and `rg` to locate files. Do not scan large unrelated directories, generated outputs, or dependency caches.
3. Check the working tree first. Existing uncommitted changes belong to the user; do not overwrite, revert, or format unrelated files unless the request explicitly includes them.
4. Confirm the existing conventions for state management, routing, storage, themes, and testing before deciding how to change the project.
5. If the project does not exist yet, or the request only concerns planning, do not create a complete Flutter project, add dependencies, or implement screens without explicit authorization.

## Private Specification Workspace

- Store all product planning, requirements, design, task, and other SDD documents in `$PRIVATE_SPEC_WORKSPACE/<repository>/.specs/`.
- `<repository>` must be the directory name of the current Git repository. Do not guess another project name or share a specification directory across projects.
- Do not create, modify, or commit `.specs/` inside the code repository unless the user explicitly overrides this rule.
- Before implementing a request, read only the private specifications directly relevant to that request. Do not scan the entire private specification workspace.
- If `$PRIVATE_SPEC_WORKSPACE` is unset, unreadable, or the target repository cannot be determined, report the cause and ask the user for direction. Do not silently write specifications into the code repository instead.
- Files in the private specification workspace must still follow the minimal-change principle. Do not overwrite planning content for another product or request.

## Development Principles

- Prefer Flutter and Dart standard capabilities. Before adding a third-party package, confirm that existing dependencies cannot already solve the problem.
- Before adding a package, explain its purpose, alternatives, maintenance status, and project impact. Add it only when the user has confirmed the need.
- Build the smallest useful flow first. Extract shared widgets, services, or abstractions only when a clear requirement justifies them.
- Do not preemptively create repositories, services, use cases, generic base classes, or deeply layered directories for hypothetical future needs.
- Keep data flow unidirectional: the UI receives state, user actions invoke explicit events, and updated state rebuilds the necessary UI.
- Every asynchronous operation must handle loading, success, failure, and cancellation or disposed-screen cases.
- Do not perform network requests, data writes, long-lived subscriptions, or state changes inside `build`.
- Do not use `BuildContext` across an asynchronous gap without checking it. Check `context.mounted` when needed.
- Do not use `dynamic`, forced casts, or ignored analyzer warnings to bypass type problems.

## Suggested Incremental Structure

When the product is still small, use a structure that is easy to read and split it further only as features grow:

```text
lib/
  main.dart
  app.dart
  features/
    <feature_name>/
      <feature_name>_page.dart
      <feature_name>_state.dart
      <feature_name>_controller.dart
      widgets/
  shared/
    widgets/
    theme/
    utils/
```

- `main.dart` should only handle application startup and required initialization.
- `app.dart` should assemble `MaterialApp`, themes, and routing.
- Feature pages should assemble the UI. Business rules, data conversion, and longer workflows should live in testable classes within the same feature.
- Move code into `shared/` only when at least two features genuinely use it.
- If the project already has an established structure, follow that convention instead of forcing this template onto it.

## UI and Accessibility

- Follow the existing design system, colors, typography, and spacing. Do not introduce an unrelated visual style.
- For older users, caregivers, or information-dense flows, prioritize clear copy, adequate touch targets, explicit status, and understandable error messages.
- Do not communicate success, failure, warnings, or selection through color alone. Pair color with text, icons, or another distinguishable signal.
- Forms must show input rules, provide immediate but non-disruptive validation, and recover gracefully from errors.
- State-changing actions such as saving, deleting, syncing, and exporting must show in-progress, success, and failure feedback.
- For irreversible or high-impact actions, clearly explain the impact and require confirmation first.

## Data, Privacy, and Security

- Do not hardcode API keys, passwords, tokens, user identifiers, or health data.
- Collect, read, and retain only the data needed to fulfill the requirement.
- Request photo, health-data, notification, and file permissions only when the relevant feature actually needs them, and explain their purpose to the user.
- Do not log sensitive content to the console, analytics services, or error messages readable by others.
- Validate the format and range of external input, deep links, files, and database content.
- Do not upload, synchronize, or share local data with external services without an explicit requirement.

## Code Quality

- Follow Dart naming conventions: use `snake_case` for file names, `PascalCase` for classes, and `camelCase` for variables and methods.
- Prefer `const` constructors and immutable data. Use mutable state only where required.
- Keep widgets focused. Extract a small, named widget when a `build` method becomes difficult to read or repeats itself.
- Comments should explain why a choice was made, not narrate what the code already says.
- Error messages, empty states, and logs must use Traditional Chinese and offer a user-actionable next step.
- Do not use mutable global state as a shortcut between features. First determine the state lifecycle and its owner.

## Verification Workflow

After every code change, run checks appropriate to the scope:

1. Run `dart format` on affected Dart files.
2. Run `flutter analyze`; do not introduce analyzer errors or unresolved warnings.
3. Run relevant `flutter test` suites. When data processing or state logic changes, prioritize unit tests.
4. When widgets, forms, loading states, or error states change, add or update widget tests.
5. Use a device or simulator only for user-visible flows. Cover normal, empty-data, loading, failure, and retry cases at minimum.
6. Finally run `git diff --check` to catch whitespace and newline problems.

If verification cannot run because of the environment, SDK, simulator, or dependencies, clearly report the command not run, the cause, and the user-actionable next step.

## Changes and Version Control

- Modify only files directly related to the request.
- Do not run `git reset --hard`, `git checkout --`, recursive deletion, or other hard-to-recover operations unless the user explicitly requests them.
- Write commit messages in Traditional Chinese and describe user-visible changes, for example: `新增每日飲水紀錄`.
- When creating a branch, use the `codex/` prefix by default unless the user requests another naming convention.
- Do not push, open a pull request, publish a package, or modify production configuration unless the user asks.

## Response Format

When work is complete, respond with:

1. Status: complete, partially complete, or blocked.
2. Changes: the key files and behaviors, stated in terms the user can understand.
3. Verification: commands run and their results. Explain any verification that was not run.
4. Risks or decisions needed: include only matters that affect the next decision.
