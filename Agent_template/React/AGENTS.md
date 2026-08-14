# Opscenter Frontend Rules

This file supplements the repository-root `AGENTS.md` and applies only to `opscenter-frontend/`.

## Stack and Structure

- Follow existing React, TypeScript, Vite, and Material UI patterns. Inspect the feature's page, components, hooks, API, and types before editing.
- Keep feature code in `src/features/<feature>/`; do not move feature-specific logic into global shared folders.
- Do not add state-management, UI, chart, drag-and-drop, or test dependencies without checking existing packages and bundle impact.
- Preserve type safety. Do not use `any`, broad casts, or ignored type errors as workarounds.

## API and Data

- Use the existing API client, schemas, and types for all requests and responses. Do not duplicate contracts or parse API data ad hoc in pages.
- Render backend-owned business metrics, report definitions, and authorization results. The frontend handles presentation and input only; it must not recalculate core business metrics.
- Provide explicit loading, empty, error, and partial-result states. Never present failed requests as empty or successful data.
- For API contract changes, update backend code, OpenAPI, frontend schemas/types, and specs as required by the root rules.

## Copy, Theme, and Accessibility

- Update `src/locales/zh-TW/`, `src/locales/zh-CN/`, and `src/locales/en/` for every user-visible string. Do not hardcode new UI copy.
- Reuse existing theme tokens and Material UI patterns. Do not introduce local colors that break light, dark, or glass themes.
- Form controls need labels, validation feedback where applicable, and disabled/loading states. Keep keyboard access and visible focus states.
- Tables, charts, and dense views must handle loading, empty, error, long text, and small-screen scrolling or responsive layouts.

## Verification

- For normal frontend changes, run:

  ```bash
  npm run typecheck
  npm test
  npm run build
  ```

- Narrow verification only for confirmed documentation/copy-only changes that cannot affect types, tests, or builds; report skipped checks and scope.
- Add or update existing Node tests for pure transformations, dates, URL state, report rendering rules, or layout behavior.
- Run `git diff --check` and manually check affected themes and desktop/mobile layouts before handoff.
