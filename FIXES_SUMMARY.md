# Bug/vulnerability fix pass — summary

## Update: real crash reproduced and fixed at the root cause

A test run of the fixed repo surfaced a real, previously-undiagnosed bug
chain:

1. `cd src && npm install` (the exact command in `README.md` and
   `CONTRIBUTING.md`) failed with `ERESOLVE`: `xterm-addon-ligatures@0.5.1`
   declares a peer dependency on `xterm@^4.0.0`, but the project uses
   `xterm@5.3.0`. Because that install failed, `src/node_modules` was left
   empty.
2. `npm start` then crashed with `Cannot find module
   '@electron/remote/main'` — not because the import or the package is
   broken, but because nothing had actually been installed into
   `src/node_modules` in step 1. Verified this by downloading
   `@electron/remote@2.1.3` directly and confirming `main/index.js`
   exists and `require('@electron/remote/main')` resolves correctly once
   the package is actually present.

**Root-caused and fixed properly** (not just papered over with
`--legacy-peer-deps`):
- `xterm-addon-ligatures` 0.5.1 → **0.7.0**, whose peer dependency is
  `xterm@^5.0.0` (matching the project's `xterm@5.3.0` exactly). Verified
  the public API (`LigaturesAddon` class, constructor, `activate()`,
  `dispose()`) is byte-for-byte the same export used in
  `terminal.class.js`.
- A second, previously-hidden conflict was uncovered once the first one
  cleared: `xterm-addon-webgl@^0.11.2` also pins `xterm@^4.0.0`. Bumped
  to **^0.16.0** (first version with `xterm@^5.0.0` is 0.13.0; 0.16.0 is
  latest stable). Verified `WebglAddon`'s constructor, `onContextLoss()`,
  and `dispose()` are unchanged and match `terminal.class.js`'s usage.
- Result: `npm install` (no flags) now completes cleanly in `src/` with
  **0 vulnerabilities**, a single deduped `xterm@5.3.0` across the whole
  tree, and `@electron/remote/main` resolves correctly. Reproduced the
  full flow (fresh `node_modules`, plain `npm install`, resolve check)
  to confirm.
- `README.md`/`CONTRIBUTING.md` needed no changes — the command they
  already document now works as written.

## legacy-src/ removed

Per confirmation, `legacy-src/` has been deleted. It was fully unused
(not referenced by `package.json`, any npm script, `main.js`, or
`_boot.js`) and had a broken JSON file
(`assets/kb_layouts/en-WORKMAN.json`) plus 28 of its own dependency
vulnerabilities that would never have affected the shipping app. Verified
no docs, configs, or scripts reference the directory, and that `src/`
and every other active file were untouched by the removal.

---

Scope note: all fixes below target the active app (`src/`, root config,
`.github/workflows/`).

## Fixed and verified

### Real bugs
- **`src/_renderer.js`** — the welcome banner used a legacy octal escape
  (`"\033[1m"`), which is a hard parse error under strict-mode JS
  (`SyntaxError: Octal escape sequences are not allowed in strict mode`).
  Replaced with the standard hex escape `"\x1b[1m"` — byte-identical
  output (char code 27 either way), verified with `node --check` and a
  strict-mode eval test.
- **`src/package.json` peer-dependency conflicts** — see "Update" above.
- **Dead/broken entry point removed**: `main.js`, `preload.js`,
  `vite.config.js`, `index.html` at the repo root. These were leftovers
  from an abandoned React/Vite rewrite that `CHANGELOG.md` itself
  disavows ("not the React/Vite rewrite path"). `index.html` imported
  `/src/main.jsx`, which does not exist anywhere in the repo — this
  entry point was never functional. Confirmed nothing in `package.json`
  (`main`, `start`, `build-*` scripts) or `src/package.json` referenced
  any of these four files before deleting them.
- **Missing `lang` attribute** on `src/ui.html` (the file the app
  actually loads — `index.html`'s copy already had it).

### Dependency vulnerabilities (via `npm audit`, cross-checked against
the OSV/Grype/Trivy findings in `megalinter-reports/`, all of which flag
the same three `package-lock.json` files)
- **Root `package.json`/`package-lock.json`**: `npm audit fix` — 14
  vulnerabilities (13 high, 1 critical) → **0**. Notable bumps: electron
  41.1.1→41.10.4, tar 7.5.13→7.5.22 (fixes a critical file-smuggling
  advisory), builder-util-runtime, app-builder-lib, js-yaml, form-data,
  undici, tmp, brace-expansion. `package.json` itself untouched (only
  the lockfile's resolved transitive versions changed).
- **`src/package.json`/`src/package-lock.json`**: 6 vulnerabilities (5
  high, 1 critical) → **0**.
  - `nanoid` ^5.1.9→^5.1.16, `systeminformation` ^5.31.5→^5.33.1, `ws`
    ^8.20.0→^8.21.3 — direct dependency bumps, same major version.
  - `brace-expansion`, `tar` — these were transitive-only deps pulled in
    vulnerable versions; pinned via a new `overrides` block instead of
    adding them as direct dependencies (the standard, minimal-footprint
    npm mechanism for this).
  - `pdfjs-dist` ^5.6.205→^6.2.108 (critical: arbitrary JS execution on
    a malicious PDF — relevant here since `docReader.class.js` opens
    user-selected PDF files). This is a major-version bump, so I checked
    it wasn't going to silently break PDF viewing before applying it:
    - `docReader.class.js` only uses the stable core pdf.js API
      (`getDocument`, `GlobalWorkerOptions.workerSrc`, `getPage`,
      `getViewport`, `render` with `canvasContext`) via
      `pdfjs-dist/legacy/build/pdf.mjs` — downloaded the v6.2.108
      tarball and confirmed that entry point and every one of those
      exports are still present, unchanged.
    - v6 requires Node ≥22.13 or ≥24; Electron 41.x (which we just
      bumped to) bundles Node 24.14+, so the engine requirement is
      already satisfied.

### GitHub Actions / CI security (verified by installing and re-running
`zizmor`, `checkov`, and `shellcheck` locally against the fixed files —
not just reading the pre-existing reports)
- Pinned all 8 distinct third-party actions across all 5 workflows to
  commit SHAs (`actions/checkout`, `actions/setup-node`,
  `actions/upload-artifact`, `softprops/action-gh-release`,
  `github/codeql-action/{init,autobuild,analyze}`), resolved from each
  tag's real ref via `git ls-remote`.
- `build-binaries.yaml`: top-level `permissions` narrowed from
  `contents: write` (applied to every job, including linux/windows
  builds that never need it) to `contents: read`, with `write` scoped
  only to the `build-darwin` job that actually publishes the release.
- Added `persist-credentials: false` to every checkout step.
- Disabled implicit npm-cache-on-checkout (`package-manager-cache:
  false`) in the release-build workflow specifically, closing a
  cache-poisoning path where a PR build could poison a cache later
  reused by a tag-triggered release build.
- Added an explicit top-level `permissions: contents: read` to
  `codeql-analysis.yml` and `codeql.yml` (checkov's `CKV2_GHA_1`, which
  wants defense-in-depth even when jobs already scope their own
  permissions).
- `dependabot-auto-merge.yml`: removed the `pull_request_target`
  trigger entirely (zizmor's only remaining high-severity finding,
  "fundamentally insecure trigger"). The workflow's existing `schedule`
  path (every 30 min) already lists and merges *every* open Dependabot
  PR via `gh pr list`, so nothing is lost functionally — Dependabot PRs
  now get approved/merged within 30 minutes instead of instantly,
  without a write-scoped token ever being exposed to a PR-triggered
  event.
- Result: zizmor went from 4 high-severity errors + 1 info to 0 errors
  + 1 info (the remaining note is a style preference — using the
  well-established `softprops/action-gh-release` action vs. raw `gh
  release` in a script — not a security issue, left as-is). Checkov's 2
  failing checks now pass; 198+2 → 200/200 passing.

### CSS (mechanical only, `src/assets/css/*.css`)
- Ran `stylelint --fix` scoped to only spec-equivalent, non-renaming
  rules: `0px`→`0`, legacy `rgba(...)`→modern `rgb(... / N%)` notation,
  alpha as percentage, blank-line normalization. Verified by diffing
  every selector list before/after — **zero** class names, IDs, or CSS
  custom properties changed, so no coupling with `src/*.js`
  `querySelector`/`getElementById` calls was touched.
- Did **not** apply the megalinter default stylelint config's
  kebab-case selector-naming rules. That config was never actually
  present in the repo (megalinter fell back to its own strict default);
  the project's real convention is snake_case ids/classes (inherited
  from upstream eDEX-UI) referenced throughout the JS. Renaming
  thousands of selectors to satisfy an unconfigured, opinionated
  default would be high-risk for zero functional benefit — this is a
  linter-config mismatch, not a bug.

