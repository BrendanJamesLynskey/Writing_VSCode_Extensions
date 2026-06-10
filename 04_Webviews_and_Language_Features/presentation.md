# Writing VS Code Extensions — Part 4: Webviews & Language Features

**Part 4 — Webviews & Language Features**

> Build custom UI in the sidebar and panels, then teach the editor about your language -- completions, hovers, diagnostics and, ultimately, a language server.

`Tree View` -> `Webview` -> `Language Providers` -> `LSP`

Tree Views - Webviews - Providers - LSP

---

## Table of Contents

1. [Topics](#topics)
2. [Custom UI — Choosing a Surface](#custom-ui--choosing-a-surface)
3. [Tree View API](#tree-view-api)
4. [Tree View Interactions](#tree-view-interactions)
5. [Webview API](#webview-api)
6. [Webview ↔ Extension Messaging](#webview--extension-messaging)
7. [Webview Security & Resources](#webview-security--resources)
8. [Webview Views (Sidebar)](#webview-views-sidebar)
9. [Language Features — The Provider Model](#language-features--the-provider-model)
10. [IntelliSense — Completions & Hover](#intellisense--completions--hover)
11. [Diagnostics (Linting)](#diagnostics-linting)
12. [Language Server Protocol (LSP)](#language-server-protocol-lsp)
13. [Summary & Further Reading](#summary--further-reading)

---

## Topics

Parts 1–3 covered activation, contributions and the `vscode` runtime API. Part 4 builds **custom UI** and **language smarts** on top of that foundation.

### Custom UI

- Choosing a UI surface
- Tree View API & `TreeDataProvider`
- Tree item commands & context menus
- Webview panels & the iframe sandbox

### Webview Deep Dive

- Bidirectional `postMessage` protocol
- CSP, nonces & `asWebviewUri`
- State persistence & retention
- Webview Views in the sidebar

### Language Features

- The provider model
- IntelliSense & hovers
- Diagnostics & quick fixes
- Definitions, references, rename

### Going Further

- The Language Server Protocol
- Client & server packages
- JSON-RPC over a separate process
- In-process providers vs LSP

---

## Custom UI — Choosing a Surface

VS Code offers several ways to render your own UI. Reach for the **simplest** one that does the job -- native widgets are cheaper and more accessible than a webview.

| Surface | API | Renders | Reach For When… |
|---------|-----|---------|-----------------|
| **Tree View** | `TreeDataProvider` | Native list/tree widget | Hierarchical or list data in the sidebar |
| **Webview Panel** | `createWebviewPanel` | HTML/CSS/JS in an editor tab | Rich, bespoke UI (charts, forms, previews) |
| **Webview View** | `WebviewViewProvider` | HTML/CSS/JS in the sidebar | Custom UI docked in a view container |
| **Custom Editor** | `CustomEditorProvider` | Webview bound to a file type | Editing a custom file format visually |
| **Status Bar** | `createStatusBarItem` | Text/icon in the status bar | Glanceable status or a quick command entry |

### Prefer Native

Tree Views, Quick Picks and Status Bar items match the user's theme automatically, are keyboard-accessible, and need no HTML. Use them first.

### Webview Trade-offs

A webview is a sandboxed iframe -- powerful, but you own all HTML/CSS/JS, theming, accessibility and security. More flexible, more responsibility.

---

## Tree View API

```typescript
import * as vscode from 'vscode';

class NodeItem extends vscode.TreeItem {
  constructor(label: string, public children: NodeItem[] = []) {
    super(label, children.length
      ? vscode.TreeItemCollapsibleState.Collapsed
      : vscode.TreeItemCollapsibleState.None);
    this.iconPath = new vscode.ThemeIcon('symbol-field');
    this.contextValue = 'node';          // keys menus
  }
}

class MyProvider implements vscode.TreeDataProvider<NodeItem> {
  private _onDidChange = new vscode.EventEmitter<NodeItem | void>();
  readonly onDidChangeTreeData = this._onDidChange.event;

  getTreeItem(el: NodeItem): vscode.TreeItem { return el; }
  getChildren(el?: NodeItem): NodeItem[] {
    return el ? el.children : this.roots;
  }
  refresh() { this._onDidChange.fire(); }   // re-render
}

const provider = new MyProvider();
vscode.window.registerTreeDataProvider('myView', provider);
// or: const view = vscode.window.createTreeView('myView',
//       { treeDataProvider: provider });
```

### Implement Two Methods

- `getChildren(el?)` -- children of a node, or roots when no arg
- `getTreeItem(el)` -- the `TreeItem` to render

### TreeItem Fields

- `label`, `description`, `tooltip`
- `collapsibleState` -- None / Collapsed / Expanded
- `command` -- run on click
- `iconPath` -- a `ThemeIcon` or file
- `contextValue` -- keys menu visibility

### Refreshing

Fire an `EventEmitter` through `onDidChangeTreeData`. Fire with no arg to refresh the whole tree, or with an element to refresh one node.

### Tree View in the Sidebar

```
MY VIEW
 v src
    v commands
       * hello.ts
       * open.ts
 > tests
```

`getChildren(root)` returns `NodeItem[]`; each element is mapped through `getTreeItem()` to a `TreeItem` (label, icon, command, collapsibleState, contextValue).

---

## Tree View Interactions

### Item Commands

Set `treeItem.command` to run a command when the row is clicked. Pass the item as an argument so the handler knows what was selected.

### Context Menus

Contribute to `view/item/context` in `package.json`. Use the `when` clause `viewItem == <contextValue>` to show actions only on matching items.

### Inline Icons

Group `"inline"` renders an action as an icon button on hover. The command receives the tree item, just like a context-menu action.

### View Title Actions

`view/title` places buttons (e.g. Refresh) in the view header, keyed on `view == myView`.

```json
{
  "contributes": {
    "commands": [
      { "command": "myView.refresh", "title": "Refresh",
        "icon": "$(refresh)" },
      { "command": "myView.delete", "title": "Delete",
        "icon": "$(trash)" }
    ],
    "menus": {
      "view/title": [
        { "command": "myView.refresh", "when": "view == myView",
          "group": "navigation" }
      ],
      "view/item/context": [
        { "command": "myView.delete",
          "when": "view == myView && viewItem == node",
          "group": "inline" }
      ]
    }
  }
}
```

```typescript
vscode.commands.registerCommand('myView.delete',
  (item: NodeItem) => {
    model.remove(item);
    provider.refresh();          // re-render the tree
  });
```

---

## Webview API

A **webview** is a fully sandboxed `iframe` rendered inside an editor tab. You own its HTML, CSS and JS -- it cannot import the `vscode` module or touch the editor directly.

```typescript
import * as vscode from 'vscode';

export function openPreview(context: vscode.ExtensionContext) {
  const panel = vscode.window.createWebviewPanel(
    'myPreview',                 // viewType (internal id)
    'My Preview',                // tab title
    vscode.ViewColumn.Beside,    // where to show it
    {
      enableScripts: true,       // allow <script> to run
      retainContextWhenHidden: true,
    }
  );

  panel.webview.html = getHtml(panel.webview);

  panel.onDidDispose(() => {
    // clean up timers, listeners, etc.
  }, null, context.subscriptions);
}

function getHtml(webview: vscode.Webview): string {
  return `<!DOCTYPE html><html><body>
    <h1>Hello from a webview</h1>
  </body></html>`;
}
```

### createWebviewPanel

- `viewType` -- stable id for serialisation
- `title` -- shown on the tab
- `showOptions` -- a `ViewColumn`
- `options` -- `enableScripts`, roots…

### Set the Content

Assign a full HTML document string to `panel.webview.html`. Reassigning it reloads the iframe from scratch.

### It Is Isolated

No Node, no `require`, no `vscode` API. Scripts are off by default -- set `enableScripts: true` to turn them on.

---

## Webview ↔ Extension Messaging

The extension host and the webview iframe talk only by **passing serialisable messages**. There is no shared memory -- this channel is the entire bridge.

```
Extension Host                                  Webview (iframe)
+----------------------------+                  +----------------------------+
| panel.webview.postMessage  | --- message ---> | window.addEventListener    |
|   onDidReceiveMessage(cb)  | <-- postMessage  | acquireVsCodeApi()          |
| Node.js, full vscode API   |                  | browser sandbox, no API    |
+----------------------------+                  +----------------------------+
```

### In the Extension

```typescript
// receive from the webview
panel.webview.onDidReceiveMessage(msg => {
  if (msg.type === 'save') save(msg.value);
}, undefined, context.subscriptions);

// send to the webview
panel.webview.postMessage({ type: 'update', count: 42 });
```

### Inside the Webview

```javascript
const vscode = acquireVsCodeApi();

// send to the extension
vscode.postMessage({ type: 'save', value: text });

// receive from the extension
window.addEventListener('message', event => {
  const msg = event.data;
  if (msg.type === 'update') render(msg.count);
});
```

---

## Webview Security & Resources

```typescript
function getHtml(webview: vscode.Webview, ext: vscode.Uri) {
  // local files must be converted to webview URIs
  const scriptUri = webview.asWebviewUri(
    vscode.Uri.joinPath(ext, 'media', 'main.js'));
  const nonce = getNonce();   // random per render

  return `<!DOCTYPE html><html><head>
    <meta http-equiv="Content-Security-Policy" content="
      default-src 'none';
      img-src ${webview.cspSource} https:;
      style-src ${webview.cspSource};
      script-src 'nonce-${nonce}';">
  </head><body>
    <div id="app"></div>
    <script nonce="${nonce}" src="${scriptUri}"></script>
  </body></html>`;
}

// allow the webview to load from these folders only
const panel = vscode.window.createWebviewPanel(
  'view', 'Title', vscode.ViewColumn.One, {
    enableScripts: true,
    localResourceRoots: [vscode.Uri.joinPath(ext, 'media')],
    retainContextWhenHidden: true,
  });
```

### Lock Down the CSP

- Start from `default-src 'none'`
- Allow scripts only via a per-render `nonce`
- Use `${webview.cspSource}` for local assets

### Load Local Files

`webview.asWebviewUri(uri)` rewrites a disk path into a URI the sandbox can fetch. List folders in `localResourceRoots`.

### Persist State

- `retainContextWhenHidden` keeps the DOM alive (costs memory)
- In the webview, `getState()` / `setState()` survive reloads cheaply

### Never Trust Input

Treat every message from the webview as untrusted -- validate before acting on it.

---

## Webview Views (Sidebar)

A **Webview View** renders custom HTML inside a sidebar view container instead of an editor tab -- the same iframe sandbox and messaging, docked in the activity bar.

```typescript
class MyViewProvider implements vscode.WebviewViewProvider {
  constructor(private readonly ext: vscode.Uri) {}

  resolveWebviewView(
    view: vscode.WebviewView,
    _ctx: vscode.WebviewViewResolveContext,
    _token: vscode.CancellationToken,
  ) {
    view.webview.options = {
      enableScripts: true,
      localResourceRoots: [this.ext],
    };
    view.webview.html = getHtml(view.webview, this.ext);

    view.webview.onDidReceiveMessage(msg => {
      /* handle messages from the view */
    });
  }
}

export function activate(context: vscode.ExtensionContext) {
  const provider = new MyViewProvider(context.extensionUri);
  context.subscriptions.push(
    vscode.window.registerWebviewViewProvider(
      'myWebviewView', provider));
}
```

### package.json

```json
"contributes": {
  "views": {
    "explorer": [
      { "type": "webview",
        "id": "myWebviewView",
        "name": "My Panel" }
    ]
  }
}
```

### Key Points

- Set `"type": "webview"` on the view contribution
- The `id` must match `registerWebviewViewProvider`
- `resolveWebviewView` is called when the view first becomes visible

---

## Language Features — The Provider Model

You don't push features into the editor. You **register a provider** against a `DocumentSelector`; VS Code calls it back when the user types, hovers, or invokes an action.

```
User Action            VS Code Editor              Your Provider
type / hover / F12 --> matches DocumentSelector --> returns items / hover / range
```

### Authoring

- `CompletionItemProvider`
- `HoverProvider`
- `SignatureHelpProvider`

### Navigation

- `DefinitionProvider`
- `ReferenceProvider`
- `DocumentSymbolProvider`

### Quality & Edits

- `DiagnosticCollection`
- `CodeActionProvider`
- `DocumentFormattingEditProvider`, `RenameProvider`

---

## IntelliSense — Completions & Hover

```typescript
const completion: vscode.CompletionItemProvider = {
  provideCompletionItems(doc, position) {
    const log = new vscode.CompletionItem(
      'console.log', vscode.CompletionItemKind.Method);
    log.insertText =
      new vscode.SnippetString('console.log($1)$0');
    log.documentation =
      new vscode.MarkdownString('Log to the **console**');

    const tag = new vscode.CompletionItem(
      'TODO', vscode.CompletionItemKind.Keyword);

    return [log, tag];   // CompletionItem[]
  },
};

context.subscriptions.push(
  vscode.languages.registerCompletionItemProvider(
    { language: 'javascript' }, completion,
    '.', ':'));          // trigger characters
```

```typescript
const hover: vscode.HoverProvider = {
  provideHover(doc, position) {
    const word = doc.getText(
      doc.getWordRangeAtPosition(position));
    return new vscode.Hover(
      new vscode.MarkdownString(`**${word}** -- symbol`));
  },
};
vscode.languages.registerHoverProvider(
  { language: 'javascript' }, hover);
```

### CompletionItem

- `label` -- text in the list
- `kind` -- `CompletionItemKind` icon
- `insertText` -- string or `SnippetString`
- `documentation` -- `MarkdownString`

### SnippetString

`$1`, `$2` are tab stops; `$0` is the final cursor; `${1:label}` gives a placeholder.

### Trigger Characters

Pass characters after the provider (e.g. `'.'`) so completions pop automatically as the user types them.

### Hover

Return a `Hover` built from a `MarkdownString`; render docs, types or links on mouse-over.

---

## Diagnostics (Linting)

```typescript
const collection =
  vscode.languages.createDiagnosticCollection('mylint');

function lint(doc: vscode.TextDocument) {
  const diagnostics: vscode.Diagnostic[] = [];

  for (let line = 0; line < doc.lineCount; line++) {
    const text = doc.lineAt(line).text;
    const idx = text.indexOf('FIXME');
    if (idx === -1) continue;

    const range = new vscode.Range(
      line, idx, line, idx + 5);
    const diag = new vscode.Diagnostic(
      range, 'Unresolved FIXME comment',
      vscode.DiagnosticSeverity.Warning);
    diag.source = 'mylint';
    diagnostics.push(diag);
  }
  collection.set(doc.uri, diagnostics);
}

// re-lint as the document changes
context.subscriptions.push(
  vscode.workspace.onDidChangeTextDocument(
    e => lint(e.document)),
  vscode.workspace.onDidOpenTextDocument(lint),
  collection);
```

### Diagnostic

- `range` -- a `Range` to underline
- `message` -- shown on hover
- `severity` -- Error / Warning / Information / Hint
- `source`, `code` -- optional metadata

### DiagnosticCollection

`collection.set(uri, diags)` publishes squiggles for a file; pass `[]` or `delete(uri)` to clear them.

### Refresh on Change

Re-run linting from `onDidChangeTextDocument` (debounce for big files) and on open/close events.

### Quick Fixes

Register a `CodeActionProvider` to offer a fix for a diagnostic -- build a `WorkspaceEdit` and attach it to the action.

---

## Language Server Protocol (LSP)

Write your language smarts **once** as a server speaking the protocol, then reuse it across VS Code, Neovim and other editors. The server runs in its **own process**.

```
Language Client                                Language Server
+----------------------------+                 +----------------------------+
| vscode-languageclient      |  initialize     | vscode-languageserver      |
| in the extension host      | --------------> | separate process           |
| forwards editor events     |  didChange      | owns parsing & analysis    |
|                            | <-------------- |                            |
|                            | publishDiag.    | editor-agnostic            |
+----------------------------+                 +----------------------------+
            JSON-RPC over stdio / IPC -- one server, many editors
```

Example messages: `initialize` -> `textDocument/didChange` -> `textDocument/completion` -> completion result -> `textDocument/publishDiagnostics`.

### Reach for LSP When…

- You want the same smarts in multiple editors
- Analysis is heavy -- isolate it in its own process
- The language is large (real parser, symbol table)
- A community LSP server already exists to wrap

### Stay In-Process When…

- A handful of providers cover your needs
- VS Code is the only target editor
- You want the simplest possible setup
- Logic is light and tied to the `vscode` API

---

## Summary & Further Reading

### Key Takeaways

- Pick the simplest UI surface -- native before webview
- Tree Views: implement `TreeDataProvider`, refresh via `EventEmitter`
- Webviews are sandboxed iframes -- you own HTML/CSS/JS
- Bridge only via `postMessage` & `onDidReceiveMessage`
- Lock down the CSP, nonce scripts, never trust input
- Language features are providers VS Code calls back
- Reach for LSP to share smarts across editors

### Hands-On Exercise

Add a **Tree View** listing TODOs found in the workspace (scan files via `workspace.findFiles`), and open a **webview detail panel** showing the surrounding code when an item is clicked.

### Further Reading

- **Tree View API** -- code.visualstudio.com/api/extension-guides/tree-view
- **Webview API Guide** -- code.visualstudio.com/api/extension-guides/webview
- **Webview View Guide** -- code.visualstudio.com/api/extension-guides/webview-view
- **Language Extensions** -- code.visualstudio.com/api/language-extensions/programmatic-language-features
- **LSP Specification** -- microsoft.github.io/language-server-protocol

### Next in the Series

`Part 4` -> `Part 5` -- Testing, Packaging & Publishing

Up next: the test runner, `vsce` packaging, and shipping to the Marketplace.
