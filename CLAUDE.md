# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- `npm test`: the full check, also what CI runs on Node 22 and 26. It runs `xo` (lint), then `c8 node --test 'test/[!_]*.js'` (tests with coverage), then `tsc --noEmit` on `source/index.d.ts` (type check).
- `npx xo` / `npx xo --fix`: lint only.
- `node --test test/chalk.js`: run one test file.
- `node --test --test-name-pattern='style string' test/chalk.js`: run one test.
- `npm run bench`: benchmarks in `benchmark.js` (uses `node:bench` with `--experimental-bench`).

The package is pure ESM (`"type": "module"`), needs Node >= 22, and has no build step. `source/` is what gets published.

## Architecture

The core is `source/index.js`. It has no runtime dependencies because `ansi-styles` and `supports-color` are vendored into `source/vendor/`. They are imported through the package.json `imports` aliases `#ansi-styles` and `#supports-color`, and `#supports-color` points to `browser.js` outside Node. `source/vendor` is excluded from lint and coverage.

How it works:
- **Instances**: `createChalk()` / `new Chalk(options)` return a plain function (`(...strings) => strings.join(' ')`) whose prototype is `createChalk.prototype`. The `Chalk` class constructor returns that function instead of `this`. The level is stored under the `LEVEL` symbol, and the public `level` accessor validates that it is an integer from 0 to 3.
- **Builders**: every style name (`red`, `bold`, …, plus `visible`) is a lazy getter. On first access it creates a builder through `createBuilder` and caches it on the object with `Object.defineProperty`. A builder is also a function with its prototype swapped to `proto`. It holds:
  - `GENERATOR`: always the root chalk instance, never the immediate parent, so reading the level costs one property load.
  - `STYLER`: a linked list of `{open, close, openAll, closeAll, parent}` with the chained open/close codes precomputed.
  - `IS_EMPTY`: used by `visible`.
- **Color models**: `rgb`, `hex` and `ansi256` (plus their `bg*` and `underline*` forms) are resolved once into an array of converters indexed by level, so the level is read when the function is called rather than when it is created.
- **`applyStyle`**:
  - When the string contains ESC, it re-opens styles after any close codes already in it (needed for nesting).
  - It closes and re-opens styles around `\n`/`\r\n` (needed for a macOS bleed issue).
  - The helpers for both are in `source/utilities.js`.
- `chalkStderr` is a second root instance whose level comes from stderr color support.

This is performance-sensitive hot-path code. Comments in `index.js` explain micro-optimizations: implicit coercion for one argument, a template literal for two, a custom `stringReplaceAll` instead of `String#replaceAll`, and reading `[LEVEL]` directly. Keep these, and the comments, when editing, and check perf-relevant changes with `npm run bench`. Lint-rule suppressions follow the existing style: `// eslint-disable-next-line <rule> -- <reason>`.

`source/index.d.ts` is the hand-written public type definition. Keep it in sync with exports, because `npm test` type-checks it.

## Tests

Tests use `node:test` with `node:assert/strict`. Files starting with `_` are fixtures run as subprocesses through `execa`, not test files. Tests usually set `chalk.level` explicitly at the top, because the test runner sets `FORCE_COLOR` when its output is a TTY. Subprocess tests pass `extendEnv: false` to control color detection.