## Reviewed and confirmed as false positives (no change made)
- DevSkim's "weak hash (MD5)" flag on `file-icons-match.js` — that's the
  literal string `"md5"` inside a filename-pattern regex (identifying
  checksum files by name), not actual cryptographic use.
- DevSkim's many `setTimeout`/localhost "dangerous function"/"debug
  code" flags — standard boilerplate and a legitimate local server used
  by the terminal/globe features, not debug leftovers. (Several of these
  did disappear for real, though, once the dead `main.js`/`vite.config.js`
  files that also tripped this rule were removed.)

## Not addressed (with reasons)
- **~22 MB of JS Standard style findings** — scanned specifically for
  parse errors (real bugs) rather than style opinions; found exactly
  one (the octal-escape bug, fixed above). The rest is formatting
  preference (quotes, indentation) across the codebase; bulk-applying
  `standard --fix` project-wide is a much bigger, higher-risk diff than
  this pass was scoped for, and wasn't requested as its own goal.
- **cspell spelling warnings, jscpd copy-paste duplication report** —
  cosmetic/structural suggestions, not bugs or vulnerabilities.
- **`softprops/action-gh-release` → raw `gh release` swap** — zizmor's
  one remaining informational note. Left the trusted, well-maintained
  action in place rather than hand-rolling release upload logic for a
  purely stylistic win.


