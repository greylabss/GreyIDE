# GreyIDE — Master Spec

> **Agent: this is one file on purpose.** Your first task (GI-000) is to split it into `/docs` using the map in §0, create the files in §8, and then replace this file with a short index that points into `/docs`.

---

## 0. Split map

| Section | Goes to |
|---|---|
| §1–2 Vision, why it beats GoLand | `docs/vision.md` |
| §3 Architecture | `docs/architecture.md` |
| §4 greyui (the GPUI port) | `docs/greyui.md` |
| §5 NO SLOP UI | `docs/ui-standards.md` |
| §6 v1 scope | `docs/v1-scope.md` |
| §7 Phases | `docs/roadmap.md` |
| §8 Multi-agent protocol | `docs/agents/protocol.md` + `AGENTS.md` + `docs/state/*` |
| §9 Setup | `docs/setup.md` |
| §10 Port map | `docs/port-map.md` (living document) |

---

## 1. What GreyIDE is

GreyIDE is a native, GPU-rendered IDE **for Go, written in Go**, built by GreyLabs. It is open source under GPL-3.0.

Go is the only language on by default. Other ecosystems come as **opt-in language packs** the user turns on themselves. The first is **Node.js/TypeScript**, for Go developers who also build frontends (Next.js and similar). With no pack enabled, none of its code loads and none of its UI appears.

It starts from Zed's architecture, ported from Rust to Go and stripped down to what a Go developer needs. It is then extended with GoLand-class Go tooling and built-in developer tools (HTTP client first).

**Pitch:** GoLand's depth, Zed's speed, written in the language it serves.

---

## 2. Why it's better than GoLand

| | GoLand | GreyIDE |
|---|---|---|
| Runtime | JVM, IntelliJ platform | Single native Go binary |
| Rendering | Java/Swing UI on the CPU | Metal, GPU-rendered, 120fps target |
| Startup and memory | Seconds to start; often GBs of RAM on large projects | Targets: under 300ms to first frame, under 200MB idle |
| Go intelligence | JetBrains reimplements Go semantics in Kotlin | gopls plus the Go toolchain's own libraries running in-process |
| Focus | General platform; Go is one plugin among many | Go first; every screen is designed for Go. Node.js/TS as an opt-in pack for full-stack Go devs |
| Built-in tools | HTTP client, DB, Docker (generic) | Same set, but Go-aware (e.g. HTTP requests generated from your router code) |
| Cost | Paid commercial license | Free, GPL-3.0 |
| Contributors | Java/Kotlin developers | Go developers, the same people who use it |

### The structural advantage

Because GreyIDE is written in Go, it can import Go's own tooling as libraries instead of reimplementing it:

- `go/ast`, `go/parser`, `go/types`, `golang.org/x/tools/go/packages`, `go/analysis`: types, packages, analyzers
- Delve (`dlv dap`): debugging
- `github.com/google/pprof/profile`: profiles parsed in-process
- `golang.org/x/vuln`: vulnerability scanning

Every Go release improves GreyIDE automatically, because the source of truth *is* its engine.

### Where GoLand still wins (be honest)

Mature refactorings, years of polish, deep database tooling, and a large plugin ecosystem. GreyIDE wins on speed, focus, design quality, and Go-native internals. The refactoring gap closes over time via gopls plus custom in-process analyzers.

---

## 3. Architecture

### 3.1 Layers (dependencies point down only)

```
cmd/greyide        entry point, wiring
features/          gotools, testing, debug, httpclient, git, terminal, tasks, (later: db, docker, grpc, pprof)
packs/             opt-in language packs: packs/node (Node.js/TypeScript) first; off by default
workspace/         panes, tabs, docks, command palette, file finder, settings, keymap, themes
editor/            buffer, rope/sum tree, display map, selections, undo, highlighting
services/          lsp (gopls), dap (delve), fs watcher, process runner, git backend
greyui/            UI framework: the GPUI port (§4)
platform/macos/    cgo + Objective-C: AppKit window, events, Metal, CoreText
```

Rule: `greyui` knows nothing about editors. `editor` knows nothing about Go. Go knowledge lives in `features/gotools` and `services`. Pack knowledge lives only in `packs/<name>`; core never imports a pack.

### 3.2 Threading model

