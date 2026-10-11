# OpenCode Auto Mode

`opencode-auto-mode` is an automatic permission classifier for OpenCode 2 that brings Claude Code- and Codex-style auto mode to tool execution without requiring the user to approve every permission prompt manually.

For operations that require LLM classification, the plugin is fail-closed. If no reviewer model can be resolved, the reviewer agent is missing, the selected model is unavailable, the review times out, or the model returns an invalid decision, the operation is blocked.

## Features

- Reviews every OpenCode tool invocation before native permission handling begins.
- Replaces native `ask` rules with `allow` on every agent when the plugin loads, while preserving explicit `deny` rules.
- Automatically permits a conservative set of read-only commands.
- Immediately blocks system-destructive commands such as `sudo`, root recursive deletion, disk formatting, and shutdown operations.
- Sends ambiguous and development-related commands to a dedicated, hidden reviewer subagent.
- Detects risky behavior including force-push, branch deletion, worktree removal, hard reset, Git clean, recursive deletion, remote-shell pipelines, and secret-looking environment-variable expansion.
- Includes compact conversation, recent command, Git state, and project-document context in LLM reviews.
- Keeps the reviewer agent tool-free so untrusted review context cannot cause tool execution.
- Supports an explicit reviewer model, configurable timeout, and enable/disable flag.
- Loads the reviewer system prompt from Markdown for easy rule customization.
- Automatically permits a small set of low-risk local tools, including well-formed patches confined to the project, without an LLM request.
- Reviews external filesystem access, source mutations, network operations, and remote-service tools with the configured model.
- Applies task-scoped policy to secrets, data transmission, downloaded code, destructive changes, persistence, network exposure, remote mutations, and subagent delegation.
- Caches decisions for the same tool call.

## Requirements

- OpenCode 2.0.0 or newer.
- At least one reachable model that can answer the reviewer session if commands can reach the LLM-review tier (the `model` option, `OPENCODE_AUTO_REVIEWER_MODEL`, or the model of the session being reviewed).
- Node.js-compatible filesystem APIs. OpenCode normally runs the plugin with Bun.

## How It Works

Every tool invocation goes through three tiers:

| Tier | Result | Examples |
| --- | --- | --- |
| Static allow | Runs immediately | local `apply_patch`, `lsp`, `skill`, `todowrite`, `question`, `ls`, `git status --short`, `gh pr view` metadata reads, `date` |
| Static block | Never executes | `sudo`, `rm -rf /`, `chmod 777`, `mkfs`, `shutdown`, command substitution (`$(...)`) and subshell grouping |
| LLM review | Reviewer returns `ALLOW` or `BLOCK` | external reads, edits, network calls, remote mutations, installs, force-push, recursive deletion |

OpenCode v2 renames several tools at runtime (`bash` → `shell`, `apply_patch` → `patch`, `task` → `subagent`) and uses `path` instead of `filePath` in the `read`, `write`, and `edit` tools. The plugin recognizes both names and both field spellings everywhere (tool hook, path inspection, static rules, and permission actions), so the same review policy applies on v1 and v2.

The plugin registers three extension points in its `setup` function:

1. A `tool` hook on `execute.before` performs the review before any tool executes. If an operation is blocked, the hook throws an error and the tool is never started.
2. A `permission` hook on `evaluate` answers `bash`/`shell` permission requests directly by setting the evaluation effect to `allow` or `deny`, so no native prompt appears.
3. An event subscription on `permission.asked` answers permission requests that still surface as prompts, replying `once` for reviewed-allowed commands and `reject` for everything else.

Only selected low-risk local operations are statically allowed. A well-formed `apply_patch`/`patch` call bypasses model review when all canonical target paths stay inside the project. Other filesystem tools such as `read`, `glob`, and `grep`, external paths, `.env` files, unknown tools, and remote operations are sent to the reviewer with bounded context.

LLM review runs in a short-lived child session using the hidden `auto-reviewer` agent. The reviewer is a tool-free subagent whose every permission is denied, so the untrusted review context cannot cause tool execution. It runs with a small step budget: if the model wastes a turn on a tool call (which is denied), it gets the remaining turns to recover and still answer with the decision line; the response is only parsed once it stabilizes, so intermediate turns are never mistaken for the final decision. The plugin creates the session under the parent conversation, prompts it with the redacted invocation and bounded context, waits for the final assistant message, parses the one-line decision, then interrupts (if the review never completed) and removes the session.

