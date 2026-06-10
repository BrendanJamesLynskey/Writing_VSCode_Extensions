# 🧩 Writing VS Code Extensions

A **step-by-step, five-part tutorial series** on building Visual Studio Code extensions — from scaffolding an empty project to publishing a tested extension on the Marketplace. Each part is a self-contained interactive [Reveal.js](https://revealjs.com) presentation with copy-and-run code and hand-drawn diagrams.

## ▶ [Open the Series Landing Page](https://brendanjameslynskey.github.io/Writing_VSCode_Extensions/)

---

## Presentations

| # | Presentation | Description |
|---|--------------|-------------|
| 01 | [Getting Started](https://brendanjameslynskey.github.io/Writing_VSCode_Extensions/01_Getting_Started/) ([md](01_Getting_Started/presentation.md)) | The Extension Host architecture, prerequisites, scaffolding with `yo code`, project anatomy, the `package.json` manifest, `activate()`/`deactivate()`, your first command, running under F5, activation events, and debugging |
| 02 | [Commands & Contribution Points](https://brendanjameslynskey.github.io/Writing_VSCode_Extensions/02_Commands_and_Contribution_Points/) ([md](02_Commands_and_Contribution_Points/presentation.md)) | The `contributes` map — commands, activation events, keybindings, menus and groups, `when`-clause contexts, configuration/settings, view containers, and the full contribution-point catalogue |
| 03 | [The Extension API](https://brendanjameslynskey.github.io/Writing_VSCode_Extensions/03_The_Extension_API/) ([md](03_The_Extension_API/presentation.md)) | The `vscode` namespace map, disposables and `context.subscriptions`, messages, quick picks, input boxes, the status bar and progress, the workspace, `TextDocument`/`TextEditor`, edits, and the file system & events |
| 04 | [Webviews & Language Features](https://brendanjameslynskey.github.io/Writing_VSCode_Extensions/04_Webviews_and_Language_Features/) ([md](04_Webviews_and_Language_Features/presentation.md)) | Tree views and `TreeDataProvider`, the Webview API and its bidirectional message channel, webview security (CSP & nonces), webview views, completion/hover/diagnostics providers, and the Language Server Protocol |
| 05 | [Testing, Packaging & Publishing](https://brendanjameslynskey.github.io/Writing_VSCode_Extensions/05_Testing_Packaging_and_Publishing/) ([md](05_Testing_Packaging_and_Publishing/presentation.md)) | Integration tests with `@vscode/test-cli`, linting and bundling, the `.vscodeignore`, packaging a `.vsix` with `vsce`, publishing to the Marketplace and Open VSX, semantic versioning, and a GitHub Actions CI/CD pipeline |

---

## Learning Arc

```
01 Foundations  ->  02 Declaring Capabilities  ->  03 The API  ->  04 Rich Experiences  ->  05 Shipping It
```

Work through the parts in order. Each one ends with a hands-on exercise and points to the next.

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | Append `?print-pdf` to the URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono

Each deck is a single self-contained `index.html` — no build step, no npm, no dependencies to install. The landing page uses Space Grotesk + Inter + JetBrains Mono.

## See also

- Series hub: [Software](https://github.com/BrendanJamesLynskey/Software) — presentations, playgrounds and reference projects.
- Companion decks in the same style: [Introduction to CI/CD](https://github.com/BrendanJamesLynskey/Introduction_to_CI_CD), [Deploying with Docker](https://github.com/BrendanJamesLynskey/Deploying_with_Docker).

## References

[VS Code Extension API](https://code.visualstudio.com/api) · [Your First Extension](https://code.visualstudio.com/api/get-started/your-first-extension) · [Contribution Points](https://code.visualstudio.com/api/references/contribution-points) · [vscode API reference](https://code.visualstudio.com/api/references/vscode-api) · [Language Server Protocol](https://microsoft.github.io/language-server-protocol/) · [Publishing Extensions](https://code.visualstudio.com/api/working-with-extensions/publishing-extension)

## License

Educational use. Code examples provided as-is.