### Dependency vulnerabilities (via `npm audit`, cross-checked against
the OSV/Grype/Trivy findings in `megalinter-reports/`, all of which flag
the same three `package-lock.json` files)
- **Root `package.json`/`package-lock.json`**: `npm audit fix` — 14
  vulnerabilities (13 high, 1 critical) → **0**. Notable bumps: electron
  41.1.1→41.10.4, tar 7.5.13→7.5.22 (fixes a critical file-smuggling
  advisory), builder-util-runtime, app-builder-lib, js-yaml, form-data,
  undici, tmp, brace-expansion. `package.json` itself untouched (only
  the lockfile's resolved transitive versions changed).
- **`src/package.json`/`src/package-lock.json`**: 6 vulnerabilities (5
  high, 1 critical) → **0**.
  - `nanoid` ^5.1.9→^5.1.16, `systeminformation` ^5.31.5→^5.33.1, `ws`
    ^8.20.0→^8.21.3 — direct dependency bumps, same major version.
  - `brace-expansion`, `tar` — these were transitive-only deps pulled in
    vulnerable versions; pinned via a new `overrides` block instead of
    adding them as direct dependencies (the standard, minimal-footprint
    npm mechanism for this).
  - `pdfjs-dist` ^5.6.205→^6.2.108 (critical: arbitrary JS execution on
    a malicious PDF — relevant here since `docReader.class.js` opens
    user-selected PDF files). This is a major-version bump, so I checked
    it wasn't going to silently break PDF viewing before applying it:
    - `docReader.class.js` only uses the stable core pdf.js API
      (`getDocument`, `GlobalWorkerOptions.workerSrc`, `getPage`,
      `getViewport`, `render` with `canvasContext`) via
      `pdfjs-dist/legacy/build/pdf.mjs` — downloaded the v6.2.108
      tarball and confirmed that entry point and every one of those
      exports are still present, unchanged.
    - v6 requires Node ≥22.13 or ≥24; Electron 41.x (which we just
      bumped to) bundles Node 24.14+, so the engine requirement is
      already satisfied.
  - First attempt at this used `npm install <pkg>@latest`, which
    silently jumped `nanoid` to a new major version (6.x) and added
    `brace-expansion`/`tar` as new direct dependencies — caught this in
    review, reverted, and redid it with pinned versions + `overrides` as
    described above.

### GitHub Actions / CI security (verified by installing and re-running
`zizmor`, `checkov`, and `shellcheck` locally against the fixed files —
not just reading the pre-existing reports)
- Pinned all 8 distinct third-party actions across all 5 workflows to
  commit SHAs (`actions/checkout`, `actions/setup-node`,
  `actions/upload-artifact`, `softprops/action-gh-release`,
  `github/codeql-action/{init,autobuild,analyze}`), resolved from each
  tag's real ref via `git ls-remote`.