The recommended base configuration keeps a top-level `permissions` ask rule outside the plugin entry:

```json
{
  "permissions": [
    { "action": "*", "resource": "*", "effect": "ask" }
  ]
}
```

When the plugin loads, it rewrites each agent's effective permission rules to `allow` (keeping explicit `deny` rules) and enforces policy in the pre-execution hooks instead. If the plugin is disabled or fails to load, OpenCode retains native permission prompts instead of silently allowing tools.

## Installation

OpenCode loads configuration only at startup. Restart OpenCode after installing the plugin or changing any plugin option, prompt rule, or source file.

### Local Git Checkout

Clone the repository to a stable location:

```bash
git clone https://github.com/nuxlic-sh/opencode-auto-mode.git ~/projects/opencode-auto-mode
```

Register the package in the global OpenCode configuration at `~/.config/opencode/opencode.json`. V2 uses a `plugins` array; local packages use the `{ "package", "options" }` entry form:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugins": [
    {
      "package": "/home/your-user/projects/opencode-auto-mode",
      "options": {
        "enabled": true,
        "model": "provider/model-id",
        "timeoutMs": 60000
      }
    }
  ],
  "permissions": [
    { "action": "*", "resource": "*", "effect": "ask" }
  ],
  "agents": {
    "auto-reviewer": {
      "description": "Hidden tool-free agent that reviews OpenCode tool invocations."
    }
  }
}
```

Use an absolute path. Replace the path and model with values available on your machine.

### Project-Local Installation

V2 discovers local plugins from `.opencode/plugins/` in the project. Copy the plugin into that directory (for example `.opencode/plugins/auto-mode.ts` keeping `auto-reviewer-prompt.md` beside it) and reference it with a relative `package` path:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugins": [
    {
      "package": ".opencode/plugins/auto-mode.ts",
      "options": {
        "enabled": true,
        "model": "provider/model-id"
      }
    }
  ]
}
```

Do not register the same plugin from two locations. Loading both copies would register the hooks twice.

### npm Installation

After the package is published to npm, use its package name as a plain array entry:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugins": [
    {
      "package": "opencode-auto-mode",
      "options": {
        "enabled": true,
        "model": "provider/model-id"
      }
    }
  ]
}
```

### Defining the Reviewer Agent

The plugin fills the reviewer agent's settings (system prompt, denied permissions, hidden subagent mode, single step, and model) through an agent transform when it loads. Some hosts only update agents that already exist, so define a minimal `auto-reviewer` agent in your configuration as shown above. If the agent is absent when a review runs, the LLM-review tier fails closed and the command is blocked.

## Configuration

The plugin accepts these options in the `options` object of its plugin entry:

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `enabled` | boolean | `true` | Enables or disables every plugin hook, transform, and subscription. |
| `model` | string | Context-dependent | Reviewer model in `provider/model` format. |
| `timeoutMs` | number | `60000` | LLM review timeout. Values are clamped between 1,000 and 300,000 milliseconds. |

### Enable or Disable

```json
{
  "enabled": false,
  "model": "provider/model-id"
}
```

When `enabled` is `false`, the plugin registers no hooks, transforms, or subscriptions. The recommended base `permissions` ask rule therefore restores OpenCode's native prompts.

### Select a Reviewer Model

Set `model` to an ID shown by OpenCode:

```bash
opencode models openai
```

Example:

```json
{
  "enabled": true,
  "model": "provider/model-id"
}
```

The value must contain both provider and model IDs. The plugin validates the `provider/model` shape during startup. OpenCode validates actual model availability when the reviewer runs.

If the configured model does not exist or cannot be reached, any command requiring LLM review is blocked. Static read-only commands and static hard blocks do not require the reviewer model.

If `model` is omitted, the plugin tries these sources in order:

1. `OPENCODE_AUTO_REVIEWER_MODEL`.
2. The model used by the session being reviewed.

V2 plugins cannot read OpenCode's global configuration, so there is no fallback to a global default or small model. If none of the sources resolves to a model, LLM review fails closed.

### Configure the Timeout

```json
{
  "enabled": true,
  "model": "provider/model-id",
  "timeoutMs": 90000
}
```

The environment variable `OPENCODE_AUTO_REVIEWER_TIMEOUT_MS` can also set the timeout. A plugin option takes precedence over the environment variable.

## Customize Review Rules

The reviewer system prompt is stored in:

```text
src/auto-reviewer-prompt.md
```

Edit the `## Rules` section to change LLM-review policy. Preserve the required one-line response contract:

