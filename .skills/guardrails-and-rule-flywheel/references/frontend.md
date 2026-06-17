# Frontend guardrails (TypeScript / React / ESLint / husky / CI)

The same taxonomy, expressed in a typical TS/React toolchain. Lint rules and CI gates are the frontend equivalent of pre-write/pre-push hooks; husky + lint-staged is the pre-commit layer.

## Feature-boundary enforcement (the import guard)

A doc saying "don't import across features" is a guideline. ESLint rejecting the import is a guardrail. Use `import/no-restricted-paths` (precise, zone-based) or `no-restricted-imports` (quick patterns).

```js
// eslint.config.js (flat config)
import boundaries from "eslint-plugin-import";

export default [
  {
    plugins: { import: boundaries },
    rules: {
      // Zone-based: a feature may not reach into another feature's internals.
      "import/no-restricted-paths": ["error", {
        zones: [
          { target: "./src/features/task", from: "./src/features", except: ["./task"] },
          { target: "./src/features", from: "./src/app" }, // app wires features; features don't import app
        ],
      }],
      // Quick guard: deep imports bypass the feature's public index.
      "no-restricted-imports": ["error", {
        patterns: [{
          group: ["@/features/*/!(index)", "@/features/*/*"],
          message: "Import a feature via its public index, not its internals.",
        }],
      }],
    },
  },
];
```

ESLint's message is the structured violation — it names the rule and the fix, the same role the backend arch-guard message plays.

## Accessibility lint (a11y as a structural gate)

```js
import jsxA11y from "eslint-plugin-jsx-a11y";

export default [
  { plugins: { "jsx-a11y": jsxA11y }, rules: { ...jsxA11y.configs.recommended.rules } },
];
```
Missing `alt`, label-less inputs, and click handlers on non-interactive elements become errors, not review comments an agent forgets next session.

## Pre-commit layer: husky + lint-staged

Run checks on staged files before the commit exists — the frontend pre-commit hook.

```jsonc
// package.json
{
  "scripts": { "prepare": "husky" },
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --max-warnings=0", "prettier --write"]
  }
}
```
```bash
# .husky/pre-commit
npx lint-staged
```

## CI gates (merge-blocking)

`tsc --noEmit` is the type-check gate; it catches the contract drift a unit test won't. Lint (incl. boundaries + a11y) and tests round out the wall. All are required status checks on the protected `main` branch.

```yaml
# .github/workflows/ci.yml
- run: npm run type-check     # tsc --noEmit — no emit, just verify types
- run: npm run lint           # eslint . --max-warnings=0 (boundaries + jsx-a11y)
- run: npm test               # vitest run
```

Branch protection on `main`: no direct pushes, PR review required before merge, these checks must pass. That is "main is sacred" expressed in repo settings.

## Suppression hygiene

Frontend suppressions (`eslint-disable`, `@ts-ignore`/`@ts-expect-error`) follow the same rule as `# nocheck`: visible, justified, tracked.

```ts
// eslint-disable-next-line no-restricted-imports -- legacy bridge, remove when #128 closes
import { internal } from "@/features/billing/internal";
```
```bash
# CI/review gate: reject suppressions with no issue reference
! grep -rnE 'eslint-disable|@ts-ignore' src/ | grep -vE '#[0-9]+'
```
Prefer `@ts-expect-error` over `@ts-ignore`: it fails the build once the underlying problem is fixed, so the suppression can't silently outlive its reason.

## The flywheel
When the same frontend defect lands in review twice — a forbidden cross-feature import, a missing `alt`, an unhandled loading state — don't leave another comment. Add the ESLint rule (or a custom rule / test) so it fires on every subsequent PR for every agent. Reviews get cheaper; the human shifts from catching problems to improving the system. Run the review step itself with `superpowers:requesting-code-review`.
