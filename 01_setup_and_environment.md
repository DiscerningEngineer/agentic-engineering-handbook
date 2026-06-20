# Chapter 01 -- Setup and Environment

**TL;DR.** The way in is the native installer (`curl -fsSL https://claude.ai/install.sh | bash`), which auto-updates in the background and drags no Node runtime behind it. Authentication runs off any paid Claude plan (Pro, Max, Team, Enterprise) or an Anthropic Console account billed per token; the free tier is not one of the doors. Configuration lives in a four-scope `settings.json` cascade, managed over local over project over user, where managed always wins, arrays merge, and scalars override. The CLI is the whole product. The VS Code and JetBrains integrations layer native diffs, selection context, and diagnostic sharing on top of it. Once you are fluent you keep your config in git, run several sessions at once, customize the statusline to watch context, cost, and rate limits, and reach for output styles when you want to reshape the system prompt. When things go sideways the loop is `/context`, `/status`, `/doctor`, and `claude --safe-mode`. All of it is version-sensitive: Claude Code ships close to daily (the latest at time of writing was v2.1.181, 2026-06-17), so check any version-pinned claim against the [changelog](https://code.claude.com/docs/en/changelog) before you lean on it.

---

## 1. Choosing an install method

Anthropic ships Claude Code through several channels, and which one fits you turns on three questions: whether you want background auto-updates, how your org hands software to its people, and whether you are pinning versions so a build stays reproducible. [^1]

| Method | Command | Auto-updates? | Needs Node? | Use when |
|--------|---------|---------------|-------------|----------|
| **Native (recommended)** | `curl -fsSL https://claude.ai/install.sh \| bash` (macOS/Linux/WSL); `irm https://claude.ai/install.ps1 \| iex` (PowerShell) | Yes (background) | No | Default for individuals |
| **Homebrew** | `brew install --cask claude-code` (stable) or `claude-code@latest` | No (manual `brew upgrade`) | No | macOS users who manage everything via brew |
| **WinGet** | `winget install Anthropic.ClaudeCode` | No (manual) | No | Windows package-managed fleets |
| **apt / dnf / apk** | signed repos at `downloads.claude.ai` | No (system upgrade flow) | No | Linux servers, CI images |
| **npm** | `npm install -g @anthropic-ai/claude-code` | No (`npm install -g ...@latest`) | Yes (Node 18+) | You already manage CLIs via npm |

[^1]

**System requirements:** macOS 13.0+, Windows 10 1809+ or Server 2019+, Ubuntu 20.04+, Debian 10+, or Alpine 3.19+; 4 GB+ RAM; x64 or ARM64; Bash, Zsh, PowerShell, or CMD; an internet connection. `ripgrep` usually ships bundled and powers search. [^1]

### 1.1 The native installer is "the same binary," not a wrapper

Here is the detail that separates the people who understand the tool from the people who copy the install line. The npm package and the native installer put the same native binary on your machine. npm just gets there the long way around, pulling it in through a per-platform optional dependency (`@anthropic-ai/claude-code-darwin-arm64` and its siblings) and a postinstall step that links it into place. The docs are explicit: "The installed `claude` binary does not itself invoke Node." That is the whole argument for the native install. You get the binary without dragging a Node runtime along behind it, and without npm's global-prefix permission headaches following you around. [^1]

> **Migration note.** If you previously ran `npm install -g @anthropic-ai/claude-code` and the npm global directory is not writable, background auto-update cannot run; Claude Code shows a one-time startup notice and `claude doctor` lists the fixes. The clean path forward is the native installer. The official docs frame native as recommended and continue to document npm as supported, so "npm is dead" is community framing, not an Anthropic deprecation. [^1]

### 1.2 Windows: native vs WSL

On Windows the choice comes down to where your projects live and which features you actually need. [^1]

- **Native Windows** runs the installer from PowerShell or CMD with no admin needed. [Git for Windows](https://git-scm.com/downloads/win) is optional but recommended: it gives Claude Code Git Bash for the Bash tool. Without it, Claude Code runs shell commands through the PowerShell tool. Sandboxing is not supported on native Windows.
- **WSL 2** is required if you want Linux toolchains or sandboxed command execution. Install and launch `claude` inside the WSL terminal, not from PowerShell. (WSL 1 works but, like native Windows, does not support sandboxing.)

When Git for Windows is installed but not auto-detected, point at it explicitly. [^1]

```json
{ "env": { "CLAUDE_CODE_GIT_BASH_PATH": "C:\\Program Files\\Git\\bin\\bash.exe" } }
```

A PowerShell tool is rolling out as an additional shell option even when Git Bash is present; opt in with `CLAUDE_CODE_USE_POWERSHELL_TOOL=1`. [^1]

### 1.3 Verifying and updating

```bash
claude --version     # confirm install
claude doctor        # deeper health check + result of the last update attempt
claude update        # apply an update now instead of waiting for the background check
```

Native installs check for updates on startup and periodically while running, download in the background, and apply on the next launch. Two settings govern how that behaves. [^1]

```json
{
  "autoUpdatesChannel": "stable",   // "latest" (default) or "stable" (~1 week behind, skips major regressions)
  "minimumVersion": "2.1.100"        // floor: refuse to install below this
}
```

- `autoUpdatesChannel: "stable"` is a version "typically about one week old, skipping releases with major regressions." It trades freshness for fewer surprises, a sane default for teams who cannot afford a bad day from a near-daily release. Homebrew picks the channel by cask name instead: `claude-code` tracks stable, `claude-code@latest` tracks latest. [^1]
- To stop background updates while keeping manual ones, set `"env": { "DISABLE_AUTOUPDATER": "1" }`; `claude update` and `claude install` still work. To block every update path including manual, set `DISABLE_UPDATES` instead. [^1]
- For Homebrew and WinGet, set `CLAUDE_CODE_PACKAGE_MANAGER_AUTO_UPDATE=1` to have Claude Code run the upgrade command for you in the background. apt, dnf, and apk still require a manual upgrade because those need elevated privileges. [^1]

> **Enterprise tip.** `minimumVersion` only constrains updates; it will not downgrade you. To make Claude Code refuse to start outside a version range, use the managed-only `requiredMinimumVersion` and `requiredMaximumVersion`. Both matter to regulated-industry fleets that must pin a vetted version. [^1]

### 1.4 Supply-chain verification

Each release publishes a `manifest.json` of SHA256 checksums for every platform binary. The manifest is GPG-signed, so verifying its signature transitively verifies every binary it lists. The release signing key fingerprint is `31DD DE24 DDFA B679 F42D 7BD2 BAA9 29FF 1A7E CACE` (`security@anthropic.com`). [^1]

```bash
curl -fsSL https://downloads.claude.ai/keys/claude-code.asc | gpg --import
gpg --fingerprint security@anthropic.com   # confirm the fingerprint above
# then, with VERSION set, from $REPO=https://downloads.claude.ai/claude-code-releases:
#   curl -fsSLO "$REPO/$VERSION/manifest.json"
#   curl -fsSLO "$REPO/$VERSION/manifest.json.sig"
gpg --verify manifest.json.sig manifest.json
```

A valid result reports `Good signature from "Anthropic Claude Code Release Signing <security@anthropic.com>"`. The `WARNING: This key is not certified with a trusted signature!` line is expected for a freshly imported key; the fingerprint check is what authenticates it. macOS binaries are additionally signed by "Anthropic PBC" and notarized (`codesign --verify --verbose ./claude`); Windows by "Anthropic, PBC" (`Get-AuthenticodeSignature .\claude.exe`). Linux binaries are not individually code-signed, so verify via the manifest, or trust the apt/dnf/apk repo signing key, which your package manager checks automatically. Manifest signatures exist from v2.1.89 onward; earlier releases ship checksums without a detached signature. [^1]

> **Alpine/musl.** The native installer on musl-based distros needs `libgcc`, `libstdc++`, and a system `ripgrep` (`apk add libgcc libstdc++ ripgrep`), then `"env": { "USE_BUILTIN_RIPGREP": "0" }`. [^1]

---

## 2. Authentication

Claude Code wants one of two things from you: a paid Claude subscription (Pro, Max, Team, or Enterprise), or an Anthropic Console account billed per token at API rates. The free Claude.ai plan does not include Claude Code. You can also route through Amazon Bedrock, Google Vertex AI, or Microsoft Foundry by setting the provider's environment variables before you launch. [^2]

- **Interactive login.** Run `claude` and follow the browser prompt. If the browser cannot reach the local callback server, which is common in WSL2, SSH sessions, and containers, the screen shows a code to paste back into the terminal. Use `/logout` to switch accounts. [^2]
- **Subscription vs Console** is a real decision with consequences. Subscriptions bill against rolling rate-limit windows that the statusline can surface as 5-hour and 7-day usage (section 6); Console bills per token. Pick based on how you want cost to behave when you are deep in a session and not watching the meter.
- **Do not quote prices as fixed facts.** Plan tiers and per-token rates drift. Send readers to the live pages: [claude.com/pricing](https://claude.com/pricing) and the [platform pricing docs](https://platform.claude.com/docs/en/about-claude/pricing).

> **CI / headless auth.** For unattended runs, the cleanest path is a long-lived OAuth token: `claude setup-token` walks you through authorization and prints a one-year token (subscription-scoped, inference-only) that you export as `CLAUDE_CODE_OAUTH_TOKEN`. For rotating credentials, `apiKeyHelper` runs a shell script that returns a key, re-called after 5 minutes or on a 401, or on the interval set by `CLAUDE_CODE_API_KEY_HELPER_TTL_MS`. Bedrock and Vertex use `awsAuthRefresh` / `gcpAuthRefresh` to keep cloud creds fresh. [^2] [^3]

> **The API-key footgun.** When `ANTHROPIC_API_KEY` is set in your shell, it takes precedence over your subscription once approved, which fails confusingly if that key belongs to a disabled org. `unset ANTHROPIC_API_KEY` to fall back to the subscription, and check `/status` to confirm which method is active. [^2]

---

## 3. The settings.json scope cascade

If you internalize one mental model from this chapter, make it this one. Claude Code reads several `settings.json` files and merges them, and the cascade is the machinery that lets you say: team rules in git, personal tweaks gitignored, org policy unbreakable. Get the cascade and most of your "why won't this setting take" mysteries dissolve before they start.

### 3.1 The four scopes and their paths

| Scope | Path | Committed? | Answers |
|-------|------|-----------|---------|
| **Managed** (policy) | macOS: `/Library/Application Support/ClaudeCode/`; Linux/WSL: `/etc/claude-code/`; Windows: `C:\Program Files\ClaudeCode\` (file: `managed-settings.json`) | Deployed by IT | "What does the org mandate?" |
| **User** | `~/.claude/settings.json` | No (your machine) | "What do I want across all repos?" |
| **Project** | `.claude/settings.json` | **Yes** (git) | "What does this team agree on?" |
| **Local** | `.claude/settings.local.json` | No (gitignored) | "What's just for me in this repo?" |

[^3]

Managed settings support a `managed-settings.d/` drop-in directory for modular policy fragments, following the systemd convention: `managed-settings.json` merges first as the base, then every `*.json` in the drop-in directory merges on top in alphabetical order (numeric prefixes like `10-telemetry.json`, `20-security.json` control the order; dotfiles are ignored). This is how a fleet keeps telemetry policy and security policy in separate, separately-owned files. [^3]

### 3.2 Precedence and merge semantics

When the same key shows up in more than one scope, precedence runs highest first: [^3] [^4]

1. **Managed** -- always wins; user, project, and local cannot override it
2. **Command-line flags** (e.g. `--model`, `--permission-mode`) and environment variables
3. **Local** (`.claude/settings.local.json`)
4. **Project** (`.claude/settings.json`)
5. **User** (`~/.claude/settings.json`)

The subtlety that trips everyone is the merge rule itself: scalars override, arrays merge, objects deep-merge. The `permissions.allow`/`deny`/`ask` arrays concatenate (and dedupe) across scopes, so a project can extend your personal allowlist rather than clobber it. A scalar like `model` is winner-take-all from the highest scope that sets it. That asymmetry is precisely why a `model` set in `settings.local.json` quietly beats the one in your user settings, and it is the single most common reason a setting seems to be getting ignored. (One exception worth knowing: `fallbackModel` does not merge; the highest-precedence file supplies the entire chain.) [^3] [^4]

> **Two different files, often confused.** `~/.claude/settings.json` holds your configuration: permissions, hooks, env, model. `~/.claude.json` holds app state: MCP server records, UI toggles, per-project caches. Putting `permissions`, `hooks`, or `env` in `~/.claude.json` silently does nothing. [^4]

### 3.3 A starter user `settings.json`

Always include the schema reference. It is the line that turns VS Code, and any JSON-schema-aware editor, into a tool that catches your typos before they cost you a debugging session. [^3]

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "model": "opus",
  "autoUpdatesChannel": "stable",
  "permissions": {
    "allow": ["Bash(npm run lint)", "Bash(npm run test:*)", "Read(~/.zshrc)"],
    "deny":  ["Read(./.env)", "Read(./.env.*)", "Read(./secrets/**)"]
  },
  "outputStyle": "Default",
  "cleanupPeriodDays": 30
}
```

High-value fields worth knowing, all documented on the settings page: `model`, `permissions`, `env`, `hooks`, `statusLine`, `outputStyle`, `cleanupPeriodDays` (transcript retention, default 30, min 1), `autoUpdatesChannel` / `minimumVersion`, `apiKeyHelper`, `attribution` (git commit and PR trailers), `claudeMdExcludes` (glob patterns to skip loading CLAUDE.md files), `editorMode` (`normal` or `vim`), `fileCheckpointingEnabled` (snapshots for `/rewind`, default on), and `enabledMcpjsonServers` / `disabledMcpjsonServers`. The full key list churns release to release; treat the live settings page as ground truth and run `/doctor` to catch invalid keys before you commit. [^3]

> **Managed settings now parse tolerantly (v2.1.169+).** An invalid entry is stripped with a warning rather than failing the whole file, so a single typo cannot disable an organization's entire policy. Security-enforcement fields fail closed per-field: a broken `allowedMcpServers` enforces only its valid subset, a broken `enforceAvailableModels` is treated as `true`. Useful to know; not an excuse to ship typos. [^3]

### 3.4 The `.claude/` directory, at a glance

```
~/.claude/
  settings.json          # user config
  CLAUDE.md              # user-global memory (see Ch. 03)
  agents/                # user subagents (see Ch. 08)
  skills/                # user skills
  output-styles/         # user output styles
  statusline.sh          # if you use a custom statusline
~/.claude.json           # app state: MCP records, UI toggles, caches

<repo>/.claude/
  settings.json          # project config (committed)
  settings.local.json    # personal overrides (gitignored)
  CLAUDE.md              # project memory (committed)
  agents/  skills/  commands/  output-styles/
<repo>/.mcp.json         # project MCP servers (committed, at repo root -- NOT inside .claude/)
```

[^1] [^4]

> **Gotcha worth memorizing:** project MCP config goes at the repo root as `.mcp.json`, not under `.claude/`, and not as an `mcpServers` key inside `settings.json` (which is never read). Both are top entries in the "MCP servers never load" troubleshooting table. [^4]

---

## 4. Terminal environment

Claude Code is a terminal-first product, and the IDE integrations in section 5 wrap the same engine. A handful of environment details quietly pay for themselves.

### 4.1 Terminal choice, hyperlinks, and notifications

- **OSC 8 clickable links** (used by custom statuslines and the footer badges) require a terminal that supports hyperlinks: iTerm2, Kitty, and WezTerm are the auto-detected ones. macOS Terminal.app does not support them. If link text appears but is not clickable, common on Windows Terminal, launch with `FORCE_HYPERLINK=1 claude`. [^5]
- **System notifications** are how you run several sessions at once without standing over each. The notification channel is configurable via `preferredNotifChannel`, whose documented values include `terminal_bell`, `iterm2`, `iterm2_with_bell`, `kitty`, and `ghostty`. This is the difference between babysitting one session and letting five run while you watch for the one that wants you. [^3]

### 4.2 Shell integration knobs

- **`/ide`** connects an external terminal session to a running VS Code or JetBrains IDE so diffs open natively and selections and diagnostics flow in. [^6]
- **Vim mode** is available for the prompt editor (`editorMode: "vim"`); the current mode is exposed to the statusline as `vim.mode`. [^5]

### 4.3 Useful environment variables

Set these in the shell or under the `env` key of `settings.json` (the `env` block is the portable, committed-with-the-project way). [^3]

| Variable | Effect |
|----------|--------|
| `ANTHROPIC_MODEL` | Override the model for the session |
| `DISABLE_AUTOUPDATER` / `DISABLE_UPDATES` | Stop background updates / all updates |
| `CLAUDE_CODE_GIT_BASH_PATH` | Tell native Windows where Git Bash is |
| `CLAUDE_CODE_USE_POWERSHELL_TOOL` | Opt in/out of the PowerShell shell tool on Windows |
| `USE_BUILTIN_RIPGREP=0` | Use system ripgrep (required on musl) |
| `FORCE_HYPERLINK=1` | Force OSC 8 link support when auto-detection misses |
| `CLAUDE_CONFIG_DIR` | Point at an alternate config dir (the basis of the clean-room repro in section 8) |
| `CLAUDE_CODE_API_KEY_HELPER_TTL_MS` | Refresh interval for `apiKeyHelper` |
| `CLAUDE_CODE_PACKAGE_MANAGER_AUTO_UPDATE=1` | Let Claude Code run Homebrew/WinGet upgrades for you |

The full env-var roster is large and version-sensitive; the authoritative list is `code.claude.com/docs/en/env-vars`. [^1] [^3]

---

## 5. IDE integrations

The CLI is the complete product. The IDE plugins give you a graphical surface (native diffs, selection context, diagnostic sharing) but they expose a subset of what the CLI does. When you need a CLI-only feature, run `claude` in the IDE's integrated terminal and you have everything again. [^7]

### 5.1 VS Code (and forks: Cursor, Kiro, and others)

**Install:** Extensions view (`Cmd/Ctrl+Shift+X`), search "Claude Code," Install, or use the direct install links for VS Code and Cursor. Other forks (the docs name Kiro and Devin Desktop) search the same way or pull from the [Open VSX registry](https://open-vsx.org/extension/Anthropic/claude-code). Requires VS Code 1.98.0 or higher. The extension bundles a private copy of the CLI for its chat panel; it does not add `claude` to your PATH, so to run `claude` in the integrated terminal you still need the [standalone CLI install](https://code.claude.com/docs/en/setup). [^7]

**What it adds:** [^7]

- **Inline side-by-side diffs** with accept/reject. Edit the proposed content in the diff before accepting and "Claude is told that you modified it so it does not assume the file matches its original proposal."
- **Automatic selection context.** Highlighted code is sent with your prompt (the footer shows the line count); toggle the eye-slash to hide it, or exclude a sensitive file with a `Read` deny rule, which suppresses both the selection and the open-file notice.
- **@-mentions** with fuzzy matching and line ranges (`@app.ts#5-10`); trailing slash for folders. For large PDFs you can ask Claude to read specific pages.
- **Plan mode rendered as an editable markdown document** where you add inline comments before Claude executes.
- **Checkpoints / rewind** -- fork the conversation, rewind code, or both (see Ch. 04).
- **Multiple conversations** in tabs or windows, with history shared with the CLI: `claude --resume` in the terminal continues an extension conversation.

**Key shortcuts:** [^7]

| Action | Shortcut |
|--------|----------|
| Toggle focus editor / Claude | `Cmd+Esc` / `Ctrl+Esc` |
| Open conversation in new tab | `Cmd+Shift+Esc` / `Ctrl+Shift+Esc` |
| Insert @-mention of current selection | `Option+K` / `Alt+K` |
| Expand/collapse all thinking blocks | `Ctrl+O` |
| Multi-line input (no send) | `Shift+Enter` |

**Extension-only settings** live in VS Code settings under Extensions -> Claude Code, distinct from `~/.claude/settings.json`: `useTerminal` (CLI-style panel), `initialPermissionMode`, `preferredLocation` (sidebar vs panel tab), `autosave`, `useCtrlEnterToSend`, `respectGitIgnore`, and `claudeProcessWrapper` (point at a separately-installed `claude` binary if your platform lacks a bundled one). Shared config (permissions, hooks, MCP, env) still lives in `~/.claude/settings.json`. [^7]

> **The hidden `ide` MCP server.** When the extension is active it runs a local MCP server (bound to `127.0.0.1` on a random high port, with a fresh per-activation auth token in a `0600` lock file under `~/.claude/ide/`) that the CLI auto-connects to for diffs, selection reads, and Jupyter execution. It is named `ide` and hidden from `/mcp`. The server hosts about a dozen tools but exposes only two to the model: `mcp__ide__getDiagnostics` (read-only) and `mcp__ide__executeCode` (runs a Jupyter cell). You need to know they exist if your org uses a `PreToolUse` hook to allowlist MCP tools. `executeCode` always shows a native confirmation in VS Code before running a cell, separate from any hook allowlist. [^7]

> **macOS Tahoe gotcha.** On Tahoe and later the system Game Overlay binds `Cmd+Esc`, swallowing the "focus Claude" shortcut. Clear it in System Settings -> Keyboard -> Keyboard Shortcuts -> Game Controllers, or rebind via VS Code's keybindings editor. [^7]

> **Usage panel (v2.1.174+).** `/usage` in the extension opens an Account and usage dialog that breaks down what is driving your plan limits, flagging behaviors that account for 10%+ of recent usage (cache misses, long context, subagent-heavy sessions) with attribution tables per skill, subagent, plugin, and MCP server. [^7]

### 5.2 JetBrains (IntelliJ, PyCharm, WebStorm, GoLand, PhpStorm, Android Studio)

JetBrains plays it differently from VS Code. The plugin does not bundle the CLI. It runs `claude` in the integrated terminal and connects to it, so you install both the [CLI](https://code.claude.com/docs/en/setup) and the [Marketplace plugin](https://plugins.jetbrains.com/plugin/27310-claude-code-beta-), then restart the IDE (sometimes more than once). [^6]

**Features:** quick launch (`Cmd+Esc` / `Ctrl+Esc`), IDE diff viewer, automatic selection and tab context sharing (blockable per file with `Read` deny rules), file-reference insertion (`Cmd+Option+K` / `Alt+Ctrl+K`, producing `@src/auth.ts#L1-99`), and automatic diagnostic sharing for lint and syntax errors. Set the diff tool to `auto` (IDE) or `terminal` via `/config`. [^6]

**Caveats worth flagging for the cohort:** the plugin is labeled [Beta]; Remote Development requires installing the plugin on the remote host, not the local client; WSL2 users frequently hit "No available IDEs detected" from NAT or firewall blocking, fixed with a Windows Firewall rule for the WSL subnet or `networkingMode=mirrored` in `.wslconfig` (mirrored mode needs Windows 11 22H2+); and the ESC key may need terminal-keybinding reconfiguration (Settings -> Tools -> Terminal) to interrupt Claude. [^6]

### 5.3 Choosing CLI vs IDE

| You want... | Reach for |
|-----------|-----------|
| The full feature set, headless/CI, parallel sessions, every slash command | **CLI** |
| Native diffs, click-to-accept, plan-as-document, selection and diagnostic context | **VS Code extension** (richest IDE surface) |
| The same in a JetBrains IDE | **JetBrains plugin** (CLI-backed, Beta) |
| A CLI-only feature while staying in the IDE | Run `claude` in the **integrated terminal** (`/ide` to link an external one) |

The extension and CLI share conversation history and `~/.claude/settings.json`. They are two front ends on one engine, not separate products. [^7]

---

## 6. Statusline

The statusline is the customizable row at the bottom of Claude Code, and it runs any shell script you point it at. The script gets JSON session data on stdin and prints text to stdout. It runs locally and consumes no API tokens, which is what makes keeping context %, session cost, and rate-limit % on screen all the time essentially free. [^5]

### 6.1 Two ways to set it up

**Let Claude write it.** `/statusline` accepts natural language and generates the script plus the settings entry:

```text
/statusline show model name and context percentage with a progress bar
```

**Configure it by hand** in `settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/statusline.sh",
    "padding": 2,
    "refreshInterval": 5,
    "hideVimModeIndicator": true
  }
}
```

- `padding` -- extra horizontal indent (default `0`).
- `refreshInterval` -- re-run every N seconds (min `1`) on top of event-driven updates; set it for clocks, or for when background subagents change git state while the main session idles.
- `hideVimModeIndicator` -- suppress the built-in `-- INSERT --` if your script renders `vim.mode` itself. [^5]

### 6.2 The data you get on stdin

The JSON payload is generous. These are the fields seniors actually reach for: [^5]

- `model.id`, `model.display_name`
- `workspace.current_dir`, `workspace.project_dir`, `workspace.git_worktree`, `workspace.repo.{host,owner,name}`
- `context_window.used_percentage` / `remaining_percentage` / `context_window_size`, and `exceeds_200k_tokens`
- `cost.total_cost_usd`, `cost.total_duration_ms`, `cost.total_lines_added` / `removed`
- `effort.level` (`low`, `medium`, `high`, `xhigh`, `max`), `thinking.enabled`
- `rate_limits.five_hour.used_percentage` and `.seven_day.used_percentage` (Pro/Max only, after the first API response; handle absence with `// empty`)
- `session_id` (stable per session, so use it, not `$$`, as a cache key), `version`, `output_style.name`, `pr.{number,url,review_state}`

> **Cost field caveat.** `cost.total_cost_usd` is "computed client-side. May differ from your actual bill." Useful as a relative gauge, not an invoice. [^5]

### 6.3 A practical example (model + context bar + cost)

```bash
#!/bin/bash
input=$(cat)
MODEL=$(jq -r '.model.display_name' <<<"$input")
PCT=$(jq -r '.context_window.used_percentage // 0' <<<"$input" | cut -d. -f1)
COST=$(jq -r '.cost.total_cost_usd // 0' <<<"$input")
# color the bar by fill
if   [ "$PCT" -ge 90 ]; then C='\033[31m'   # red
elif [ "$PCT" -ge 70 ]; then C='\033[33m'   # yellow
else                         C='\033[32m';  fi   # green
FILLED=$((PCT/10)); EMPTY=$((10-FILLED))
printf -v FILL "%${FILLED}s"; printf -v PAD "%${EMPTY}s"
BAR="${FILL// /#}${PAD// /.}"
printf "%b[%s]%b %b%s%b %s%% | \$%.2f\n" '\033[36m' "$MODEL" '\033[0m' "$C" "$BAR" '\033[0m' "$PCT" "$COST"
```

`chmod +x ~/.claude/statusline.sh`, then point `statusLine.command` at it. Test before wiring it in:

```bash
echo '{"model":{"display_name":"Opus"},"context_window":{"used_percentage":42},"cost":{"total_cost_usd":1.23},"session_id":"test"}' | ~/.claude/statusline.sh
```

[^5]

### 6.4 Operational notes

- **Updates** fire after each assistant message, after `/compact`, on permission-mode change, and on vim-mode toggle, debounced at 300 ms. If a new update triggers while your script is still running, the in-flight execution is cancelled, so keep it fast (cache `git status` to a temp file keyed by `session_id`). [^5]
- **Terminal width:** `tput cols` will not work because output is captured, not attached to the TTY. Read the `COLUMNS` and `LINES` env vars instead (v2.1.153+). [^5]
- **Workspace trust required.** Because it executes a shell command, the statusline only runs after you accept the trust dialog; if `disableAllHooks: true`, the statusline is disabled too. [^5]
- **`subagentStatusLine`** overrides the per-subagent row in the agent panel; the same trust and `disableAllHooks` gates apply. [^5]
- **Do not reinvent it.** The docs name community projects [`ccstatusline`](https://github.com/sirmalloc/ccstatusline) and [`starship-claude`](https://github.com/martinemde/starship-claude) for prebuilt themed configs. Third-party; vet before installing. [^5]

---

## 7. Output styles

An output style "changes how Claude responds, not what Claude knows. They modify the system prompt to set role, tone, and output format." Reach for one when you keep re-prompting for the same voice every turn, or when Claude is doing something that is not software engineering at all. For the facts and conventions of your project, that job belongs to `CLAUDE.md` (Ch. 03), not an output style. [^8]

### 7.1 Built-in styles

- **Default** -- the standard software-engineering system prompt.
- **Proactive** -- executes immediately, makes reasonable assumptions instead of pausing for routine decisions, and prefers action over planning. Stronger autonomy than auto mode, and it works without changing your permission mode, so you still see permission prompts.
- **Explanatory** -- provides educational "Insights" while completing tasks.
- **Learning** -- collaborative; adds `TODO(human)` markers asking you to write small, strategic pieces of code yourself. [^8]

> **Note:** Explanatory and Learning "produce longer responses than Default by design," which increases output tokens. Worth knowing for cost-sensitive or latency-sensitive work. [^8]

### 7.2 Switching and creating

Switch via `/config` -> Output style (saved to `.claude/settings.local.json`), or set `outputStyle` directly. The standalone `/output-style` command was deprecated in v2.1.73 and removed in v2.1.91; use `/config` or the setting. Output style is part of the system prompt, read once at session start, so changes take effect after `/clear` or a new session. [^8]

A custom style is a Markdown file with frontmatter, saved at user (`~/.claude/output-styles`), project (`.claude/output-styles`), or managed-policy level:

```markdown
---
name: Diagrams first
description: Lead every explanation with a diagram
keep-coding-instructions: true
---
When explaining code, architecture, or data flow, start with a Mermaid diagram, then explain in prose.
```

Set `keep-coding-instructions: true` when you are changing how Claude communicates but still want its built-in software-engineering behavior; leave it out (the default is `false`) when Claude is not coding at all, like a writing assistant or data analyst. [^8]

### 7.3 Output style vs the alternatives

| Feature | How it works | Use when |
|---------|--------------|----------|
| **Output style** | Modifies the system prompt | A different role/tone/format every turn |
| **CLAUDE.md** | Adds a user message after the system prompt | Project conventions Claude should always know (Ch. 03) |
| **`--append-system-prompt`** | Appends to the system prompt, removes nothing | A one-off addition for a single invocation |
| **Subagent** | Runs with its own system prompt, model, and tools | A scoped helper for a focused task (Ch. 08) |
| **Skill** | Loads task-specific instructions when invoked or relevant | A reusable workflow |

[^8]

---

## 8. Diagnostics: `/context`, `/status`, `/doctor`, `--safe-mode`

When config misbehaves, the cause is almost always one of three things: the file did not load, it loaded from somewhere you did not expect, or a higher scope quietly overrode it. The cure is the same in every case. Stop guessing and inspect what actually loaded. [^4]

### 8.1 The inspection toolkit

| Command | Shows |
|---------|-------|
| `/context` | Everything occupying the context window: system prompt, memory, skills, MCP tools, messages. Run it first |
| `/status` | Which settings sources are active, including whether managed settings are in effect |
| `/doctor` | Configuration diagnostics: invalid keys, schema errors, installation health. Press `f` to send the report to Claude for guided fixes |
| `/config` | Interactive config menu: model, output style, auto-update channel, diff tool, theme |
| `/memory` | Which `CLAUDE.md` and rules files loaded, plus auto-memory entries |
| `/permissions` | Resolved allow and deny rules in effect |
| `/hooks` | Active hooks grouped by event |
| `/mcp` | Connected servers, status, and per-project approval |
| `/agents`, `/skills` | Configured subagents and available skills |

[^4]

`claude doctor` from the shell (no session) is the install-health entry point; it also reports the result of the last background update attempt. [^1]

### 8.2 `--debug` and `claude --safe-mode`

- **`/debug [issue]`** (or `claude --debug`) enables debug logging and has Claude diagnose using the log output and settings paths. `claude --debug hooks` watches hook evaluation live (which matchers were checked, exit codes); `claude --debug mcp` surfaces a server's stderr. [^4]
- **`claude --safe-mode`** (v2.1.169+) launches "with all customizations disabled, including `CLAUDE.md`, skills, plugins, hooks, MCP servers, and custom commands and agents. Authentication, model selection, built-in tools, and permissions work normally." If the problem vanishes here, one of those disabled surfaces is the cause; narrow it with the targeted checks above. One caveat: org-deployed managed settings "still partially apply, so policy-configured hooks and status line run even in safe mode." [^4]

### 8.3 The clean-room repro

When even safe mode does not isolate it, bypass everything under `~/.claude` and launch from a directory with no project config. [^4]

```bash
cd /tmp && CLAUDE_CONFIG_DIR=/tmp/claude-clean claude
```

If the problem disappears here, reintroduce your real files one at a time to find the culprit. Managed settings still apply, since they live at a system path outside `~/.claude`. On Linux and Windows you will re-login because credentials live in the config dir; on macOS they are in the Keychain and carry over. [^4]

### 8.4 The config surprises that account for most "it's not working"

[^4]

1. **A `settings.json` value seems ignored** -> the same key is set in `settings.local.json`, which overrides `settings.json`, and both override user settings.
2. **`permissions` / `hooks` / `env` set globally are ignored** -> they were put in `~/.claude.json` (app state) instead of `~/.claude/settings.json` (config). Two different files.
3. **A hook never fires** -> `matcher` is a JSON array instead of a `|`-joined string (an array is a schema error and the entry is dropped), or it is lowercase (`"bash"` not `"Bash"`; matching is case-sensitive), or it lives in a standalone file instead of under the `"hooks"` key.
4. **An MCP server never loads** -> `.mcp.json` is under `.claude/` or at the wrong path instead of the repo root, a relative `command`/`args` path resolved against the launch directory, or a project-scoped server's one-time approval was dismissed (re-approve from `/mcp`).
5. **A skill never invokes** -> the file is `skills/name.md` instead of `skills/name/SKILL.md`, or it carries `disable-model-invocation: true` (the "user-only" badge in `/skills`).

---

## 9. How power users actually configure their environment

Read Anthropic's own engineering guidance and the same conclusion keeps surfacing. Config is code. Check it into git, keep it lean, and put your energy into feedback loops rather than front-loaded prompts. The people getting the most out of the tool are not the ones with the most elaborate setups.

### 9.1 What Anthropic's best-practices guide actually says

The guidance treats `CLAUDE.md` as a short, git-committed file the whole team contributes to, holding "things that apply broadly": Bash commands Claude cannot guess, code-style rules that differ from defaults, testing instructions and preferred runners, repo etiquette, project-specific architectural decisions, and developer-environment quirks like required env vars. Deliberately excluded: anything Claude can figure out by reading code, standard conventions it already knows, and long tutorials. The litmus test for every line is "Would removing this cause Claude to make mistakes?" The warning is blunt: "Bloated CLAUDE.md files cause Claude to ignore your actual instructions!" Full treatment of the memory hierarchy is in Ch. 03. [^9]

On permissions, the doc names three ways to cut the interruption tax without giving up control: auto mode (a classifier reviews commands and blocks only what looks risky), permission allowlists (permit specific safe tools like `npm run lint` or `git commit`), and sandboxing (OS-level filesystem and network isolation). The workflow it recommends around all of this is explore, then plan, then code: use plan mode to separate research from execution so you do not solve the wrong problem fast. [^9]

The single most-emphasized lever is verification. The guide's framing: "Give Claude a check it can run: tests, a build, a screenshot to compare. It's the difference between a session you watch and one you walk away from." A check that returns a pass/fail signal Claude can read closes the loop without you in it. The full treatment of verification, TDD with agents, and adversarial review lives in Ch. 12. [^9]

For scale, the doc points at parallel sessions over isolated git worktrees, non-interactive `claude -p` for CI and fan-out loops, and a Writer/Reviewer pattern where a fresh-context session reviews the work, since a fresh context carries no bias toward code it just produced. Sessions are persistent and reversible, so the discipline is to course-correct early and `/clear` between unrelated tasks rather than let one session accumulate cruft. [^9] The worktree flag itself is `claude --worktree <name>` (short form `-w`), documented on the IDE page: each worktree keeps independent file state while sharing git history, so parallel sessions do not collide. [^7]

> **Vanilla-plus-verification beats elaborate config.** The best-practices guide makes the priority explicit: keep `CLAUDE.md` short and prune it ("Bloated CLAUDE.md files cause Claude to ignore your actual instructions!"), and spend the saved effort on a check Claude can run on its own ("It's the difference between a session you watch and one you walk away from"). The doc also closes by warning that its patterns "aren't set in stone" and that you should develop your own intuition rather than copy a fixed setup. Teach that principle rather than any practitioner's tab count or model pin, both of which drift. [^9]

### 9.2 How Anthropic's internal teams use it

Anthropic's own write-up gives concrete, attributed examples. New data scientists on the Infrastructure team "feed Claude Code their entire codebase to get productive quickly," letting it read the codebase's `CLAUDE.md` files and explain pipeline dependencies. The Security Engineering team has Claude "ingest multiple documentation sources to create markdown runbooks and troubleshooting guides." The Product Design team sets up "autonomous loops where Claude Code writes the code for the new feature, runs tests, and iterates continuously." The throughline matches the best-practices doc: shared memory files, MCP-fed context, and self-verifying loops. [^10]

### 9.3 A recommended baseline for a senior engineer

1. **Native install**, `autoUpdatesChannel: "stable"` if you cannot tolerate a bad release day; pin `minimumVersion` on shared and CI images, `requiredMinimumVersion` if the org must enforce it.
2. **User `settings.json`** with `$schema`, a sane default `model`, a conservative `permissions.deny` for secrets and env files, and a short `permissions.allow` for routine read-only and test commands.
3. **Project `settings.json` committed**, `settings.local.json` gitignored for personal overrides. Keep `CLAUDE.md` lean and in git (Ch. 03).
4. **A custom statusline** showing model, context %, cost, and rate-limit % (plus worktree/branch if you run parallel sessions).
5. **IDE integration** for the diff and selection ergonomics, with `claude` in the integrated terminal for full power.
6. **Know the diagnostic loop cold:** `/context` -> `/status` -> `/doctor` -> `claude --safe-mode` -> clean-room `CLAUDE_CONFIG_DIR`.

---

## Sources

- https://code.claude.com/docs/en/setup
- https://code.claude.com/docs/en/authentication
- https://code.claude.com/docs/en/settings
- https://code.claude.com/docs/en/debug-your-config
- https://code.claude.com/docs/en/statusline
- https://code.claude.com/docs/en/output-styles
- https://code.claude.com/docs/en/vs-code
- https://code.claude.com/docs/en/jetbrains
- https://code.claude.com/docs/en/best-practices
- https://code.claude.com/docs/en/env-vars
- https://code.claude.com/docs/en/changelog
- https://claude.com/blog/how-anthropic-teams-use-claude-code
- https://claude.com/pricing
- https://platform.claude.com/docs/en/about-claude/pricing

[^1]: web | 2026-06-18 | [https://code.claude.com/docs/en/setup](https://code.claude.com/docs/en/setup)
[^2]: web | 2026-06-18 | [https://code.claude.com/docs/en/authentication](https://code.claude.com/docs/en/authentication)
[^3]: web | 2026-06-18 | [https://code.claude.com/docs/en/settings](https://code.claude.com/docs/en/settings)
[^4]: web | 2026-06-18 | [https://code.claude.com/docs/en/debug-your-config](https://code.claude.com/docs/en/debug-your-config)
[^5]: web | 2026-06-18 | [https://code.claude.com/docs/en/statusline](https://code.claude.com/docs/en/statusline)
[^6]: web | 2026-06-18 | [https://code.claude.com/docs/en/jetbrains](https://code.claude.com/docs/en/jetbrains)
[^7]: web | 2026-06-18 | [https://code.claude.com/docs/en/vs-code](https://code.claude.com/docs/en/vs-code)
[^8]: web | 2026-06-18 | [https://code.claude.com/docs/en/output-styles](https://code.claude.com/docs/en/output-styles)
[^9]: web | 2026-06-18 | [https://code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices)
[^10]: web | 2026-06-18 | [https://claude.com/blog/how-anthropic-teams-use-claude-code](https://claude.com/blog/how-anthropic-teams-use-claude-code)