```text
ALLOW: <brief reason>
```

or:

```text
BLOCK: <brief reason>
```

The plugin fails to load if the prompt file is missing or empty. Restart OpenCode after editing it.

Static tier rules are implemented in `src/index.ts`:

- `AUTO_PERMITTED` controls commands that bypass the LLM.
- `AUTO_PERMITTED_COMPOUND` controls narrowly defined read-only command chains that bypass the LLM.
- `AUTO_BLOCKED` controls commands that are always rejected.
- `defeatsAutoPermit` prevents shell composition, redirection, substitution, pipes, backgrounding, and secret-looking variables from bypassing LLM review.
- `analyzeCommand` labels risky command behavior for the reviewer prompt.
- `STATICALLY_ALLOWED_TOOLS` identifies local operations that can bypass LLM review, including well-formed `apply_patch` calls confined to the project.
- `inspectToolPaths` canonicalizes nested file targets and shell path operands so symlink escapes cannot receive a static allow or reach review under a misleading lexical path.
- `removeNativePrompts` converts native `ask` rules to plugin-enforced pre-execution review while retaining `deny` rules. It is idempotent so repeated transform replays do not accumulate duplicate rules.

Changes to static rules also require an OpenCode restart.

## Fail-Closed Behavior

The following conditions block tool invocations that need LLM review:

- The `auto-reviewer` agent is not defined.
- Reviewer provider or model does not exist.
- Provider authentication is missing or invalid.
- The provider request fails.
- The reviewer prompt is never delivered to the session.
- Review exceeds `timeoutMs`.
- Reviewer returns anything other than `ALLOW: reason` or `BLOCK: reason`.
- Reviewer session creation fails before a valid decision is produced.
- Tool arguments exceed the serializer's array, object, nesting, or total-size limits.

The plugin logs the failure and throws before tool execution. It never falls back to an unreviewed automatic allow. Incomplete-argument denials name the specific cause in parentheses (for example, `command contains a value that had to be redacted`), so each block is debuggable without instrumentation.

## Context Sent to the Reviewer

The LLM receives bounded context containing:

- A contiguous window of up to nine complete recent user messages within a 4,000-character budget, including the original task while it remains in that window. If the newest message exceeds the budget, no user text is treated as authorization context.
- The two most recent assistant text blocks.
- Command names and derived behavior labels for the five most recent shell commands. Raw historical arguments are omitted.
- Current Git branch and porcelain status.
- The first 50 lines of a regular, non-symlinked `AGENTS.md` or `README.md` whose canonical path remains in the workspace.
- Detected command behaviors, the workspace root, canonical file targets, and the effective shell working directory.

Actual user-role messages are JSON-encoded and labeled as scope-only authorization context. The original task and recent user messages can authorize a named external project or exact path, but cannot alter the reviewer's security rules or response format. Later explicit revocations or restrictions override earlier authorization. Assistant text, prior commands, Git state, and project documentation remain explicitly untrusted.

Tool outputs and reasoning blocks are excluded. The current invocation is included as a bounded, redacted summary. Unknown string fields, content, bodies, comments, patches, replacement strings, credentials, tokens, passwords, URL user information, URL path segments, and URL query names and values are removed; only the query-parameter count remains visible so the reviewer knows data was omitted. The `private` keyword matches identifiers such as `private_key`, not the macOS/BSD `/private/` filesystem prefix, so ordinary temp paths under `/private/var` or `/private/tmp` are not treated as secrets.

File-content tools (`write` and `edit`) carry ordinary file data, so their payloads are projected as a bounded preview of up to 4,000 characters per field with the exact payload size and an explicit omission count; the canonical target paths still gate the decision. Content that matches a secret pattern fails closed, exactly as with patches. Oversized patches (over 8,000 characters), unknown-tool payloads that exceed the serializer's array, object, nesting, or total-size limits, and overlong path or shell summaries block before review rather than silently authorizing a partial projection.

