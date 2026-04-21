# Rules: freeze-scope

This document specifies the error handling, logging, configuration, and release process for `molecule-freeze-scope`.

---

## Error Handling

### Freeze File Missing

**Condition:** `.claude/freeze` does not exist at the configured `freeze_file_path`.

**Log event:** `plugin_warning`
```
2026-03-22T09:11:05.440Z WARN [freeze-scope] [plugin_warning]
  code=freeze_file_missing
  path=/path/to/project/.claude/freeze
  mode=open
  message="Freeze file not found. All edits are permitted."
```

**Behavior:** Controlled by `missing_freeze_mode` in `plugin.yaml`:

| `missing_freeze_mode` | Behavior |
|----------------------|----------|
| `open` (default) | All edits permitted. A warning is logged. |
| `closed` | All edits blocked. A `scope_violation` event is logged for every edit attempt. |

**Recovery:** Create a `.claude/freeze` file in the project root with the desired glob patterns.

---

### Path Outside Freeze Scope

**Condition:** One or more target paths in an `Edit` or `Write` tool call do not match any glob pattern in `.claude/freeze`.

**Log event:** `scope_violation`
```
2026-03-22T09:12:33.001Z ERROR [freeze-scope] [scope_violation]
  user=bob@example.com
  role=developer
  tool=Edit
  requested_path=src/prod-config/secrets.yaml
  allowed_patterns=["src/frontend/**","src/shared/ui/**"]
  error_code=ERR_SCOPE_FROZEN
```

| Field | Description |
|-------|-------------|
| `user` | User principal identifier |
| `role` | User's role at time of the attempt |
| `tool` | Tool that was being called (`Edit` or `Write`) |
| `requested_path` | The target path that was outside the freeze scope |
| `allowed_patterns` | All glob patterns currently configured in `.claude/freeze` |
| `error_code` | Always `ERR_SCOPE_FROZEN` |

**Behavior:** The `Edit` or `Write` tool call is blocked. The agent receives a structured error response (see below). No file is modified.

**Error response to agent:**
```
[freeze-scope] ERR_SCOPE_FROZEN: edit to "src/prod-config/secrets.yaml" is not permitted.
Allowed scope is defined in .claude/freeze. Matching patterns: src/frontend/**, src/shared/ui/**

Requested path: src/prod-config/secrets.yaml
Allowed patterns: src/frontend/**, src/shared/ui/**

Contact your administrator to update .claude/freeze if this path should be editable.
```

**Recovery:** Contact the project administrator to update `.claude/freeze` if the path should be editable. Do not attempt to bypass the scope check manually.

---

### Malformed Glob Pattern

**Condition:** A line in `.claude/freeze` contains a pattern that `minimatch` cannot parse.

**Log event:** `plugin_error`
```
2026-03-22T09:14:01.002Z ERROR [freeze-scope] [plugin_error]
  code=malformed_glob
  line="src/[bad pattern"
  detail="Invalid glob pattern: brackets not closed in path segment"
```

**Behavior:** The malformed line is skipped. The plugin continues processing remaining patterns. A warning is logged for each skipped line.

**Recovery:** Edit `.claude/freeze` and fix the malformed line. Remember to escape literal brackets as `\[` or `\]` in minimatch patterns.

---

### Override Role Bypass

**Condition:** The user's role is in `override_roles`. The freeze scope check is skipped entirely.

**Log event:** `scope_bypassed` (only if `log_permitted_edits: true`; otherwise this event is not emitted)
```
2026-03-22T09:15:44.003Z INFO [freeze-scope] [scope_bypassed]
  user=admin@example.com
  role=admin
  tool=Write
  requested_path=src/prod-config/settings.yaml
  bypass_match=admin
```

**Behavior:** The `Edit` or `Write` tool call is permitted. No error is returned.

**Recovery:** This is expected behavior for users with override roles. No action needed.

---

### Freeze File Not a File (Unexpected Type)

**Condition:** The path at `freeze_file_path` exists but is not a regular file (e.g., it is a directory or a named pipe).

**Log event:** `plugin_error`
```
2026-03-22T09:16:00.001Z ERROR [freeze-scope] [plugin_error]
  code=freeze_path_not_file
  path=/path/to/project/.claude/freeze
  detail="Freeze path exists but is not a regular file"
```

**Behavior:** The plugin falls back to `missing_freeze_mode` behavior.

**Recovery:** Remove the non-file at the configured path and replace it with a regular `.claude/freeze` file.

---

## Logging

The plugin emits structured log events to stdout:

```
[timestamp] [level] [freeze-scope] [event_type] payload
```

### Event Types

| Event Type | When Emitted | Log Level |
|------------|--------------|-----------|
| `scope_violation` | Edit blocked because path is outside freeze scope | ERROR |
| `scope_permitted` | Edit allowed because all paths are within scope (only if `log_permitted_edits: true`) | INFO |
| `scope_bypassed` | Edit allowed because user has an override role | INFO |
| `plugin_warning` | Freeze file missing or other non-fatal condition | WARN |
| `plugin_error` | Malformed glob, unreadable file, or other fatal condition | ERROR |

---

## Configuration Reference

### `settings.freeze_file_path`

Type: `string`
Default: `.claude/freeze`

Path to the freeze file, relative to the project root (where `.claude/` lives).

### `settings.override_roles`

Type: `Array<string>`
Default: `[]`

Users whose role matches any entry in this list bypass the freeze scope check entirely. No `scope_violation` is logged for these users.

### `settings.missing_freeze_mode`

Type: `string` (`"open"` | `"closed"`)
Default: `"open"`

Behavior when the freeze file is missing:
- `"open"`: permit all edits, log a `plugin_warning`
- `"closed"`: block all edits, log a `scope_violation` for every attempt

### `settings.log_permitted_edits`

Type: `boolean`
Default: `false`

When `true`, emit a `scope_permitted` event for every allowed edit. When `false`, only emit events for blocked or bypassed edits.

---

## Release Process

### Versioning

Follows [Semantic Versioning](https://semver.org/):
- **MAJOR** — Breaking changes to the scope enforcement logic, changes in the `.claude/freeze` file format, or changes to `ERR_SCOPE_FROZEN` error message format
- **MINOR** — New configuration fields, new log event types, or new error codes
- **PATCH** — Bug fixes, documentation updates, test additions

### Pre-Release Checklist

Before publishing a new version:

1. Run YAML validation: `python3 -c "import yaml; yaml.safe_load(open('plugin.yaml'))"`
2. Update `plugin.yaml` `version` field.
3. Run the test suite: `npm test`
4. Run scope-checker unit tests with `npm test -- --testPathPattern=scope-checker`
5. Verify that the glob matching tests cover at least the following cases:
   - File inside recursive glob (`src/frontend/**`)
   - File outside recursive glob
   - Exact file name match (`*.config.js`)
   - Negated or escaped pattern (if escaping is documented)
6. In a clean test project, follow the runbook steps for testing inside/outside freeze scope.
7. Verify `scope_violation` events appear in the log when editing an out-of-scope path.
8. Verify `scope_bypassed` events appear for override role users.
9. Update `CHANGELOG.md`.
10. Tag and push: `git tag -a vX.Y.Z -m "Release vX.Y.Z"` && `git push origin vX.Y.Z`.

### Post-Release

- Notify consumers via release notes.
- For any change to `ERR_SCOPE_FROZEN` message format, update the Error Handling section of this document.
- For any change to log event fields, update the Logging section of this document.
