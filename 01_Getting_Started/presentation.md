# Writing VS Code Extensions — Part 1: Getting Started

**Part 1 — Getting Started: Your First Extension**

> Go from an empty folder to a working "Hello World" command running inside a live Extension Development Host -- the foundations every VS Code extension is built on.

`Scaffold` -> `Code` -> `Run (F5)` -> `Reload`

Scaffold - Manifest - Activate - Debug

---

## Table of Contents

1. [Topics](#topics)
2. [What Is a VS Code Extension?](#what-is-a-vs-code-extension)
3. [Architecture -- the Extension Host](#architecture--the-extension-host)
4. [Prerequisites](#prerequisites)
5. [Scaffolding with Yeoman](#scaffolding-with-yeoman)
6. [Anatomy of the Project](#anatomy-of-the-project)
7. [package.json -- the Manifest](#packagejson--the-manifest)
8. [src/extension.ts](#srcextensionts)
9. [Your First Command](#your-first-command)
10. [Running It -- Press F5](#running-it--press-f5)
11. [The Activation Lifecycle](#the-activation-lifecycle)
12. [Debugging Basics](#debugging-basics)
13. [Summary & Further Reading](#summary--further-reading)

---

## Topics

This part takes you from zero to a running extension. By the end you will have **scaffolded**, **coded**, **run** and **debugged** your first command.

### Concepts & Setup

- What a VS Code extension is
- The Extension Host architecture
- Prerequisites -- Node, npm, git
- Scaffolding with `yo code`

### Anatomy of a Project

- The generated file tree
- `package.json` -- the manifest
- `src/extension.ts` entry point
- `tsconfig.json` & `.vscodeignore`

### Your First Command

- `activate()` & `deactivate()`
- Registering a command
- Disposables & `context.subscriptions`
- Running it with F5

### Lifecycle & Debugging

- Activation events
- Breakpoints & the Debug Console
- `Developer: Reload Window`
- Summary & next steps

---

## What Is a VS Code Extension?

An extension is a **Node.js module** that VS Code loads to add new behaviour. The editor exposes the `vscode` API; your code *contributes* features through it -- almost everything except the core text-editing engine is built this way.

### Commands & UI

- Command Palette actions
- Status bar & notifications
- Tree views & the sidebar
- Quick Picks & input boxes

### Languages

- Syntax highlighting (grammars)
- Completion, hover, diagnostics
- Formatters & code actions
- Language servers (LSP)

### Themes

- Colour themes
- File & product icon themes
- Pure JSON -- no code needed

### Debuggers

- Debug adapters (DAP)
- Custom launch configurations
- Breakpoints for any runtime

### Webviews

- Custom HTML/CSS/JS panels
- Custom editors
- Rich, app-like experiences

### Workspace Tools

- Tasks & terminals
- Source control providers
- File system providers

VS Code itself ships many of its built-in features as extensions -- you are using the *same* API that the core team uses.

---

## Architecture -- the Extension Host

VS Code is an **Electron** app. Your extension does *not* run in the UI; it runs in a separate **Extension Host** -- a Node.js process that talks to the editor over **RPC**.

```
+--- VS Code Window (Electron) ----------------+        +------------------+
|  +---------------------+                      |        |  Your Extension  |
|  |    Main Process     |                      |        |                  |
|  |  window & lifecycle |                      | loads  | activate(context)|
|  +---------------------+                      |------->|                  |
|  +---------------------+     RPC   +--------+ |        | import * as      |
|  |    Renderer (UI)    |<--------->|  Ext   | |        |   vscode from    |
|  |  the editor & DOM   |           |  Host  | |        |   'vscode'       |
|  |  workbench/webviews |           | (Node) | |        +------------------+
|  +---------------------+           +--------+ |
+-----------------------------------------------+
```

The Extension Host is a separate Node.js process that loads the `vscode` module and your extension. It has **NO direct DOM access**.

### Why a Separate Process?

- A slow or crashing extension cannot freeze the UI
- Extensions are sandboxed from the renderer
- Everything is asynchronous -- APIs return Promises (Thenables)

### You Cannot Touch the DOM

Extension code has *no* `document` or `window`. To render custom UI you use a **Webview** (covered in Part 4) and message it over a postMessage bridge -- never reach into the editor's HTML.

---

## Prerequisites

The toolchain is small and all free. Install these before scaffolding your first extension.

### Node.js (LTS)

The Extension Host runs on Node. Use the current LTS (18+). `npm` ships with it.

### Visual Studio Code

The editor itself -- it provides the Extension Development Host and the debugger.

### Git

The generator can initialise a repository for you and is needed for publishing later.

### TypeScript

Recommended (and what this series uses). Installed locally per-project -- no global install needed.

```bash
# Check Node.js and npm are installed
node --version
# v20.11.1

npm --version
# 10.2.4

# Check git
git --version
# git version 2.43.0

# Check VS Code from the command line
code --version
# 1.90.0
```

### The `code` Command

If `code` is not found, open VS Code and run `Shell Command: Install 'code' command in PATH` from the Command Palette.

### Tip

Match your `engines.vscode` manifest field to a version you actually have installed, so the dev host can launch.

---

## Scaffolding with Yeoman

The official generator, `generator-code`, runs through **Yeoman** (`yo`) and creates a ready-to-run project -- manifest, TypeScript config, entry point and a launch task.

```bash
# Install the generator (one-off, global)
npm install -g yo generator-code

# Run it in an empty folder
yo code
```

```text
? What type of extension do you want to create?
> New Extension (TypeScript)
  New Extension (JavaScript)
  New Color Theme
  New Language Support
  New Code Snippets
  New Keymap
  New Extension Pack
  New Web Extension (TypeScript)

? What's the name of your extension? Hello World
? What's the identifier? hello-world
? What's the description?
? Initialize a git repository? Yes
? Bundle the source code with webpack? No
? Which package manager to use? npm
```

### What You Get

- A working "Hello World" command out of the box
- `package.json` manifest pre-filled
- `src/extension.ts` with `activate` / `deactivate`
- `.vscode/launch.json` wired for F5
- `npm install` already run for you

### No Global Install?

You can scaffold without installing globally:

```bash
npx --package yo --package generator-code -- yo code
```

### Then Open It

```bash
cd hello-world
code .
```

---

## Anatomy of the Project

```text
hello-world/
|-- .vscode/
|   |-- launch.json      > F5 debug config
|   `-- tasks.json       > build task (tsc)
|-- src/
|   `-- extension.ts     > entry point
|-- node_modules/        > deps (gitignored)
|-- .vscodeignore        > files to exclude from the .vsix
|-- package.json         > the manifest
|-- tsconfig.json        > TS compiler config
|-- .gitignore
`-- README.md            > shown on Marketplace
```

### package.json

The **extension manifest**. Declares metadata, the entry point, activation events and everything your extension contributes.

### src/extension.ts

Your code. Exports `activate()` and optionally `deactivate()`. Compiled to `out/extension.js` by default.

### tsconfig.json & .vscode/

`tsconfig.json` targets a Node-compatible build; `launch.json` and `tasks.json` let F5 compile and launch the dev host.

### .vscodeignore & node_modules

`.vscodeignore` keeps source, tests and dev deps out of the published `.vsix`; `node_modules` is installed locally and gitignored.

---

## package.json -- the Manifest

```json
{
  "name": "hello-world",
  "displayName": "Hello World",
  "description": "My first VS Code extension",
  "publisher": "my-publisher",
  "version": "0.0.1",
  "engines": {
    "vscode": "^1.90.0"
  },
  "categories": ["Other"],
  "main": "./out/extension.js",
  "activationEvents": [],
  "contributes": {
    "commands": [
      {
        "command": "hello-world.helloWorld",
        "title": "Hello World"
      }
    ]
  },
  "scripts": {
    "compile": "tsc -p ./",
    "watch": "tsc -watch -p ./"
  },
  "devDependencies": {
    "@types/vscode": "^1.90.0",
    "typescript": "^5.4.0"
  }
}
```

### Identity

- `name` + `publisher` form the unique ID `publisher.name`
- `displayName` / `description` show on the Marketplace
- `version` -- semantic versioning

### Runtime Contract

- `engines.vscode` -- minimum API version supported
- `main` -- the compiled entry file to load

### Behaviour

- `activationEvents` -- when to wake the extension
- `contributes` -- commands, menus, settings, themes...

### Note

Since VS Code 1.74, an `onCommand` event for each contributed command is *inferred automatically* -- so `activationEvents` can stay empty here.

---

## src/extension.ts

Every extension exports an `activate()` function. VS Code calls it **once**, the first time an activation event fires -- not at startup unless you ask for that.

```typescript
import * as vscode from 'vscode';

// Called once, on first activation event
export function activate(context: vscode.ExtensionContext) {

  console.log('hello-world is now active!');

  // ... register commands, providers, listeners here ...
  // Each registration returns a Disposable.
}

// Called when the extension is deactivated
// (optional — omit if there is nothing to tidy up)
export function deactivate() {
  // close connections, flush state, etc.
}
```

### activate(context)

- Runs once, lazily, on first activation event
- Your set-up entry point
- May return a Promise if set-up is async

### ExtensionContext

- `subscriptions` -- array of disposables to clean up
- `extensionUri` / `extensionPath` -- on-disk location
- `globalState` / `workspaceState` -- persisted storage
- `secrets` -- secure secret storage

### deactivate()

Optional. Use it only for teardown that disposables don't already handle.

---

## Your First Command

A command has two halves: **declared** in the manifest (so it appears in the palette) and **implemented** in code with `registerCommand`. The command ID must match exactly.

```typescript
import * as vscode from 'vscode';

export function activate(context: vscode.ExtensionContext) {

  // The ID must match contributes.commands in package.json
  const disposable = vscode.commands.registerCommand(
    'hello-world.helloWorld',
    () => {
      vscode.window.showInformationMessage('Hello World!');
    }
  );

  // Push the disposable so VS Code disposes it on deactivate
  context.subscriptions.push(disposable);
}

export function deactivate() {}
```

### registerCommand

Binds a command ID to a callback and returns a **Disposable**. Invoked from the palette, a keybinding, a menu, or programmatically via `executeCommand`.

### context.subscriptions

Push every disposable here. When the extension deactivates, VS Code disposes them all -- no manual cleanup, no leaks.

### Common Mistake

Registering the same command ID twice -- or a mismatch between the manifest ID and the code -- throws on activation. Keep them identical.

---

## Running It -- Press F5

Hit F5. VS Code compiles your TypeScript, then launches a second window -- the **Extension Development Host** -- with your extension loaded.

```
Press F5  ->  tsc compiles          ->  Dev Host opens               ->  Run the command
in project    src -> out/extension.js   [Extension Development Host]      Ctrl/Cmd+Shift+P
                                        your extension is loaded          -> "Hello World"
                                        debugger attached                 activate() fires (once)
                                                                          -> "Hello World!" toast
```

### In the Dev Host Window

- Open the Command Palette: `Ctrl/Cmd+Shift+P`
- Type and run **Hello World**
- The notification appears -- your extension ran!

### Two Windows

Edit code in your *original* window; it *runs* in the dev host. The title bar of the dev host is marked `[Extension Development Host]`.

---

## The Activation Lifecycle

Extensions are **activated lazily** for fast startup. An *activation event* tells VS Code when to load and run `activate()`. Declare them in `activationEvents`.

| Event | Fires when... |
|-------|---------------|
| `onCommand:...` | A contributed command is invoked (auto-inferred since 1.74) |
| `onLanguage:...` | A file of that language ID is opened |
| `onStartupFinished` | Shortly after the window finishes loading |
| `workspaceContains:...` | The workspace has a file matching a glob |
| `onView:...` | A contributed tree view is expanded |
| `onDebug` | A debug session is about to start |
| `*` | On startup -- avoid; hurts performance |

### Auto-Generated Events

Since VS Code 1.74, events for `contributes` entries (commands, views, languages...) are **inferred automatically**. For a command-only extension, leave `activationEvents` empty.

### Be Lazy on Purpose

- Activate as late as possible -- only when needed
- Prefer specific events over `onStartupFinished`
- Never use `*` unless truly unavoidable

### activate() Runs Once

Whichever event fires *first* triggers a single `activate()` call. Subsequent triggers just invoke the relevant handler.

---

## Debugging Basics

The dev host launches with the **debugger already attached**. You debug an extension exactly like any Node program -- breakpoints, stepping, watches and all.

### Breakpoints

- Click the gutter in `extension.ts`
- Trigger the command in the dev host
- Execution pauses; inspect variables & the call stack
- Conditional & logpoint breakpoints supported

### Debug Console

- `console.log(...)` output appears in your *original* window's Debug Console
- Evaluate expressions live at a breakpoint
- Inspect the `vscode` namespace interactively

### Reload After Changes

- Edit code, then in the dev host run `Developer: Reload Window` (`Ctrl/Cmd+R`)
- Or use the green *Restart* control in the debug toolbar
- `watch` task recompiles TS on save

### Watching the Host

- `Developer: Toggle Developer Tools` for the renderer / webviews
- `Developer: Show Running Extensions` -- see activation times & CPU
- Use the *Extension Host* output channel for host-level logs

### Tip

If a breakpoint is "unbound", check the `watch` task compiled successfully and that source maps are enabled in `tsconfig.json`.

---

## Summary & Further Reading

### Key Takeaways

- An extension is a Node module loaded by the Extension Host
- It runs in a separate process -- no direct DOM access
- `yo code` scaffolds a ready-to-run TypeScript project
- `package.json` is the manifest -- identity, engine, `contributes`
- `activate()` runs once, lazily, on an activation event
- Register commands and push disposables to `context.subscriptions`
- F5 launches the Extension Development Host with the debugger

### Hands-On Exercise

- Scaffold an extension with `yo code`
- Run it with F5 and confirm "Hello World"
- Add a **second command** -- declare it in `contributes.commands` and register it
- Set a breakpoint and inspect `context`

### Further Reading

- `code.visualstudio.com/api` -- the official Extension API docs
- **Your First Extension** -- the get-started walkthrough
- **Extension Anatomy** -- manifest & activation reference
- `@types/vscode` -- the typed API surface
- `microsoft/vscode-extension-samples` on GitHub

### Tooling You'll Meet Next

`@vscode/vsce` (package & publish), `@vscode/test-cli` (testing), the *Contribution Points* reference -- all covered later in the series.

### Next -> Part 2

**Commands & Contribution Points** -- menus, keybindings, the palette, command arguments, `when` clauses and wiring real UI into the editor.

`Part 1 done` -> `Part 2: Contributions`
