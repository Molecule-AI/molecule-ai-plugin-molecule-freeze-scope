# molecule-freeze-scope

A Molecule AI plugin that locks the agent's edit capabilities to one or more path glob patterns. When a user or the agent attempts to edit or write a file outside the allowed glob, the plugin intercepts the operation, logs the attempt, and returns a structured error. The allowed paths are configured in `.claude/freeze` in the project root.

**Hook:** `pre-edit-freeze` (fires before every Edit or Write tool call)
**Runtime:** `claude_code` only
**Tags:** `molecule`, `guardrails`

---

## Purpose

AI coding agents are powerful and can modify many files across a codebase. In sensitive or multi-tenant environments, it is often desirable to limit the agent's edit surface to a specific directory, package, or file set. `molecule-freeze-scope` enforces that constraint by reading a frozen allowlist from `.claude/freeze` and blocking any edit operation whose target path does not fall within the configured glob patterns.

This is useful for:
- Limiting the agent to a specific service or module in a monorepo
- Preventing accidental edits to shared libraries, config files, or infrastructure code
- Enforcing a "read mostly" posture for production-adjacent code paths
- Compliance scenarios where the agent must not modify certain files

---

## The `.claude/freeze` File Format

The `.claude/freeze` file lives in the project root (same directory as `.claude/`). It is a plain text file with one glob pattern per line. Lines starting with `#` are comments and are ignored.

**Example `.claude/freeze`:**

```
# Allowed edit paths for this project
# Only the frontend package can be modified by the agent

src/frontend/**
src/shared/ui/**
```

**Rules:**
- Patterns are evaluated using Node.js `minimatch` (same glob semantics as `.gitignore`).
- Pattern is relative to the project root (the directory containing `.claude/`).
- A trailing `**` matches all contents recursively.
- A pattern without `**` matches files and directories at that exact level only.
- Empty lines are ignored.

### Example Patterns

| Pattern | What it matches |
|---------|-----------------|
| `src/frontend/**` | Anything under `src/frontend/`, recursively |
| `src/frontend/*` | Direct children of `src/frontend/` only (not nested) |
| `*.config.js` | Any `.config.js` file at the project root |
| `src/**/*.test.ts` | Any `.test.ts` file under `src/` |
| `src/utils/*.ts` | Any `.ts` file directly in `src/utils/` |

---

## Hook: `pre-edit-freeze`

### Execution Order

1. The agent prepares to call the `Edit` or `Write` tool.
2. The `pre-edit-freeze` hook fires before the tool call is dispatched.
3. The plugin reads `.claude/freeze` from the project root.
4. If the file does not exist, the plugin emits a warning log and permits all edits (fail-open). See known issue #2.
5. If the file exists, each target path is checked against the allowlist globs.
6. If **all** target paths are within the allowed set, the edit is permitted and the hook passes through.
7. If **any** target path is outside the allowed set, the plugin:
   a. Logs a `scope_violation` event.
   b. Returns a structured error to the agent: `ERR_SCOPE_FROZEN: path 'X' is outside the allowed edit scope defined in .claude/freeze`.
   c. The `Edit` or `Write` tool call is not executed.

### Scope Checking Logic

The path check uses the following algorithm:

```
for each target_path in tool_call.target_paths:
    matched = false
    for each glob in .claude/freeze:
        if minimatch(target_path, glob):
            matched = true
            break
    if not matched:
        BLOCK the operation
        log scope_violation for target_path
```

An edit is permitted only if **all** target paths match at least one glob. A single path outside the freeze scope blocks the entire operation.

### Error Message Format

When an edit is blocked, the agent receives:

```
[freeze-scope] ERR_SCOPE_FROZEN: edit to "src/prod-config/secrets.yaml" is not permitted.
Allowed scope is defined in .claude/freeze. Matching patterns: src/frontend/**, src/shared/ui/**

Requested path: src/prod-config/secrets.yaml
Allowed patterns: src/frontend/**, src/shared/ui/**

Contact your administrator to update .claude/freeze if this path should be editable.
```

---

## Configuration (`plugin.yaml`)

```yaml
settings:
  # Override the freeze file location. Defaults to ".claude/freeze"
  # relative to the project root. Use an absolute path only when necessary.
  freeze_file_path: .claude/freeze

  # Roles that are exempt from the freeze scope check. Users with any of
  # these roles can edit any file regardless of the freeze file.
  # Default: empty array (no overrides).
  override_roles:
    - admin
    - repo-owner

  # Fail mode when the freeze file is missing. "open" permits all edits;
  # "closed" blocks all edits. Default: "open".
  # See known issue #2 for context.
  missing_freeze_mode: open

  # If true, log every permitted edit in addition to blocked ones.
  # Default: false.
  log_permitted_edits: false
```

### Override Roles vs. Bypass Roles

- **override_roles** — Users with these roles bypass the freeze check entirely. No log event is written for their edits. Use for administrators who need full access.
- `bypass_roles` — Not used by this plugin. Bypass is handled via `override_roles`.

---

## Development Setup

### Prerequisites

- Node.js 20+
- npm 10+
- A Molecule AI development environment with `claude_code` runtime
- A test project with a `.claude/` directory

### Clone and Install

```bash
git clone https://github.com/your-org/molecule-ai-plugin-molecule-freeze-scope.git
cd molecule-ai-plugin-molecule-freeze-scope
npm install
```

### Validate `plugin.yaml`

```bash
python3 -c "import yaml; yaml.safe_load(open('plugin.yaml'))"
```

Exit code 0 with no output means the YAML is valid.

### Link for Local Testing

```bash
ln -s /workspace/repos/molecule-ai-plugin-molecule-freeze-scope \
   /path/to/test-project/.claude/plugins/molecule-freeze-scope
```

### Create a Test Freeze File

In the test project root:

```bash
echo "src/frontend/**" > /path/to/test-project/.claude/freeze
echo "src/shared/ui/**" >> /path/to/test-project/.claude/freeze
echo "# Infrastructure code is off-limits to the agent" >> /path/to/test-project/.claude/freeze
```

### Testing

See `runbooks/local-dev-setup.md` for the full test procedure, including verifying that edits inside and outside the freeze scope behave as expected.

---

## Architecture

```
plugin.yaml              # Plugin manifest and configuration
src/
  index.ts               # Plugin entry, registers pre-edit-freeze hook
  freeze-reader.ts       # Reads and parses .claude/freeze
  scope-checker.ts       # Applies minimatch against target paths
  error-builder.ts       # Constructs the ERR_SCOPE_FROZEN error message
  logger.ts              # Writes scope_violation and scope_permitted events
  types.ts               # Shared types: FreezeEntry, ViolationEvent, etc.
tests/
  scope-checker.test.ts   # Unit tests for glob matching
  freeze-reader.test.ts  # Tests for .claude/freeze parsing
  integration.test.ts    # End-to-end test with mock Edit tool calls
```

---

## File Locations

- **Plugin root:** `/workspace/repos/molecule-ai-plugin-molecule-freeze-scope/`
- **Manifest:** `plugin.yaml`
- **Hook rules:** `rules/freeze-scope.md`
- **HITL gates:** `rules/hitl-gates.md`
- **Local dev runbook:** `runbooks/local-dev-setup.md`
- **Known issues:** `known-issues.md`
