# Writing VS Code Extensions

**Part 3 — The Extension API in Depth**

> A guided tour of the `vscode` namespace -- the objects, events and disposables you use to read, edit and react to the editor.

`window` -> `workspace` -> `editor` -> `fs`

Namespaces - Disposables - Editors - Events

---

## Table of Contents

1. [Agenda](#agenda)
2. [The vscode Namespace Map](#the-vscode-namespace-map)
3. [Disposables & context.subscriptions](#disposables--contextsubscriptions)
4. [window — User Messages](#window--user-messages)
5. [window — Quick Pick](#window--quick-pick)
6. [window — Input Box](#window--input-box)
7. [window — Status Bar](#window--status-bar)
8. [window — Progress & Long Tasks](#window--progress--long-tasks)
9. [workspace — Folders & Files](#workspace--folders--files)
10. [TextDocument & TextEditor](#textdocument--texteditor)
11. [Editing Text](#editing-text)
12. [The File System & Events](#the-file-system--events)
13. [Summary & Further Reading](#summary--further-reading)

---

## Agenda

This part assumes you have an activated extension from Parts 1-2. We now go deep on the **runtime API surface** -- the `vscode` module your code imports.

### Foundations

- The `vscode` namespace map
- Disposables & `context.subscriptions`
- Why everything you register must be disposed

### The `window` namespace

- Messages, Quick Pick, Input Box
- Status Bar items with codicons
- Progress for long-running tasks

### The `workspace` & editors

- Folders, `findFiles`, configuration
- `TextDocument` & `TextEditor`
- Editing with `edit()` & `WorkspaceEdit`

### Files, state & events

- `workspace.fs` & file-system watchers
- Document save/change events
- State: global, workspace, secrets

---

## The vscode Namespace Map

Your extension imports a single module -- `import * as vscode from 'vscode';` -- whose top-level namespaces group every API by concern.

```
                         +----------+
                         |  vscode  |
                         +----+-----+
        +---------+---------+--+--+---------+---------+
        |         |         |     |         |         |
    window   workspace  commands  languages  env  extensions  debug  tasks
```

- **window** -- editors, UI, messages, status bar
- **workspace** -- folders, files, config, fs, events
- **commands** -- register & execute commands
- **languages** -- completion, diagnostics, hovers
- **env** -- clipboard, openExternal, UI kind
- **extensions** -- inspect & call other extensions
- **debug** -- debug sessions & breakpoints
- **tasks** -- register & run build/run tasks

Also: `commands.executeCommand(...)` is the universal bridge between every namespace.

This deck focuses on **window**, **workspace** and the editor/document/file types they expose -- the spine of almost every extension.

---

## Disposables & context.subscriptions

Almost everything you *register* returns a **`vscode.Disposable`**. Push each one onto `context.subscriptions` so VS Code disposes it automatically on `deactivate`.

```typescript
import * as vscode from 'vscode';

export function activate(context: vscode.ExtensionContext) {
  // registerCommand returns a Disposable
  const cmd = vscode.commands.registerCommand(
    'myExt.hello', () => vscode.window.showInformationMessage('Hi!')
  );

  // Event subscriptions are Disposables too
  const sub = vscode.workspace.onDidSaveTextDocument(doc =>
    console.log('saved', doc.fileName)
  );

  // Hand ownership to VS Code — cleaned up on deactivate
  context.subscriptions.push(cmd, sub);

  // Manual disposal is also possible:
  // const d = vscode.Disposable.from(cmd, sub); d.dispose();
}

export function deactivate() {}
```

### The Pattern

- Register -> get a `Disposable`
- `.push()` it to `subscriptions`
- VS Code calls `.dispose()` for you

### If You Don't

- Listeners survive deactivation -> **leaks**
- Commands double-register on reload -> errors
- Status bar / watchers linger in memory

### Rule of Thumb

If a call returns a `Disposable`, it belongs in `context.subscriptions`.

---

## window — User Messages

Three notification levels, each returning a **promise**. Pass string arguments to add buttons; the promise resolves to the clicked label (or `undefined`).

```typescript
// Plain notifications
vscode.window.showInformationMessage('Build complete.');
vscode.window.showWarningMessage('Unsaved changes.');
vscode.window.showErrorMessage('Build failed.');

// Buttons → resolves to the chosen string
const choice = await vscode.window.showInformationMessage(
  'Delete this file?',
  'Delete', 'Cancel'
);
if (choice === 'Delete') { /* ... */ }

// Modal blocks until answered
const ok = await vscode.window.showWarningMessage(
  'This cannot be undone.',
  { modal: true },
  'Proceed'
);

// MessageItem objects let you attach metadata
const picked = await vscode.window.showErrorMessage(
  'Port in use.',
  { title: 'Retry' }, { title: 'Ignore', isCloseAffordance: true }
);
```

### Levels

- `showInformationMessage`
- `showWarningMessage`
- `showErrorMessage`

### Options

- Strings -> buttons
- `{ modal: true }` forces a response
- `MessageItem` -- `title`, `isCloseAffordance`

### Tip

Always `await` the result -- the promise is how you learn which button was pressed.

---

## window — Quick Pick

The Quick Pick is VS Code's searchable dropdown. Use `showQuickPick` for the simple case, or `createQuickPick` when you need full control.

### Simple — `showQuickPick`

```typescript
// From an array of strings
const fruit = await vscode.window.showQuickPick(
  ['Apple', 'Banana', 'Cherry'],
  { placeHolder: 'Pick a fruit' }
);

// From rich QuickPickItems
const items: vscode.QuickPickItem[] = [
  { label: '$(file) Open File', description: 'Ctrl+O',
    detail: 'Open a file from disk' },
  { label: '$(search) Find',    description: 'Ctrl+F' },
];
const chosen = await vscode.window.showQuickPick(items, {
  matchOnDescription: true, canPickMany: false,
});
chosen?.label;  // selected item
```

### Rich — `createQuickPick`

```typescript
const qp = vscode.window.createQuickPick();
qp.placeholder = 'Type to filter…';
qp.items = [
  { label: 'main.ts',  description: 'src/' },
  { label: 'utils.ts', description: 'src/lib/' },
];
qp.onDidChangeValue(v => { /* live filter / fetch */ });
qp.onDidAccept(() => {
  const sel = qp.selectedItems[0];
  vscode.window.showInformationMessage(sel.label);
  qp.hide();
});
qp.onDidHide(() => qp.dispose());
qp.show();
```

### QuickPickItem

`label` (with `$(codicon)`) - `description` (greyed, inline) - `detail` (second line).

---

## window — Input Box

Prompt the user for free text with `showInputBox`. `validateInput` runs on every keystroke -- return a string to show an error, or `undefined` / `null` to accept.

```typescript
const name = await vscode.window.showInputBox({
  prompt: 'Name your new component',
  placeHolder: 'e.g. UserCard',
  value: 'MyComponent',          // pre-filled default
  ignoreFocusOut: true,          // survive focus loss
  validateInput: (text) => {
    if (!text.trim()) return 'Name cannot be empty';
    if (!/^[A-Z]/.test(text))
      return 'Components must start with a capital letter';
    return undefined;            // valid
  },
});

if (name === undefined) {
  return;  // user pressed Escape
}
vscode.window.showInformationMessage(`Creating ${name}…`);
```

### InputBoxOptions

- `prompt` -- the label text
- `placeHolder` -- ghost hint
- `value` -- initial contents
- `password` -- masks input
- `ignoreFocusOut` -- keep open

### validateInput

- Runs on every change
- Return `string` -> error message
- Return `undefined` -> valid
- May be `async` (returns a promise)

### Escape

Resolves to `undefined` -- always check before proceeding.

---

## window — Status Bar

Create a persistent item with `createStatusBarItem(alignment, priority)`. Set `.text` (with `$(codicon)` icons), wire a `.command`, then call `.show()`.

```typescript
const item = vscode.window.createStatusBarItem(
  vscode.StatusBarAlignment.Right,  // Left | Right
  100                               // priority (higher = further left)
);

item.text = '$(rocket) Deploy';     // codicon + label
item.tooltip = 'Deploy to staging';
item.command = 'myExt.deploy';      // click → runs command
item.backgroundColor =
  new vscode.ThemeColor('statusBarItem.warningBackground');
item.show();

context.subscriptions.push(item);   // dispose on deactivate

// Update it later, e.g. from an event
item.text = '$(sync~spin) Deploying…';
```

### Status Bar Diagram

```
+------------------------------------------------------------------+
| $(source-control) main*   $(error) 0  $(warning) 2   [ ↗ Deploy ]|
+------------------------------------------------------------------+
  (our item: Alignment.Right, priority 100, highlighted on the right)
```

### Alignment & Priority

- `StatusBarAlignment.Left` / `.Right`
- Higher priority sits closer to the centre

### Properties

- `.text` -- supports `$(icon)` codicons
- `.tooltip`, `.command`
- `.color` / `.backgroundColor` via `ThemeColor`
- `.show()` / `.hide()`

### Codicons

Animate with `~spin`, e.g. `$(sync~spin)`. Browse the full set in the codicon reference.

---

## window — Progress & Long Tasks

Wrap any async work in `withProgress`. VS Code shows a spinner or bar and resolves when your callback's promise settles. Report incremental updates via `progress.report`.

```typescript
await vscode.window.withProgress(
  {
    location: vscode.ProgressLocation.Notification,
    title: 'Indexing project',
    cancellable: true,
  },
  async (progress, token) => {
    const files = await vscode.workspace.findFiles('**/*.ts');
    const step = 100 / files.length;

    for (const [i, file] of files.entries()) {
      if (token.isCancellationRequested) break;

      progress.report({
        increment: step,                  // adds to the bar (0–100)
        message: `${i + 1}/${files.length} ${file.path}`,
      });
      await indexFile(file);
    }
  }
);
```

### ProgressLocation

- `.Notification` -- toast with bar
- `.Window` -- spinner in status bar
- `.SourceControl` -- in the SCM view

### progress.report

- `increment` -- percentage to add
- `message` -- live status line
- Omit `increment` for indeterminate

### Cancellation

Set `cancellable: true`, then poll `token.isCancellationRequested` in your loop.

---

## workspace — Folders & Files

The `workspace` namespace describes the folders the user has open and lets you find, open and configure files across them.

```typescript
// Open folders (undefined if none)
const folders = vscode.workspace.workspaceFolders;
const root = folders?.[0].uri;          // a vscode.Uri

// Glob for files (respects .gitignore by default)
const tsFiles = await vscode.workspace.findFiles(
  '**/*.ts',            // include glob
  '**/node_modules/**', // exclude glob
  100                   // max results
);

// Open a document (does not show it in an editor)
const doc = await vscode.workspace.openTextDocument(tsFiles[0]);
const docFromString = await vscode.workspace.openTextDocument({
  language: 'json', content: '{ "ok": true }',
});

// Read settings (extension + user + workspace merged)
const cfg = vscode.workspace.getConfiguration('myExt');
const max = cfg.get<number>('maxItems', 50);   // with default
await cfg.update('maxItems', 100,
  vscode.ConfigurationTarget.Workspace);
```

### Folders

- `workspaceFolders` -- array or `undefined`
- Each has a `uri`, `name`, `index`
- `getWorkspaceFolder(uri)` resolves owner

### Finding & Opening

- `findFiles(include, exclude, max)`
- `openTextDocument(uri | options)`
- Show it with `window.showTextDocument(doc)`

### Configuration

- `getConfiguration('section')`
- `.get` / `.update` / `.has`
- Listen via `onDidChangeConfiguration`

---

## TextDocument & TextEditor

A **`TextDocument`** is the file's content model; a **`TextEditor`** is a view onto it, carrying the selection. Positions are zero-based.

```typescript
const editor = vscode.window.activeTextEditor;
if (!editor) return;                 // no editor focused

const doc = editor.document;
const fullText = doc.getText();      // entire file as string
const lineCount = doc.lineCount;
const langId = doc.languageId;       // 'typescript', etc.

// The current selection (a Range of two Positions)
const sel: vscode.Selection = editor.selection;
const selectedText = doc.getText(sel);
const cursor: vscode.Position = sel.active;   // line, character

// Build positions and ranges explicitly
const start = new vscode.Position(0, 0);
const end   = new vscode.Position(2, 10);
const range = new vscode.Range(start, end);
const firstThreeLines = doc.getText(range);

// Inspect a single line
const line = doc.lineAt(cursor.line);
line.text;                 // the line's string
line.isEmptyOrWhitespace;  // boolean
line.range;                // a Range covering it
```

### TextDocument

- `getText(range?)`, `lineAt(n)`
- `lineCount`, `languageId`, `uri`
- `isDirty`, `save()`
- `offsetAt` / `positionAt`

### TextEditor

- `document`, `selection`, `selections`
- `visibleRanges`, `revealRange()`
- `edit(...)` -- mutate the buffer

### Position / Range

Both are **immutable**. A `Selection` is a `Range` with `anchor` and `active` ends.

---

## Editing Text

For the **active editor**, use `editor.edit()`. For changes spanning **multiple files** (or applied without focus), build a `WorkspaceEdit` and apply it.

### Single editor — `editor.edit`

```typescript
const editor = vscode.window.activeTextEditor!;
await editor.edit(editBuilder => {
  // Insert at a Position
  editBuilder.insert(
    new vscode.Position(0, 0), '// header\n'
  );
  // Replace a Range
  editBuilder.replace(editor.selection, 'REPLACED');
  // Delete a Range
  editBuilder.delete(
    new vscode.Range(5, 0, 6, 0)
  );
});
// All edits in one builder = one undo step
```

Scope: operates on the **visible, active** editor only. Atomic & undoable as a unit.

### Multi-file — `WorkspaceEdit`

```typescript
const edit = new vscode.WorkspaceEdit();

edit.insert(uriA, new vscode.Position(0, 0), '// A\n');
edit.replace(uriB,
  new vscode.Range(1, 0, 1, 5), 'hello');
edit.delete(uriC, new vscode.Range(0, 0, 1, 0));

// Can also create / rename / delete whole files
edit.createFile(newUri, { ignoreIfExists: true });
edit.renameFile(oldUri, newUri);

const applied = await vscode.workspace.applyEdit(edit);
```

Scope: spans **many files**, no editor needed. Powers refactors, rename providers and code actions.

---

## The File System & Events

Use `workspace.fs` for raw file I/O (works on remote/virtual file systems), and the event APIs to react as documents change.

```typescript
const fs = vscode.workspace.fs;

// Read → returns a Uint8Array
const bytes = await fs.readFile(uri);
const text  = new TextDecoder().decode(bytes);

// Write ← expects a Uint8Array
await fs.writeFile(uri,
  new TextEncoder().encode('hello\n'));

// Metadata, dirs, copy, delete
const info = await fs.stat(uri);       // FileStat
await fs.createDirectory(dirUri);
await fs.delete(uri, { recursive: true });

// Watch a glob for create/change/delete
const watcher =
  vscode.workspace.createFileSystemWatcher('**/*.ts');
watcher.onDidCreate(uri => { /* … */ });
watcher.onDidChange(uri => { /* … */ });
watcher.onDidDelete(uri => { /* … */ });

// Document lifecycle events
vscode.workspace.onDidSaveTextDocument(doc => {});
vscode.workspace.onDidChangeTextDocument(e => {
  e.document; e.contentChanges; // what changed
});
// Remember: push every watcher/listener to subscriptions
```

### workspace.fs

- `readFile` / `writeFile` -- `Uint8Array`
- `stat`, `readDirectory`, `copy`, `delete`
- Encode/decode with `TextEncoder`/`TextDecoder`

### Events

- `createFileSystemWatcher(glob)`
- `onDidSaveTextDocument`
- `onDidChangeTextDocument`
- All return `Disposable`s

### Extension State

- `context.globalState` -- across windows
- `context.workspaceState` -- per workspace
- `context.secrets` -- encrypted store
- `context.extensionUri` -- bundled assets

---

## Summary & Further Reading

### Key Takeaways

- The `vscode` module groups APIs into namespaces -- `window`, `workspace`, `commands`, `languages`...
- Everything you register is a `Disposable` -> push to `context.subscriptions`
- `window` drives all UI: messages, Quick Pick, Input Box, status bar, progress
- `workspace` finds files, reads config, and exposes `fs` + events
- Edit with `editor.edit()` (one file) or `WorkspaceEdit` (many)

### The Through-Line

`window` -> `workspace` -> `editor` -> `fs`

### Hands-On Exercise

Build a command that:

- `findFiles` -> feed results into a **Quick Pick**
- `openTextDocument` + `showTextDocument` the chosen file
- Use `editor.edit()` to **insert a timestamped header** at the top
- Register it, and push the `Disposable` to `subscriptions`

### Further Reading

- **VS Code API Reference** -- code.visualstudio.com/api/references/vscode-api
- **Extension Capabilities** -- code.visualstudio.com/api/extension-capabilities/overview
- **Codicon Reference** -- for `$(icon)` names

### Next

**Part 4 — Webviews & Language Features:** custom UI panels, completion, hovers, diagnostics and code actions.