- macOS requires AppKit on the main thread. `main` calls `runtime.LockOSThread()` in `init()`, and the main goroutine owns all UI state.
- Background work runs in goroutines and returns results through a foreground task queue (port of GPUI's foreground/background executors). Never mutate UI state off the main goroutine.
- Frame pipeline: input → update entities → build element tree → layout → paint into a scene (draw list) → **one** cgo call submits the scene to Metal.

### 3.3 Go performance rules

- Zero allocations per frame in steady state: reuse scene buffers, pool element trees per frame. Verify with `-benchmem` and `runtime/metrics`.
- One cgo crossing per frame for drawing, never one per primitive.
- Keep hot data pointer-light: slices of structs and indices, not graphs of pointers. Tune `GOGC`/`GOMEMLIMIT`.
- Rope/sum tree for buffers: O(log n) edits on huge files.
- No blocking I/O on the main goroutine, ever.

### 3.4 Performance budgets (targets; enforced in CI later)

| Metric | Target |
|---|---|
| Keystroke to frame | < 8 ms |
| Scroll/animation | 120 fps on ProMotion displays |
| Cold start to first frame | < 300 ms |
| Idle memory, medium repo | < 200 MB |
| gopls slow or busy | Typing is never blocked |

### 3.5 What to keep and strip from Zed

Verify names with `ls ~/zed/crates` before porting; Zed's crate layout changes over time.

- **Port:** `gpui`, `editor`, `text`, `rope`, `sum_tree`, `language` (Go only), `lsp`, `project`, `workspace`, `file_finder`, `command_palette`, `search`, `theme`, `settings`, terminal view, git UI, debugger UI (DAP).
- **Port, never strip (Zed fundamentals):** terminal, tasks, docks and panels, splits, command palette, project search, outline and symbol search, diagnostics panel, keymap system. See §3.7.
- **Drop:** collaboration (`collab`, `call`, `channel`, `rpc`, livekit), remote/SSH, WASM extension host, AI/assistant crates (revisit post-v1), Linux and Windows platform code (v1 is macOS only), vim mode, auto-update.
- **Languages:** core keeps Go plus config formats (go.mod, JSON, YAML, TOML, Markdown, SQL, Dockerfile, .env, .http). JavaScript, TypeScript, TSX, CSS and HTML support moves into `packs/node`, not deleted.
- **Simplify:** Zed's text buffer is a CRDT built for collaboration. Without collab, use a plain rope plus undo stack, keeping the sum tree.

### 3.6 Services

| Concern | Choice |
|---|---|
| Language server | gopls over stdio, one per workspace |
| In-process analysis | `features/gotools` on `go/packages` + `go/types`: route detection (net/http, chi, gin, echo, fiber), table-test discovery, struct tags, interface stubs |
| Debugger | Delve via DAP (`dlv dap`) |
| Syntax | tree-sitter via official Go bindings (`github.com/tree-sitter/go-tree-sitter`) + tree-sitter-go |
| Terminal | `creack/pty` + a VT parser rendered by greyui (ADR needed: port vs existing library) |
| Git | `git` CLI for writes, go-git for fast reads (ADR needed) |
| File watching | fsnotify |

### 3.7 Zed fundamentals (must survive the port)

These are what make Zed feel like Zed. Stripping or weakening any of them is a regression, not a simplification.

**Terminal**
- Real PTY shell (the user's `$SHELL`, login environment, full color and TUI apps like `htop`, `vim`, `lazygit` work).
- Terminal panel docked at the bottom, toggled with one key; also openable as a tab in the center pane.
- Multiple terminals as tabs, plus splits.
- New terminal opens in the project root, or in the folder of the focused file.
- Cmd-click on file paths and `file:line:col` in output opens them in the editor; URLs are clickable.
- Scrollback, search in terminal, copy on select optional, standard copy/paste.
- Same GPU text renderer and font settings as the editor; it must scroll at the §3.4 budget.

**Tasks**
- `tasks.json` per project: named commands (`go test ./...`, `npm run dev`) run in a terminal tab, rerunnable with one key.
- Run configurations (§6) build on tasks.

**Workspace**
- Left/right/bottom docks with panels (project tree, outline, git, terminal, test explorer, debugger).
- Pane splits in any direction, tabs per pane, drag to rearrange.
- Command palette for every action; fuzzy file finder; project-wide search and replace.
- Go to symbol (file and project), outline panel, diagnostics panel.
- Keymap: every action bindable, Zed-style key contexts, base keymap presets plus import from other editors (§3.7.1).

#### 3.7.1 Keymap presets and import

Like Zed, the keymap is a **base keymap** (a preset) plus the user's own overrides layered on top.

**Base keymap presets (ship in v1):** GreyIDE default, VS Code, Cursor, JetBrains (GoLand), Sublime Text, Zed. Switchable anytime from Settings or the palette (`keymap: select base`); switching never touches user overrides.

**First run:** one screen asks "Which editor are you coming from?" with those options. Picking one sets the base keymap and offers to import that editor's custom bindings and basic settings. One screen, skippable, no tour.

**Import custom bindings:**

| Source | Reads (macOS) |
|---|---|
| VS Code | `~/Library/Application Support/Code/User/keybindings.json` |
| Cursor | `~/Library/Application Support/Cursor/User/keybindings.json` |
| JetBrains/GoLand | the active keymap XML in the IDE's config folder (version-specific path; detect the latest installed) |
| Zed | `~/.config/zed/keymap.json` |
| Sublime Text | `Default (OSX).sublime-keymap` in the user's Packages/User folder |

Verify these paths when implementing; editors move them between versions.

**How the import works:**
- Each source has a mapping table from its command IDs to GreyIDE actions (`features/keymap/import/<source>.go`), e.g. VS Code `editor.action.rename` → `editor: rename`.
- VS Code `when` clauses translate to GreyIDE key contexts where an equivalent exists.
- The import writes to the user's keymap file as a clearly marked block; it never overwrites existing user bindings without asking.
- Afterwards, one summary: how many bindings were imported, plus a list of the ones that couldn't be mapped (unknown command, or no GreyIDE equivalent yet), each with its original key and command. No silent drops.
- Unmapped commands are logged to `docs/state/keymap-gaps.md` during development so agents know which actions are missing.

**Settings import (same flow, optional):** theme preference (dark/light), editor font family and size, tab size, format on save, word wrap. Anything without an equivalent is listed, not guessed.

### 3.8 Language packs

A pack is a self-contained module under `packs/<name>` that registers grammars, language servers, debug adapters, task templates and run-config types through a small interface in core.

- **Off by default.** The user enables a pack in Settings → Language Packs, or from the palette (`packs: enable node`).
- **Suggested, never forced.** If a `package.json` is detected and the Node pack is off, show one quiet, dismissible line in the status bar offering to enable it. Never a modal, never repeated after dismissal.
- **Zero cost when off.** A disabled pack spawns no processes, loads no grammars, and adds no UI.
- **Dependencies are explicit.** A pack checks for its external tools (e.g. `node`) and says plainly what is missing; it doesn't silently install runtimes.

**`packs/node` (Node.js/TypeScript) contents**

| Concern | Choice |
|---|---|
| Languages | JavaScript, TypeScript, JSX/TSX, CSS, HTML, JSON with comments |
| Syntax | tree-sitter grammars for each |
| Language server | TypeScript language server (vtsls or typescript-language-server; ADR) plus ESLint and Tailwind CSS language servers when the project uses them |
| Formatting | Prettier or Biome, whichever the project is configured for |
| Package managers | npm, pnpm, yarn, bun, detected from the lockfile |
| Tasks | `package.json` scripts appear as tasks automatically (`dev`, `build`, `test`) |
| Debugging | Node via the JavaScript debug adapter (DAP) |
| Framework awareness | Next.js: run `dev` from a task, route files visible in the outline; deeper support post-v1 |

The Go HTTP client works against Node backends too; route detection is Go-only at first.

---

## 4. greyui — the GPUI port

### 4.1 What GPUI is (preserve this essence)

- **Hybrid model:** views re-render an element tree every frame; state lives in retained entities.
- **Entities:** app-owned state behind handles, with observe/notify/subscribe/emit.
- **Layout:** flexbox (Zed uses Taffy).
- **Styling:** Tailwind-like builder API.
- **Scene:** primitives batched by kind (quads with borders and radii, shadows, paths, underlines, glyph sprites, image sprites) and drawn with instanced draws.
- **Text:** platform shaping (CoreText on macOS) into a glyph atlas texture.
- **Actions and keymap:** dispatch tree with key contexts.

### 4.2 Go design

- `greyui.Entity[T]`: generic handle (ID) into an app-owned store. Mutate via `cx.Update(e, func(*T, *Context))`. No pointer sharing across goroutines.
- Element API: fluent builders, e.g. `ui.Div().Flex().Gap(2).Child(...)`. Elements are plain structs allocated from a per-frame arena.
- Layout: port Taffy's flexbox subset, or write a minimal flexbox (ADR).
- Renderer: Metal through cgo and an Objective-C shim in `platform/macos`; MSL shaders ported from Zed's Metal shaders.
- Text: CoreText via cgo (pixel-identical to native macOS text). Portable fallback later: `go-text/typesetting`.
- Prior art to study: Gio (gioui.org) and Cogent Core, both GPU UI in Go.

### 4.3 Port order inside greyui

1. Window and event loop
2. Metal clear color
3. Quads (fill, border, radius)
4. Text
5. Input and focus
6. Layout
7. Entities and actions
8. Virtualized lists and scrolling
9. Overlays and popovers

Each step ends with a runnable demo in `examples/`.

---

## 5. NO SLOP UI — this is law

**Slop** is the generic look of AI-generated interfaces: decoration without purpose, inconsistency, and visual noise. GreyIDE's UI must feel like a precise instrument. Reference standard: Zed. If a screen would look at home in a generic AI dashboard template, it fails.

### 5.1 Banned

- Gradients anywhere in chrome. Glassmorphism, blur, glow, neon.
- Purple/blue "AI" accents, or more than one accent hue.
- Emoji as icons, or anywhere in UI copy. Sparkle icons.
- Every panel as a floating rounded card with a drop shadow.
- Shadows on anything that isn't floating (menus, palette, popovers only).
- Mixed corner radii, mixed icon sets, mixed stroke widths.
- Oversized padding and airy "landing page" spacing in a tool.
- Bounce, spring or decorative animation. Animating layout on resize or scroll.
- Toasts for routine events, badges everywhere, modal dialogs for non-destructive actions.
- Illustrations in empty states. Lorem ipsum. "Oops!", exclamation marks, cute copy.
- Hardcoded colors or sizes outside the token files.

### 5.2 Rules

- **Tokens only.** Every color, size, radius and duration comes from `workspace/theme` tokens. Themes are token files.
- **Color:** neutral grays carry the UI. One accent hue, used only for focus, selection and the primary action. Semantic colors (error, warning, info, success) only for diagnostics and status. WCAG AA contrast minimum. Dark theme is the default; light theme ships in v1.
- **Typography:** one UI sans at 13px (12px secondary), one monospace for code at 14px default with 1.5 line height. Sentence case everywhere. Font choice is an ADR; candidates: IBM Plex Sans/Mono, Inter + Geist Mono.
- **Spacing:** 4px grid. Allowed steps: 2, 4, 6, 8, 12, 16, 24.
- **Radius:** 4px for controls, 6px for floating layers, 0 for panels. Panels sit flush and are separated by 1px borders.
- **Density:** list rows 24px, tabs 32px, status bar 24px. Dense but never cramped.
- **Icons:** one monochrome set, 16px, consistent stroke. Icons supplement labels, never replace meaning.
- **Motion:** ≤150ms, ease-out, only for overlays appearing or disappearing. Respect reduced-motion.
- **Keyboard first:** every action is a command in the palette and can be bound. Focus is always visible.
- **Empty states:** one line of text plus one action.
- **Errors:** inline, next to their cause, in plain words.

### 5.3 Review gate (every UI change)

1. Screenshot of the change in dark and light themes attached to the commit or PR notes.
2. Side-by-side with the equivalent Zed screen where one exists.
3. Passes every item in §5.1 and §5.2.
4. No new token added without updating `docs/ui-standards.md`.

If an agent is unsure whether something is slop, it is slop. Remove it.

---

## 6. v1 scope

### In v1

- Editor: rope buffer, multi-cursor, tree-sitter highlighting, folding, minimap off by default
- Workspace: file tree, tabs, splits, command palette, fuzzy file finder, project search
- Go intelligence via gopls: completion, diagnostics, hover, go-to-definition, references, rename, code actions, format on save (gofmt/goimports)
- Go actions: generate struct tags, implement interface, fill struct, generate test
- Run configurations (args, env, build tags, working directory)
- Test explorer: run/debug single tests and table-test subcases, inline coverage gutters
- Terminal and tasks, with every Zed fundamental in §3.7
- Language pack system (§3.8), with no packs shipped yet
- Git basics: status, diff gutter, stage, commit, branch switch
- Debugger: Delve with breakpoints, stepping, variables, goroutine list
- HTTP client: `.http` files, environments, response viewer, request generated from detected routes
- Settings and keymap (JSON), dark and light themes
- Base keymap presets and import from VS Code, Cursor, JetBrains, Zed and Sublime (§3.7.1)

### Not in v1

Node.js pack (first release after v1), DB client, Docker, gRPC client, pprof and trace viewers, concurrency visualizer, TinyGo/embedded mode, Linux/Windows, third-party plugins, AI features, vim mode, collaboration.

### v1 definition of done

Opens a real medium-size Go repo, hits every §3.4 budget, passes the §5.3 review gate on every screen, and a Go developer can spend a full working day in it without opening GoLand.

---

## 7. Phases

| Phase | Goal | Exit criterion |
|---|---|---|
| P0 | Setup, docs split, agent protocol | GI-000 done; `go build ./...` passes |
| P1 | greyui foundations (§4.3 steps 1–5) | Window with crisp text and input |
| P2 | greyui complete (§4.3 steps 6–9) | Demo app: scrolling list, popover, keymap |
| P3 | Editor core | Edit a 100k-line file at budget |
| P4 | Workspace shell | File tree, tabs, splits, palette, finder |
| P5 | Go intelligence | gopls features + `gotools` actions |
| P6 | Terminal, tasks, run, test | Every §3.7 terminal item works; test explorer with coverage |
| P7 | Git + debugger | Delve session end to end |
| P8 | HTTP client | Route detection to request to response |
| P9 | Polish and budgets | v1 definition of done met |

After v1: **Node.js pack** → DB client → Docker → pprof/trace → concurrency visualizer → gRPC → TinyGo mode → Linux.

---

## 8. Multi-agent protocol

Multiple agents (Claude, Codex, Muse, others) work on this repo, sometimes in parallel, and can be swapped mid-task. **The repo is the only memory.** Nothing important may live only in an agent's chat.

### 8.1 Files

```
AGENTS.md                 entry point for every agent (rules + read order)
CLAUDE.md -> AGENTS.md    symlink so Claude Code picks it up
docs/state/STATE.md       where the project is right now; the one "continue from here" file
docs/state/TASKS.md       task ledger
docs/state/LOG.md         append-only session log
docs/decisions/ADR-NNN-*.md   architecture decisions
docs/port-map.md          Zed source → GreyIDE package, with status
```

Codex reads `AGENTS.md` natively; Claude Code reads `CLAUDE.md` (symlinked). For any other agent, the first prompt is: *"Read AGENTS.md and follow it."*

### 8.2 STATE.md format

```
# State
Phase: P1 greyui foundations
Active tasks: GI-014 (agent: codex, branch: gi-014-metal-quads)
Last verified build: <commit hash>, <date>

## Continue from here
GI-014: quad pipeline renders fills; borders not started.
Next step: add border width/color to quad shader (platform/macos/shaders.metal).
Run: go run ./examples/quads

## Blockers / open questions
- ADR-003 font choice pending
```

### 8.3 TASKS.md format

| ID | Title | Status | Agent | Branch | Depends on |
|---|---|---|---|---|---|
| GI-014 | Metal quad pipeline | in progress | codex | gi-014-metal-quads | GI-012 |

Statuses: `todo`, `in progress`, `blocked`, `review`, `done`. One task per agent at a time.

### 8.4 Session protocol

**Start:**
1. Read `AGENTS.md` → `docs/state/STATE.md` → the active task's row and its last `LOG.md` entry.
2. Run `git log --oneline -15` and `git status`. **If STATE.md and git disagree, git wins**; fix STATE.md first.
3. Claim the task in `TASKS.md` (set Agent and status).

**During:**
- Small commits, every message prefixed with the task ID: `GI-014: add quad border uniforms`.
- Unfinished work is committed as `GI-014: wip — <what is half-done>` rather than left uncommitted.
- Any non-trivial design choice gets an ADR before code.
- Porting updates `docs/port-map.md` row status in the same commit.

**Before stopping (or when context is running low, or when told to hand off):**
1. Commit everything.
2. Update "Continue from here" in STATE.md: exact next step, the file to open, the command to run.
3. Append to LOG.md:

```
## 2026-11-02 14:30 — codex — GI-014
Done: quad fill pipeline; examples/quads renders.
Not done: borders, radius.
Next: border uniforms in shaders.metal, then radius SDF.
Files: platform/macos/metal.m, greyui/scene/quad.go
Commits: a1b2c3d..e4f5g6h
```

A task is never handed off with a failing build unless LOG.md says exactly what is broken and why.

### 8.5 Parallel agents

- Each agent works on its own branch in its own git worktree (`git worktree add ../greyide-gi-014 gi-014-metal-quads`).
- Two agents never own tasks in the same package at the same time; the `Depends on` column prevents collisions.
- Merging to `main` requires a green build and an updated STATE.md.

---

## 9. Setup

The `greyide` GitHub repo is a fork of Zed. It holds two branches:

| Branch | Contents | Local folder | Who writes |
|---|---|---|---|
| `main` | GreyIDE only (orphan branch, no Zed history) | `~/greyide` | Agents |
| `zed-upstream` | Zed's original code | `~/zed` (git worktree) | Nobody; read-only reference |

One-time setup (already done by the owner; agents don't repeat it):

```bash
cd ~/greyide
git branch -m main zed-upstream
git checkout --orphan main
git rm -rf .
git add GREYIDE.md && git commit -m "GI-000: GreyIDE spec"
git worktree add ~/zed zed-upstream
git push -f origin main && git push origin zed-upstream
```

GI-000 then scaffolds `main`:

```bash
go mod init github.com/greylabs/greyide   # change to the real org path
mkdir -p cmd/greyide greyui platform/macos editor workspace services features packs examples \
         docs/state docs/decisions docs/agents
ln -s AGENTS.md CLAUDE.md
```

**Agent rules for the reference:** read Zed at `~/zed/...` by path. Never commit to `zed-upstream`, never merge it into `main`, never copy Zed's agent instruction files into `main`. To update the reference: `git -C ~/zed pull https://github.com/zed-industries/zed.git main`.

Requirements: macOS on Apple Silicon, Go (latest stable), Xcode Command Line Tools (cgo, Metal shader compiler).

**License:** GreyIDE is GPL-3.0 because it ports code from Zed's GPL crates. Keep Zed's copyright notices in ported files' headers. GPUI is Apache-2.0, so `greyui` could later be split out under Apache-2.0 as a standalone Go UI library, if it contains no GPL-derived code.

---

## 10. Port map (seed — agents extend this)

| Zed source (`~/zed/crates/…`) | GreyIDE package | Phase | Status |
|---|---|---|---|
| `gpui` (platform/mac) | `platform/macos` | P1 | todo |
| `gpui` (scene, renderer, shaders) | `greyui/scene`, `platform/macos` | P1 | todo |
| `gpui` (text system) | `greyui/text` | P1 | todo |
| `gpui` (elements, style, layout) | `greyui/element`, `greyui/layout` | P2 | todo |
| `gpui` (app, entities, executors, keymap) | `greyui/app` | P2 | todo |
| `sum_tree`, `rope` | `editor/sumtree`, `editor/rope` | P3 | todo |
| `text`, `editor` (minus CRDT) | `editor` | P3 | todo |
| `language` (Go only) | `editor/syntax` | P3 | todo |
| `workspace`, `file_finder`, `command_palette`, `search` | `workspace/...` | P4 | todo |
| `theme`, `settings` | `workspace/theme`, `workspace/settings` | P4 | todo |
| `lsp`, `project` | `services/lsp`, `services/project` | P5 | todo |
| terminal crates | `features/terminal` | P6 | todo |
| task crates | `features/tasks` | P6 | todo |
| keymap files (`assets/keymaps/*`, base keymap presets) | `features/keymap`, `features/keymap/import` | P4 | todo |
| JS/TS/CSS/HTML language support | `packs/node` | post-v1 | todo |
| git crates | `services/git`, `features/git` | P7 | todo |
| debugger crates (DAP) | `services/dap`, `features/debug` | P7 | todo |
| — (new) | `features/gotools`, `features/testing`, `features/httpclient` | P5–P8 | todo |

Statuses: `todo`, `porting`, `ported`, `partial`, `skipped (reason)`.