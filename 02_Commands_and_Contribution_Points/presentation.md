# Writing VS Code Extensions

**Part 2 — Commands & Contribution Points**

> How extensions surface functionality in the editor -- declaring contribution points in the manifest and wiring them to runtime code.

`Declare (package.json)` -> `Register (extension.ts)` -> `Invoke`

Declare - Register - Invoke - Gate

---

## Table of Contents

1. [Topics](#topics)
2. [The contributes Map](#the-contributes-map)
3. [The commands Contribution](#the-commands-contribution)
4. [Activation Events Deep Dive](#activation-events-deep-dive)
5. [The Command Palette](#the-command-palette)
6. [Keybindings Contribution](#keybindings-contribution)
7. [Menus Contribution](#menus-contribution)
8. [when Clause Contexts](#when-clause-contexts)
9. [Configuration / Settings Contribution](#configuration--settings-contribution)
10. [Reading Settings in Code](#reading-settings-in-code)
11. [Views & View Containers](#views--view-containers)
12. [Other Contribution Points](#other-contribution-points)
13. [Summary & Further Reading](#summary--further-reading)

---

## Topics

### The contributes Map

- Declarative manifest vs imperative code
- The `commands` contribution
- Activation events deep dive
- The Command Palette

### Bindings & Menus

- Keybindings & chords
- Menus & menu groups
- `when` clause contexts
- Custom contexts via `setContext`

### Settings & Views

- The `configuration` contribution
- Reading settings in code
- Views & view containers (teaser)

### Wrap-Up

- Other contribution points
- Hands-on exercise
- Further reading
- Next: Part 3 -- the Extension API

---

## The contributes Map

Almost everything an extension adds to VS Code is **declared statically** in the `contributes` object of `package.json` -- then **backed at runtime** by code in `extension.ts`.

```
package.json                     VS Code UI
+---------------------+          +------------------+
| "contributes": {    | declare  | Command Palette  |
|   commands          |--------->| Menus & context  |
|   keybindings       |          | Keybindings      |
|   menus             |          | Settings UI      |
|   configuration     |          | Activity Bar     |
|   views             |          +------------------+
| }                   |                  ^ invoke
+---------------------+          +------------------+
        \  register              | extension.ts     |
         \------------------>    | activate(ctx) {  |
                                 |   registerCommand|
                                 |   ('myExt.hello')|
                                 | }                |
                                 +------------------+
```

### Declare

Static JSON in `contributes`. VS Code reads it *without running your code*, so palette entries, menus and settings appear immediately on install.

### Register

Runtime code in `activate()` supplies the *behaviour* -- e.g. `commands.registerCommand` binds a declared command ID to a handler.

### The split matters

A command declared but not registered shows in the palette but errors on invoke. A command registered but not declared works in code but is invisible in the UI.

---

## The commands Contribution

### Declare in package.json

```json
{
  "contributes": {
    "commands": [
      {
        "command": "myExt.helloWorld",
        "title": "Hello World",
        "category": "My Extension",
        "icon": "$(rocket)",
        "enablement": "editorHasSelection"
      }
    ]
  }
}
```

### Register in extension.ts

```typescript
import * as vscode from 'vscode';

export function activate(
  context: vscode.ExtensionContext
) {
  const disposable =
    vscode.commands.registerCommand(
      'myExt.helloWorld',
      () => {
        vscode.window.showInformationMessage(
          'Hello World from My Extension!'
        );
      }
    );

  context.subscriptions.push(disposable);
}
```

### title & category

In the palette the entry shows as `My Extension: Hello World` -- `category` is the prefix, `title` the label. The ID itself is never shown to users.

### icon & enablement

`icon` uses a `$(codicon)` reference (or light/dark paths) for toolbar buttons. `enablement` greys the command out when its `when`-style clause is false.

---

## Activation Events Deep Dive

Extensions are **lazily loaded**. `activationEvents` in `package.json` tells VS Code *when* to run your `activate()` -- keeping startup fast.

| Event | Fires when... | Use for |
|-------|---------------|---------|
| `onCommand:myExt.foo` | The command is invoked | Most commands |
| `onLanguage:python` | A file of that language opens | Language tooling |
| `onStartupFinished` | Shortly after the window loads | Background setup |
| `workspaceContains:**/*.foo` | A matching file exists in the folder | Project-specific tools |
| `onView:myView` | A contributed view becomes visible | Tree / webview views |
| `onUri` | A `vscode://` URI targets the extension | Deep links / auth callbacks |
| `onDebug` | A debug session starts | Debug adapters |
| `*` | On startup, always | Avoid -- slows the editor |

### Auto-derived events

Since VS Code 1.74, an `onCommand` event is **generated automatically** for every command in `contributes.commands`. Modern generators (`yo code`) ship an empty `activationEvents` array. The same applies to `onLanguage` from contributed languages, and `onView` / `onWebviewPanel` from contributed views.

### Rule of thumb

Activate as *late* as possible. Never use `*` unless you genuinely must run on every window.

---

## The Command Palette

The palette (`Ctrl/Cmd+Shift+P`) is the primary surface for commands. Typing `>` switches it into command mode (the default when opened with the shortcut).

### How a command surfaces

- Every entry in `contributes.commands` appears by default
- Shown as `Category: Title`, sorted & fuzzy-matched
- Recently used commands float to the top
- A bound keybinding is shown on the right

### The > prefix

The Quick Open widget is shared: no prefix = files, `>` = commands, `@` = symbols, `:` = go to line. Your commands live behind `>`.

### Gating palette visibility

To hide a command from the palette unless a condition holds, add a `commandPalette` entry under `menus` with a `when` clause:

```json
"menus": {
  "commandPalette": [
    {
      "command": "myExt.formatPython",
      "when": "editorLangId == python"
    }
  ]
}
```

A `commandPalette` `when` only controls *visibility*; use `enablement` on the command to grey it out instead of hiding it.

---

## Keybindings Contribution

```json
{
  "contributes": {
    "keybindings": [
      {
        "command": "myExt.helloWorld",
        "key": "ctrl+alt+h",
        "mac": "cmd+alt+h",
        "when": "editorTextFocus"
      },
      {
        "command": "myExt.formatPython",
        "key": "ctrl+k ctrl+f",
        "when": "editorLangId == python"
      }
    ]
  }
}
```

### The fields

- `command` -- the ID to run
- `key` -- Windows/Linux binding
- `mac` / `linux` / `win` -- platform overrides
- `when` -- context gate for the binding
- `args` -- optional argument passed to the command

### Chords

Two-stroke bindings use a space: `ctrl+k ctrl+c`. The first chord shows a pending indicator in the status bar.

### Don't override defaults

Avoid hijacking built-in keys. Prefer the `ctrl+k` prefix space, always set a `when` clause, and let users rebind freely.

---

## Menus Contribution

```json
"menus": {
  "editor/context": [
    {
      "command": "myExt.formatPython",
      "when": "editorLangId == python",
      "group": "1_modification@1"
    }
  ],
  "editor/title": [
    {
      "command": "myExt.preview",
      "when": "resourceExtname == .md",
      "group": "navigation@1"
    }
  ],
  "explorer/context": [
    { "command": "myExt.analyse", "group": "2_workspace" }
  ],
  "view/title": [
    { "command": "myExt.refresh", "when": "view == myView",
      "group": "navigation" }
  ]
}
```

### Common menu IDs

- `commandPalette` -- palette visibility
- `editor/context` -- right-click in editor
- `editor/title` -- editor tab toolbar
- `explorer/context` -- file explorer
- `view/title` -- a view's header
- `view/item/context` -- tree item right-click

### Groups & ordering

- Named groups sort alphabetically: `navigation` first, then `1_modification`, `2_workspace`...
- `@1`, `@2` set order *within* a group
- `navigation` renders as inline icons in title bars

---

## when Clause Contexts

The `when` clause is a small **boolean expression language** evaluated against the editor's current state. It gates keybindings, menus and view visibility.

| Context key | Meaning |
|-------------|---------|
| `editorLangId == 'python'` | Active editor's language |
| `editorHasSelection` | Text is selected |
| `editorTextFocus` | Focus is in a text editor |
| `resourceExtname == .md` | File extension of the resource |
| `resourceScheme == file` | URI scheme of the resource |
| `view == myView` | The focused view's ID |
| `viewItem == folder` | A tree item's `contextValue` |
| `isLinux` / `isMac` / `isWindows` | Host platform |

### Operators

`&&` and, `||` or, `!` not, `==` / `!=`, `=~` regex, `in`. Example: `editorLangId == python && editorHasSelection`.

### Custom contexts

Set your own boolean context from code, then reference it in any `when` clause:

```typescript
// In extension.ts
vscode.commands.executeCommand(
  'setContext',
  'myExt.isAuthed',
  true
);
```

```json
// In package.json
{
  "command": "myExt.sync",
  "when": "myExt.isAuthed"
}
```

Keep custom keys namespaced (`myExt.*`) and update them whenever the underlying state changes.

---

## Configuration / Settings Contribution

```json
"configuration": {
  "title": "My Extension",
  "properties": {
    "myExt.maxItems": {
      "type": "number",
      "default": 20,
      "minimum": 1,
      "description": "Maximum items to show.",
      "scope": "resource"
    },
    "myExt.logLevel": {
      "type": "string",
      "default": "info",
      "enum": ["off", "info", "debug"],
      "enumDescriptions": [
        "No logging",
        "Standard logging",
        "Verbose logging"
      ],
      "description": "Logging verbosity.",
      "scope": "window"
    },
    "myExt.enableTelemetry": {
      "type": "boolean",
      "default": true,
      "description": "Send anonymous usage data.",
      "scope": "application"
    }
  }
}
```

### Property shape

- `type` -- `string`, `number`, `boolean`, `array`, `object`
- `default` -- value when unset
- `description` / `markdownDescription`
- `enum` + `enumDescriptions` -- fixed choices

### scope

- `application` -- user-global only
- `window` -- user or workspace
- `resource` -- per folder / file (also `language-overridable`)

Always prefix keys with your extension name (`myExt.*`) so settings group neatly in the Settings UI.

---

## Reading Settings in Code

```typescript
import * as vscode from 'vscode';

// Read a value (typed, with fallback)
const config =
  vscode.workspace.getConfiguration('myExt');
const max = config.get<number>('maxItems', 20);
const level =
  config.get<string>('logLevel', 'info');

// Update a value
await config.update(
  'logLevel',
  'debug',
  vscode.ConfigurationTarget.Workspace
);

// React to changes
vscode.workspace.onDidChangeConfiguration(e => {
  if (e.affectsConfiguration('myExt.logLevel')) {
    reconfigureLogger();
  }
});
```

### getConfiguration

Pass the section prefix to scope reads. `.get<T>(key, default)` returns the effective value after merging user, workspace and folder settings.

### update

- `ConfigurationTarget.Global` -- user settings
- `.Workspace` -- `.vscode/settings.json`
- `.WorkspaceFolder` -- specific folder

### onDidChangeConfiguration

Fires on any settings change. Guard with `e.affectsConfiguration('myExt.x')` so you only react to *your* keys.

### Pass a resource

For `resource`-scoped settings, pass a URI: `getConfiguration('myExt', uri)` to read the folder-specific value.

---

## Views & View Containers

Add a custom panel to the **Activity Bar** by contributing a view *container* plus the *views* inside it. A teaser here -- tree data providers come in Part 4.

```json
"viewsContainers": {
  "activitybar": [
    {
      "id": "myExt-explorer",
      "title": "My Extension",
      "icon": "resources/icon.svg"
    }
  ]
},
"views": {
  "myExt-explorer": [
    {
      "id": "myExt.itemsView",
      "name": "Items",
      "icon": "$(list-tree)",
      "contextualTitle": "My Extension Items"
    }
  ]
}
```

### Activity Bar layout

```
+--+------------------------+
|[]| MY EXTENSION           |
|[]| +--------------------+ |
|[]| | Items           [refresh]
|[*]| v Group A          |
|  | |   item-1           |
|  | |   item-2           |
|  | | v Group B          |
|  | |   item-3           |
|  | +--------------------+ |
+--+------------------------+
 ^ your custom icon         view: myExt.itemsView
```

### Two parts

`viewsContainers` creates the Activity Bar entry; `views` populates it. The container `id` is the key that links them.

---

## Other Contribution Points

The `contributes` map has many more keys. A quick tour of the most common -- each is purely declarative JSON.

| Contribution point | What it adds |
|--------------------|--------------|
| `languages` | Registers a language ID, file extensions, and a configuration file (brackets, comments) |
| `grammars` | TextMate grammar for syntax highlighting, mapped to a language scope |
| `snippets` | Code snippets for a language, loaded from a JSON snippets file |
| `themes` | Colour themes -- a label, `uiTheme` base, and a theme JSON path |
| `iconThemes` | File icon themes that style the explorer and tabs |
| `jsonValidation` | Associates a JSON schema with a file pattern for validation & IntelliSense |
| `customEditors` | Webview-based editors for custom file types (e.g. an image or CAD viewer) |
| `walkthroughs` | Step-by-step Getting Started pages shown on the Welcome screen |

### Discover them all

The full, authoritative list lives in the Contribution Points reference. Many of these (`customEditors`, `walkthroughs`, views) also need runtime registration -- covered in later parts.

---

## Summary & Further Reading

### Key Takeaways

- `contributes` declares UI; `activate()` registers behaviour
- Commands need both a declaration and a `registerCommand`
- Activation events keep startup lazy -- `onCommand` is auto-derived
- Keybindings, menus & palette all gate on `when` clauses
- Menu `group` + `@n` control placement & order
- Settings live in `configuration.properties`, read via `getConfiguration`
- View containers add panels to the Activity Bar

### Hands-On Exercise

- Declare a command and register a handler that shows a message
- Add a `keybindings` entry (try a `ctrl+k` chord)
- Add an `editor/context` menu entry gated by a `when` clause (e.g. `editorLangId == markdown`)
- Add a boolean setting and read it in the handler

### Further Reading

- **Contribution Points** -- code.visualstudio.com/api/references/contribution-points
- **Activation Events** -- code.visualstudio.com/api/references/activation-events
- **when clause contexts** -- code.visualstudio.com/api/references/when-clause-contexts
- **Commands guide** -- code.visualstudio.com/api/extension-guides/command

### Tools Worth Knowing

`yo code` (scaffold) - `vsce` (package & publish) - the Extension Development Host (F5) - the built-in `Developer: Inspect Context Keys` command for debugging `when` clauses.

### Next

`Part 2 done` -> `Part 3 — The Extension API in Depth`