- `build-binaries.yaml`: top-level `permissions` narrowed from
  `contents: write` (applied to every job, including linux/windows
  builds that never need it) to `contents: read`, with `write` scoped
  only to the `build-darwin` job that actually publishes the release.
- Added `persist-credentials: false` to every checkout step.
- Disabled implicit npm-cache-on-checkout (`package-manager-cache:
  false`) in the release-build workflow specifically, closing a
  cache-poisoning path where a PR build could poison a cache later
  reused by a tag-triggered release build.
- Added an explicit top-level `permissions: contents: read` to
  `codeql-analysis.yml` and `codeql.yml` (checkov's `CKV2_GHA_1`, which
  wants defense-in-depth even when jobs already scope their own
  permissions).
- `dependabot-auto-merge.yml`: removed the `pull_request_target`
  trigger entirely (zizmor's only remaining high-severity finding,
  "fundamentally insecure trigger"). The workflow's existing `schedule`
  path (every 30 min) already lists and merges *every* open Dependabot
  PR via `gh pr list`, so nothing is lost functionally — Dependabot PRs
  now get approved/merged within 30 minutes instead of instantly,
  without a write-scoped token ever being exposed to a PR-triggered
  event.
- Result: zizmor went from 4 high-severity errors + 1 info to 0 errors
  + 1 info (the remaining note is a style preference — using the
  well-established `softprops/action-gh-release` action vs. raw `gh
  release` in a script — not a security issue, left as-is). Checkov's 2
  failing checks now pass; 198+2 → 200/200 passing.

### CSS (mechanical only, `src/assets/css/*.css`)
- Ran `stylelint --fix` scoped to only spec-equivalent, non-renaming
  rules: `0px`→`0`, legacy `rgba(...)`→modern `rgb(... / N%)` notation,
  alpha as percentage, blank-line normalization. Verified by diffing
  every selector list before/after — **zero** class names, IDs, or CSS
  custom properties changed, so no coupling with `src/*.js`
  `querySelector`/`getElementById` calls was touched.
- Did **not** apply the megalinter default stylelint config's
  kebab-case selector-naming rules. That config was never actually
  present in the repo (megalinter fell back to its own strict default);
  the project's real convention is snake_case ids/classes (inherited
  from upstream eDEX-UI) referenced throughout the JS. Renaming
  thousands of selectors to satisfy an unconfigured, opinionated
  default would be high-risk for zero functional benefit — this is a
  linter-config mismatch, not a bug.

## Reviewed and confirmed as false positives (no change made)
- DevSkim's "weak hash (MD5)" flag on `file-icons-match.js` — that's the
  literal string `"md5"` inside a filename-pattern regex (identifying
  checksum files by name), not actual cryptographic use.
- DevSkim's many `setTimeout`/localhost "dangerous function"/"debug
  code" flags — standard boilerplate and a legitimate local server used
  by the terminal/globe features, not debug leftovers. (Several of these
  did disappear for real, though, once the dead `main.js`/`vite.config.js`
  files that also tripped this rule were removed.)

## Not addressed (with reasons)
- **~22 MB of JS Standard style findings** — scanned specifically for
  parse errors (real bugs) rather than style opinions; found exactly
  one (the octal-escape bug, fixed above). The rest is formatting
  preference (quotes, indentation) across the codebase; bulk-applying
  `standard --fix` project-wide is a much bigger, higher-risk diff than
  this pass was scoped for, and wasn't requested as its own goal.
- **cspell spelling warnings, jscpd copy-paste duplication report** —
  cosmetic/structural suggestions, not bugs or vulnerabilities.
- **`softprops/action-gh-release` → raw `gh release` swap** — zizmor's
  one remaining informational note. Left the trusted, well-maintained
  action in place rather than hand-rolling release upload logic for a
  purely stylistic win.