Current shell commands retain their reviewable control structure but pass through deterministic redaction for authorization headers, secret-named assignments and options, private keys, common credential formats, URL paths, and URL query values. External or ambiguous `apply_patch` calls include their patch text so the reviewer can inspect the exact change; detected secrets and oversized patches still fail closed. Raw recent shell commands are never retransmitted. Shell reviews also include canonicalized path operands and the effective working directory. Command substitution (`$(...)`), process substitution, backticks, and subshell grouping block before reviewer submission because they can execute code the reviewer cannot see. Plain `$VAR` references cannot execute code, so they go to the reviewer with an explicit `env-expansion` behavior naming the variables (secret-looking names get a `secret-expansion` warning instead); the variable values are never visible to the reviewer, and the reviewer decides whether the unknown values are acceptable for that command. Ambiguous quoting or path resolution, and other shell forms whose effective targets cannot be represented exactly, also block. Pattern-based redaction cannot recognize every possible opaque credential, so secrets should still be supplied through environment variables or protected files rather than literal command-line arguments. Reviewer context can be sent to a different provider than the parent conversation; choose the reviewer provider according to your data-handling requirements.

## Troubleshooting

### Confirm the Plugin Loaded

Confirm that the plugin id `auto-reviewer` appears in OpenCode's active plugin list or diagnostics, and that its source matches the expected location.

### Inspect the Reviewer Agent

The resolved `auto-reviewer` agent should be hidden, use the configured model, have `steps: 3`, deny every permission, and carry the system prompt from `src/auto-reviewer-prompt.md`.

### Plugin Is Disabled

With `enabled: false`, the agent transform does not run and no hooks are registered. Native permission prompts from the base `permissions` ask rule return.

### Operation Is Blocked as Incomplete Arguments

Denials of the form `Auto-reviewer cannot safely review incomplete <tool> arguments (<reasons>; operation blocked)` name the specific cause. Common causes:

- A command line or file payload that matches a secret pattern (for example, a `KEY=value` assignment or a bearer token on the command line). Move the secret into an environment variable or a protected file.
- A shell command longer than 8,000 characters, such as a large heredoc. Use the `write` tool for file content instead of shell redirection.
- Shell syntax whose path targets cannot be verified, such as unbalanced quoting.
- Unknown-tool payloads that exceed the serializer's size, nesting, or item-count limits.

If a legitimate workflow is repeatedly blocked, adjust the corresponding limit or pattern in `src/index.ts` and rerun the test suite.

### Commands Are Blocked Because Review Fails

Check that the model exists and provider authentication works:

```bash
opencode models <provider>
```

Run OpenCode with logs enabled for more detail:

```bash
opencode --print-logs --log-level DEBUG
```

### Reviewer Response Is Unclear

A review can fail with `Reviewer response was unclear` when the reviewer model does not answer with the required single line (`ALLOW: <reason>` or `BLOCK: <reason>`) — for example when it narrates an analysis or calls a tool, which is denied. The operation fails closed. The reviewer runs with a three-step budget so it can recover from a wasted turn, and it is instructed not to use tools; if failures persist, point the `model` option at a model that follows output-format instructions reliably.

### Recover From an Invalid Reviewer Model

Fail-closed behavior can also block edits to OpenCode's own configuration when the configured reviewer model is invalid. Start one recovery session without external plugins:

```bash
OPENCODE_PURE=1 opencode
```

Correct the `model` option in `opencode.json`, quit that recovery session, and restart OpenCode normally. The base `permissions` ask rule remains active in pure mode, so OpenCode asks for native approval instead of silently allowing tools.

Do not use `OPENCODE_PURE=1` for normal work. It intentionally disables all external plugins for the recovery process.

### A Native Permission Prompt Still Appears

Restart OpenCode first. The plugin converts effective `ask` rules to `allow` and performs enforcement in the pre-execution hooks. Explicit `deny` rules remain denied. If a prompt persists, confirm the plugin loaded from the expected location and that the agent transform ran for the session's agent.

## Development

Install development dependencies, run the tests, and run the type checker:

```bash
npm install
npm test
npm run typecheck
```

Inspect the package contents before publishing:

```bash
npm pack --dry-run
```

The package default-exports a V2 plugin definition (`Plugin.define` with id `auto-reviewer`); it is the entrypoint used by OpenCode 2 plugin loading.

## Security Notes

- This plugin reduces permission fatigue; it is not an operating-system sandbox.
- Every tool call is intercepted, but only conservative local commands and low-risk coordination or introspection tools bypass LLM review.
- Static rules are intentionally conservative, but shell syntax is complex and platform-specific.
- LLM decisions are probabilistic. Keep hard security invariants in static block rules.
- Do not use OpenCode's `--auto` flag as a replacement for this plugin's review policy.
- Keep OpenCode and the plugin updated, and review local rule changes before use.

## License

MIT
