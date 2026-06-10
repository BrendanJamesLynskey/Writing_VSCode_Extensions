# Writing VS Code Extensions — Part 5: Testing, Packaging & Publishing

**Part 5 — Testing, Packaging & Publishing** (Tutorial Series · Part 5 of 5)

> The final part -- take a working extension from local code to the Marketplace and Open VSX, with automated tests and a release pipeline that ships it for you.

`Test` -> `Package (.vsix)` -> `Publish` -> `CI/CD`

Test - Package - Publish - Automate

---

## Table of Contents

1. [Topics](#topics)
2. [The Release Pipeline](#the-release-pipeline)
3. [Testing Setup](#testing-setup)
4. [Writing Tests](#writing-tests)
5. [Running Tests](#running-tests)
6. [Linting & Compiling](#linting--compiling)
7. [Bundling](#bundling)
8. [The .vscodeignore File](#the-vscodeignore-file)
9. [Packaging with vsce](#packaging-with-vsce)
10. [Preparing to Publish](#preparing-to-publish)
11. [Publishing](#publishing)
12. [The Open VSX Registry](#the-open-vsx-registry)
13. [CI/CD with GitHub Actions](#cicd-with-github-actions)
14. [Summary, Best Practices & Series Recap](#summary-best-practices--series-recap)

---

## Topics

### Testing

- The release pipeline overview
- Test setup -- `@vscode/test-cli`
- Writing & running tests
- Linting & type-checking

### Bundling & Packaging

- Bundling with esbuild
- The `.vscodeignore` file
- Building a `.vsix` with vsce
- Required manifest fields

### Publishing

- Marketplace publisher & Azure PAT
- Publishing & version bumps
- The Open VSX Registry
- Dual-publishing strategy

### Automation & Recap

- CI/CD with GitHub Actions
- Release best practices
- The full 5-part series recap
- Where to go next

---

## The Release Pipeline

From source to two registries: each stage gates the next, and **CI/CD** wraps the whole thing so a git tag ships your extension automatically.

```
+--- CI/CD — GitHub Actions (on tag push) -----------------------------------+
|                                                                            |
|  Code -> Lint/Compile -> Test -> Bundle -> Package ----+--> VS Marketplace |
|  src/*.ts                Ext.Host  esbuild  vsce/.vsix |    (vsce publish)  |
|                                                        +--> Open VSX        |
|                                                             (ovsx publish)  |
+----------------------------------------------------------------------------+
   git tag v1.0.0 -> push -> the pipeline runs end-to-end with no manual steps
```

### Key Stages

- **Test** -- Run automated tests inside a real VS Code Extension Host before anything ships.
- **Package** -- Bundle, then build a single `.vsix` archive -- the unit of distribution.
- **Publish** -- Push the same `.vsix` to both the Marketplace and Open VSX for full reach.

---

## Testing Setup

VS Code tests run inside a **real, downloaded copy of VS Code** -- your test code executes in the Extension Host with the full `vscode` API available.

```jsonc
// .vscode-test.mjs
import { defineConfig } from '@vscode/test-cli';

export default defineConfig({
  // glob(s) for your compiled test files
  files: 'out/test/**/*.test.js',
  // which VS Code build to download & run
  version: 'stable',         // or 'insiders', '1.90.0'
  // workspace to open for the tests
  workspaceFolder: './sampleWorkspace',
  mocha: {
    ui: 'tdd',               // suite() / test()
    timeout: 20000,
  },
});
```

```json
// package.json — scripts
{
  "scripts": {
    "compile": "tsc -p ./",
    "watch": "tsc -watch -p ./",
    "pretest": "npm run compile && npm run lint",
    "test": "vscode-test"
  }
}
```

```bash
npm i -D @vscode/test-cli @vscode/test-electron mocha @types/mocha
```

- **`@vscode/test-cli`** -- the test runner; reads `.vscode-test.mjs`, drives Mocha, and exposes the `vscode-test` command.
- **`@vscode/test-electron`** -- downloads the requested VS Code build and launches it with your extension loaded.
- **Why a real VS Code?** -- the API only exists inside the host; tests exercise commands, editors and the workspace exactly as users do.

---

## Writing Tests

Import `* as vscode` and `* as assert`, then drive the real API -- open documents, run your commands, and assert on the result.

```typescript
// src/test/extension.test.ts
import * as assert from 'assert';
import * as vscode from 'vscode';

suite('My Extension Test Suite', () => {

  test('command is registered', async () => {
    const cmds = await vscode.commands.getCommands(true);
    assert.ok(cmds.includes('myExt.helloWorld'));
  });

  test('command shows a message', async () => {
    // run the command the user would invoke
    await vscode.commands.executeCommand('myExt.helloWorld');
    // (assert on side-effects / state here)
  });

  test('opens and edits a document', async () => {
    const doc = await vscode.workspace.openTextDocument({
      content: 'hello',
      language: 'plaintext',
    });
    const editor = await vscode.window.showTextDocument(doc);
    await editor.edit(b =>
      b.insert(new vscode.Position(0, 5), ' world'));
    assert.strictEqual(doc.getText(), 'hello world');
  });
});
```

### Anatomy

- `suite()` groups related tests
- `test()` is one case (Mocha TDD UI)
- `suiteSetup` / `teardown` for fixtures
- Async/await throughout -- the API is promise-based

### What to assert

- Commands are contributed & runnable
- Edits land in the document
- Diagnostics / completions appear
- Configuration is read correctly

**Tip:** Use `vscode.extensions.getExtension(id)?.activate()` if a test needs activation before the API is ready.

---

## Running Tests

```bash
# Local run — compiles, lints, then tests
npm test

# Or invoke the runner directly
npx vscode-test

# Run against a specific build / label
npx vscode-test --label unit

# On Linux CI there is no display server,
# so wrap the runner in a virtual one:
xvfb-run -a npm test
```

### Headless on CI

VS Code is an Electron app and needs a display. On Linux runners use `xvfb-run` to provide a virtual framebuffer. macOS and Windows runners need no wrapper.

### Integration tests

- Run inside the Extension Host
- Full `vscode` API available
- Slower -- download + launch VS Code
- Best for command / editor behaviour

### Unit tests

- Plain Mocha/Jest -- no VS Code
- Import pure helper modules directly
- Fast; mock the `vscode` module
- Best for parsers, utils, business logic

**Split your code:** keep logic out of `activate()` in pure modules -- you can then unit-test most of it without launching VS Code, and reserve integration tests for the API surface.

---

## Linting & Compiling

Catch problems before they ship: **lint** for style and common bugs, **type-check** for correctness. Wire both into `pretest` so `npm test` always validates first.

```json
// package.json — scripts
{
  "scripts": {
    "lint": "eslint src --ext ts",
    "check-types": "tsc --noEmit",
    "compile": "tsc -p ./",
    "pretest": "npm run compile && npm run lint",
    "test": "vscode-test"
  }
}
```

```javascript
// eslint.config.mjs (flat config)
import tseslint from 'typescript-eslint';

export default tseslint.config(
  ...tseslint.configs.recommended,
  {
    rules: {
      'no-console': 'warn',
      '@typescript-eslint/no-unused-vars': 'error',
    },
  },
);
```

- **`eslint`** -- lints TypeScript via `typescript-eslint`; the generator scaffolds a flat `eslint.config.mjs` for you.
- **`tsc --noEmit`** -- type-checks without producing output; fast, perfect for CI when a bundler (not `tsc`) emits the JS.
- **`pretest` hook** -- npm runs any `pre<script>` automatically; `pretest` compiles & lints, so a failing lint fails the test run.
- **Gate the release** -- run the same `lint` + `check-types` in CI before packaging; never publish code that doesn't type-check.

---

## Bundling

Bundle your extension into **a single file** before packaging. Fewer files means a smaller `.vsix`, faster install, and faster activation.

```bash
# One-off production bundle with esbuild
esbuild src/extension.ts \
  --bundle \
  --outfile=dist/extension.js \
  --external:vscode \
  --format=cjs \
  --platform=node \
  --minify
```

```json
// package.json
{
  "main": "./dist/extension.js",
  "scripts": {
    "esbuild-base": "esbuild src/extension.ts --bundle --outfile=dist/extension.js --external:vscode --format=cjs --platform=node",
    "vscode:prepublish": "npm run esbuild-base -- --minify",
    "watch": "npm run esbuild-base -- --sourcemap --watch"
  }
}
```

### Unbundled

Hundreds of files across `node_modules`. Slow to read from disk -- activation drags.

### Bundled

One `dist/extension.js`. Smaller package, fewer file reads, noticeably faster activation.

### Notes

- **Keep `vscode` external** -- the host provides the `vscode` module at runtime; never bundle it. Mark it `--external:vscode`.
- **`vscode:prepublish`** -- `vsce` runs this script automatically before packaging, so the published bundle is always the minified build.
- **esbuild or webpack** -- esbuild is fast and simple; webpack suits complex setups. Either works -- the wins are the same.

---

## The .vscodeignore File

Like `.gitignore`, but for the package: `.vscodeignore` tells **vsce** what to leave out of the `.vsix`. Once you bundle, exclude source, tests, and dev dependencies.

```ini
# .vscodeignore
.vscode/**
.vscode-test/**
.vscode-test.mjs

# source & tests — only ship the bundle
src/**
out/**
**/*.ts
**/*.map

# tooling & config
.gitignore
.eslintrc*
eslint.config.mjs
tsconfig.json
esbuild.js
**/.github/**

# when bundling, drop node_modules entirely
node_modules/**
```

### Why it matters

- Smaller download & faster install for users
- You ship build output, not raw TypeScript
- No test fixtures or CI config in the package

### Bundled? Drop node_modules

If esbuild inlines deps into `dist/extension.js`, you can ignore all of `node_modules` -- the runtime deps are already in the bundle.

### Always keep

- `dist/**` (your bundle)
- `package.json`, `README.md`
- `CHANGELOG.md`, `LICENSE`, icon

**Verify before you ship:** `vsce ls` lists exactly what the package will contain. Run it to confirm nothing private leaks in.

---

## Packaging with vsce

**vsce** (now published as `@vscode/vsce`) is the official packaging & publishing CLI. `vsce package` produces a single `.vsix` archive.

```bash
# Install the CLI globally
npm install -g @vscode/vsce

# Build the package → my-ext-0.0.1.vsix
vsce package

# List what will be included (dry run)
vsce ls

# Install the .vsix locally to test it
code --install-extension my-ext-0.0.1.vsix

# …or in VS Code: Extensions view →
#   "…" menu → "Install from VSIX…"
```

### The .vsix

A zip of your bundle + manifest + assets, named `<name>-<version>.vsix`. It is the unit you install or publish.

### Required manifest fields

- `name`, `displayName`, `version`
- `publisher` -- your Marketplace publisher ID
- `engines.vscode` -- min supported API
- `repository` -- source URL

### Strongly recommended

- `icon` -- 128x128 PNG
- `README.md` -- the Marketplace page
- `LICENSE` -- or vsce warns
- `categories`, `keywords` for discovery

**vsce checks for you:** packaging fails or warns on a missing `repository`, `LICENSE`, or a placeholder README -- fix these before publishing.

---

## Preparing to Publish

Publishing to the Marketplace needs a **publisher** identity and an **Azure DevOps Personal Access Token (PAT)**. This is a one-time setup.

### 1. Create a publisher

In the Visual Studio Marketplace management portal (`marketplace.visualstudio.com/manage`), create a publisher. Its ID becomes the `publisher` field in `package.json`.

### 2. Create an Azure DevOps PAT

- Sign in at `dev.azure.com`
- User settings -> Personal Access Tokens
- Organisation: `All accessible organizations`
- Scope: **Marketplace -> Manage**
- Copy the token -- shown only once

### 3. Log in with vsce

```bash
vsce login <publisher>
# paste the PAT when prompted

# CI uses the PAT non-interactively
export VSCE_PAT=xxxxxxxxxxxxxxxx
vsce publish
```

### Keep the PAT secret

- Never commit it or print it in logs
- Store it as an encrypted CI secret (`VSCE_PAT`)
- Set a short expiry and rotate it regularly
- Use the narrowest scope: Marketplace -> Manage only

**Verify the publisher:** a verified publisher (domain check) earns a badge and builds user trust -- optional but recommended.

---

## Publishing

With a publisher and PAT in place, `vsce publish` packages and uploads in one step. Pass a semver keyword to bump the version, commit and tag automatically.

```bash
# Publish the current version in package.json
vsce publish

# Bump version, then publish (semver):
vsce publish patch   # 1.0.0 → 1.0.1
vsce publish minor   # 1.0.1 → 1.1.0
vsce publish major   # 1.1.0 → 2.0.0

# Or publish an exact version
vsce publish 1.2.3

# Publish a pre-built .vsix
vsce publish --packagePath my-ext-1.2.3.vsix

# CI: authenticate via env, no prompt
vsce publish -p $VSCE_PAT
```

### Semantic Versioning

```
major . minor . patch
breaking  features  fixes

   vsce publish <major|minor|patch>
```

- **major** -- breaking changes
- **minor** -- new features
- **patch** -- bug fixes

**Version bumps do the work:** `vsce publish minor` updates `version` in `package.json`, makes a git commit, and creates a tag -- all before uploading.

**After publishing:** the new version appears on the Marketplace within minutes and auto-updates for installed users. Push your tags with `git push --follow-tags`.

---

## The Open VSX Registry

The Marketplace only serves Microsoft's VS Code build. **Open VSX** is the open registry used by VSCodium, Gitpod, Theia, Eclipse and others -- dual-publish to reach those users too.

```bash
# Install the Open VSX CLI
npm install -g ovsx

# Publish an existing .vsix to Open VSX
npx ovsx publish my-ext-1.2.3.vsix -p <token>

# …or package + publish in one go
npx ovsx publish -p <token>

# In CI, read the token from a secret
npx ovsx publish *.vsix -p $OVSX_PAT
```

### Get a token

- Sign in at `open-vsx.org` (GitHub)
- Create an eclipse.org account & sign the Publisher Agreement
- Generate an access token from your profile
- Create your namespace: `ovsx create-namespace <name>`

### Why dual-publish

- Reach VSCodium, Gitpod, Theia, Eclipse users
- Same `.vsix` -- no separate build
- Open-source-friendly distribution
- Many cloud IDEs default to Open VSX

**Same artefact, two homes:** build the `.vsix` once with `vsce package`, then publish that exact file to both registries. No divergence between them.

**Keep tokens separate:** `VSCE_PAT` for the Marketplace, `OVSX_PAT` for Open VSX. Store both as encrypted CI secrets.

---

## CI/CD with GitHub Actions

Wire it all together: on a version tag, the workflow installs, lints, tests (headless via `xvfb-run`), packages, and publishes to **both** registries from repo secrets.

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    tags: ['v*']

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci

      # lint + type-check + tests
      - run: npm run lint
      - run: xvfb-run -a npm test

      # build the .vsix once
      - run: npm install -g @vscode/vsce ovsx
      - run: vsce package -o extension.vsix

      # publish to the VS Marketplace
      - run: vsce publish -p $VSCE_PAT --packagePath extension.vsix
        env:
          VSCE_PAT: ${{ secrets.VSCE_PAT }}

      # publish the same file to Open VSX
      - run: npx ovsx publish extension.vsix -p $OVSX_PAT
        env:
          OVSX_PAT: ${{ secrets.OVSX_PAT }}
```

### How it runs

- `git tag v1.2.3 && git push --tags` triggers it
- `xvfb-run` gives the Linux runner a display
- Secrets `VSCE_PAT` / `OVSX_PAT` stay encrypted
- One push -> tested, packaged, dual-published

---

## Summary, Best Practices & Series Recap

### Release Best Practices

- Bundle -- small `.vsix`, fast install
- Don't block activation; keep `activate()` lean
- Narrow activation events -- activate only when needed
- Polish the README, icon, categories & keywords
- Keep a `CHANGELOG.md`; follow semver discipline
- Test in the host + unit-test pure logic
- Dual-publish; automate the release on tag

### You can now…

`Build` -> `Test` -> `Ship` (end-to-end)

You can build, test and ship a VS Code extension end-to-end -- from `activate()` to the Marketplace and Open VSX.

### The 5-Part Series

- **Part 1 — Getting Started** -- Scaffolding with Yeoman, the manifest, `activate()`, the dev loop & first command.
- **Part 2 — Commands & Contribution Points** -- Commands, menus, keybindings, configuration & activation events.
- **Part 3 — The Extension API** -- Editors, the workspace, windows, status bar, events & disposables.
- **Part 4 — Webviews & Language Features** -- Webview UIs, completions, hovers, diagnostics & language providers.
- **Part 5 — Testing, Packaging & Publishing** -- Tests, bundling, the `.vsix`, vsce, Open VSX & a CI/CD release pipeline.

### Further Reading

- **Publishing Extensions** -- code.visualstudio.com/api/working-with-extensions/publishing-extension
- **Testing Extensions** -- code.visualstudio.com/api/working-with-extensions/testing-extension
- **Open VSX** -- open-vsx.org · github.com/eclipse/openvsx
