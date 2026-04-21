# Known Issues

This document tracks open and known limitations in `molecule-freeze-scope`. Each entry describes the issue, its impact, and any available workarounds.

---

## Issue 1: Glob pattern escaping edge cases

**Severity:** Medium
**Status:** Open
**Discovered:** 2026-01-20

### Description

The plugin uses `minimatch` for glob pattern matching. `minimatch` uses the same escaping rules as `.gitignore`, which differ from shell globbing in subtle ways. Specifically:

- A literal `!` at the start of a pattern segment must be escaped as `\!` to be treated as a character literal rather than a negation operator.
- Square brackets used for character classes must follow minimatch's bracket expression rules. A pattern like `foo[bar]` is interpreted as matching `fooa`, `foob`, or `foor` — not the literal string `foo[bar]`.
- A pattern ending in `\` (backslash) is treated as an escape sequence and may be silently dropped or misinterpreted.

Users who are unfamiliar with glob escaping conventions may write patterns that do not behave as intended, leading to either false permissions (edit allowed when it should be blocked) or false denials (edit blocked when it should be allowed).

### Impact

- The freeze file may not protect the intended paths if a pattern is miswritten.
- Security teams reviewing the freeze file may not realize a pattern is ineffective.
- Debugging a "why was this edit allowed" issue requires familiarity with minimatch internals.

### Workaround

- Use [globtester](https://github.com/your-org/globtester) to validate `.claude/freeze` patterns before committing the file.
- Avoid special characters in file names. If a file contains `[`, `]`, `!`, or `\` in its name, use an exact-path pattern: `path/to/file\[with brackets\].txt` (note the backslash escape).
- Test patterns manually by temporarily adding a file matching the pattern and confirming the behavior matches expectations.
- For character-class literals, wrap the pattern in a directory-level glob that excludes the problematic segment (if feasible).

### Tracking

GitHub issue: [#9 — Document glob escaping requirements and add pattern validator](https://github.com/your-org/molecule-ai-plugin-molecule-freeze-scope/issues/9)

---

## Issue 2: Freeze file not created on first use

**Severity:** Medium
**Status:** Open
**Discovered:** 2026-02-11

### Description

When `molecule-freeze-scope` is activated in a project for the first time, there is no `.claude/freeze` file present. The plugin's `missing_freeze_mode` setting controls the behavior:

- `missing_freeze_mode: open` (default) — All edits are permitted when the freeze file is missing. The plugin logs a `plugin_warning` event.
- `missing_freeze_mode: closed` — All edits are blocked when the freeze file is missing. This prevents unintended edits but also prevents the agent from creating the freeze file itself (which is a catch-22).

The plugin does not automatically create a default `.claude/freeze` file on first use. If the intent is to lock down the project by default, the admin must create the file manually before or immediately after activating the plugin.

### Impact

- A project that has never had a freeze file configured is effectively unprotected.
- Operators may not realize the plugin is in fail-open mode and believe edits are being restricted.
- In `closed` mode, the agent cannot bootstrap the freeze file — a human admin must create it out-of-band.

### Workaround

- Always create a `.claude/freeze` file as part of project onboarding before activating the plugin.
- Set `missing_freeze_mode: closed` in `plugin.yaml` during initial rollout to force an admin to create the freeze file before any edits can happen.
- Add a pre-commit hook to the repository that fails if `.claude/freeze` is missing from a project that has the plugin installed.

### Tracking

GitHub issue: [#14 — Auto-create a default .claude/freeze file when none exists on first activation](https://github.com/your-org/molecule-ai-plugin-molecule-freeze-scope/issues/14)

---

## Issue 3: Agent can rename the freeze file itself

**Severity:** High
**Severity Note:** This is a deliberate design trade-off, not a bug. It is documented here for transparency.

**Status:** Open (by design)
**Discovered:** 2026-02-18

### Description

The `pre-edit-freeze` hook only intercepts `Edit` and `Write` tool calls. File rename operations (using the shell, `git mv`, or the filesystem) are not blocked. A sufficiently capable agent can rename `.claude/freeze` to `.claude/freeze.bak` or move it outside the project directory, effectively removing the freeze constraint without triggering the hook.

For example:

```
> mv .claude/freeze .claude/freeze.bak
```

After this, all subsequent edits are permitted (in `open` mode) or all edits are blocked (in `closed` mode with an empty file).

### Impact

- The freeze scope is not tamper-resistant against an agent that has shell access.
- In high-security environments, this plugin alone is insufficient to enforce a hard boundary.
- An operator with a sophisticated agent may be able to circumvent the guardrail without raising an obvious alert.

### Workaround

- Restrict the agent's shell permissions using `settings.json` permissions configuration (e.g., disable `bash` or `mcp__bash__run`).
- Use a filesystem-level ACL or immutable flag on `.claude/freeze` (e.g., `chattr +i .claude/freeze` on Linux) to prevent renaming even by privileged processes.
- Pair this plugin with the `molecule-prompt-watchdog` plugin to surface a warning when an agent attempts destructive-looking shell operations.
- For environments requiring tamper-resistant enforcement, implement a filesystem watcher that alerts on modifications to `.claude/freeze`.

### Tracking

GitHub issue: [#15 — Evaluate adding pre-rename and pre-delete hooks to close the rename circumvention vector](https://github.com/your-org/molecule-ai-plugin-molecule-freeze-scope/issues/15)

---

## Issue 4: Symlink traversal not blocked

**Severity:** Medium
**Status:** Open
**Discovered:** 2026-03-02

### Description

The plugin resolves target paths using `minimatch` against the literal path string provided by the tool call. It does not resolve symlinks before checking the scope. This means a symlink from inside the allowed scope pointing to outside the allowed scope allows the agent to reach and edit files that should be protected.

Example: `src/frontend/public_links/` is in the allowed scope and contains a symlink:

```
src/frontend/public_links/secret -> /etc/passwd
```

When the agent edits `src/frontend/public_links/secret`, the plugin checks that the path matches `src/frontend/**` (it does) and permits the edit. The actual file modified is `/etc/passwd`, which is outside the freeze scope.

### Impact

- A carefully placed symlink can bypass the freeze scope to edit protected files.
- This is particularly dangerous in shared-namespace environments where multiple services' files are accessible via symlinks.
- The agent may appear to be editing an allowed file while actually modifying a protected one.

### Workaround

- Avoid creating symlinks inside directories covered by the freeze scope in environments where this plugin is used.
- Use `realpath` to resolve symlinks in the scope checker (planned for a future version — see tracking issue).
- For high-security environments, disable symlink creation in the relevant directories using filesystem permissions (`chmod -h` to remove symlink permission or using ACLs).

### Tracking

GitHub issue: [#22 — Add symlink resolution to scope checker to prevent traversal attacks](https://github.com/your-org/molecule-ai-plugin-molecule-freeze-scope/issues/22)
