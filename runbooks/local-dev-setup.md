# Local Dev Setup: molecule-freeze-scope

This runbook covers cloning the plugin, validating the manifest, creating a `.claude/freeze` file, and testing the scope enforcement behavior in a local Claude Code session.

---

## Step 1: Clone the Repository

```bash
git clone https://github.com/your-org/molecule-ai-plugin-molecule-freeze-scope.git
cd molecule-ai-plugin-molecule-freeze-scope
```

If working from a pre-existing local copy:

```bash
git checkout main
git pull
```

---

## Step 2: Validate `plugin.yaml`

Always run this before loading the plugin in a session:

```bash
python3 -c "import yaml; yaml.safe_load(open('plugin.yaml'))"
```

Exit code 0 with no output means the YAML is valid.

Common failure causes:
- Inconsistent indentation (YAML requires consistent whitespace)
- Missing required `settings` fields
- Invalid value for `missing_freeze_mode` (must be `open` or `closed`)

Example failure:
```
Traceback (most recent call last):
  File "<string>", line 1, in <module>
  yaml.scanner.ScannerError: mapping keys are not allowed here
  in "string", line 8, column 1
```

---

## Step 3: Install Dependencies

```bash
npm install
```

---

## Step 4: Link the Plugin to a Test Project

```bash
ln -s /workspace/repos/molecule-ai-plugin-molecule-freeze-scope \
   /path/to/test-project/.claude/plugins/molecule-freeze-scope
```

Verify the symlink:

```bash
ls -la /path/to/test-project/.claude/plugins/
```

You should see `molecule-freeze-scope -> /workspace/repos/molecule-ai-plugin-molecule-freeze-scope`.

---

## Step 5: Create a `.claude/freeze` File in the Test Project

The freeze file must be at `.claude/freeze` in the project root. Create it now:

```bash
mkdir -p /path/to/test-project/.claude
cat > /path/to/test-project/.claude/freeze << 'EOF'
# Freeze scope for test project
# Only edits within src/frontend and src/shared/ui are permitted

src/frontend/**
src/shared/ui/**
EOF
```

Verify the contents:

```bash
cat /path/to/test-project/.claude/freeze
```

Expected output:
```
# Freeze scope for test project
# Only edits within src/frontend and src/shared/ui are permitted

src/frontend/**
src/shared/ui/**
```

### Verify the Freeze File Loads

You can test the freeze file reader directly:

```bash
node -e "
const reader = require('./src/freeze-reader');
const patterns = reader.readFreezeFile('/path/to/test-project/.claude/freeze');
console.log('Loaded patterns:', patterns);
"
```

Expected output (pattern count may vary):
```
Loaded patterns: [ 'src/frontend/**', 'src/shared/ui/**' ]
```

---

## Step 6: Create Test Files Inside and Outside the Freeze Scope

```bash
# Create directory structure
mkdir -p /path/to/test-project/src/frontend/components
mkdir -p /path/to/test-project/src/shared/ui
mkdir -p /path/to/test-project/src/backend
mkdir -p /path/to/test-project/infrastructure

# File inside scope — should be editable
cat > /path/to/test-project/src/frontend/components/Hello.tsx << 'EOF'
export function Hello() {
  return <div>Hello</div>;
}
EOF

# File inside scope — should be editable
cat > /path/to/test-project/src/shared/ui/Button.ts << 'EOF'
export function Button() {}
EOF

# File outside scope — should be blocked
cat > /path/to/test-project/src/backend/index.ts << 'EOF'
// placeholder
EOF

# File outside scope — should be blocked
cat > /path/to/test-project/infrastructure/terraform/main.tf << 'EOF'
# placeholder
EOF
```

---

## Step 7: Start a Claude Code Session

```bash
cd /path/to/test-project
claude_code
```

---

## Step 8: Test Editing Inside the Freeze Scope

Submit the following prompt:

```
Add a prop called 'variant' to the Hello component in src/frontend/components/Hello.tsx
```

### Expected Behavior

The agent is able to edit the file. No error is returned. Check the log output for a `scope_permitted` event (only if `log_permitted_edits: true` in `plugin.yaml`):

```
2026-03-22T10:05:12.001Z INFO [freeze-scope] [scope_permitted]
  user=dev-user
  role=developer
  tool=Edit
  requested_path=src/frontend/components/Hello.tsx
  matched_pattern=src/frontend/**
```

If `log_permitted_edits: false` (default), no event is emitted for permitted edits. You can temporarily set it to `true` to verify the behavior during testing.

---

## Step 9: Test Editing Outside the Freeze Scope

Submit the following prompt:

```
Add a route handler to src/backend/index.ts that responds to GET /health
```

### Expected Behavior

The agent's `Edit` tool call is blocked. The agent should receive the `ERR_SCOPE_FROZEN` error:

```
[freeze-scope] ERR_SCOPE_FROZEN: edit to "src/backend/index.ts" is not permitted.
Allowed scope is defined in .claude/freeze. Matching patterns: src/frontend/**, src/shared/ui/**

Requested path: src/backend/index.ts
Allowed patterns: src/frontend/**, src/shared/ui/**

Contact your administrator to update .claude/freeze if this path should be editable.
```

### Verify the Log Event

Check for a `scope_violation` event in the log output:

```
2026-03-22T10:07:44.003Z ERROR [freeze-scope] [scope_violation]
  user=dev-user
  role=developer
  tool=Edit
  requested_path=src/backend/index.ts
  allowed_patterns=["src/frontend/**","src/shared/ui/**"]
  error_code=ERR_SCOPE_FROZEN
```

---

## Step 10: Test Infrastructure File Outside Scope

Submit:

```
Update the Terraform file at infrastructure/terraform/main.tf to add a comment
```

### Expected Behavior

Blocked. `ERR_SCOPE_FROZEN` returned. `scope_violation` logged.

---

## Step 11: Test Bypass via Override Role (Negative Case)

Add a test role to `override_roles` in `plugin.yaml`:

```yaml
settings:
  override_roles:
    - admin
    - dev-tester    # add this for testing
```

Reload the Claude Code session. Submit the out-of-scope prompt from Step 9 again.

### Expected Behavior

The edit is permitted. If `log_permitted_edits: true`, a `scope_bypassed` event is logged:

```
2026-03-22T10:10:01.001Z INFO [freeze-scope] [scope_bypassed]
  user=dev-user
  role=dev-tester
  tool=Edit
  requested_path=src/backend/index.ts
  bypass_match=dev-tester
```

Remove the temporary `dev-tester` entry before committing.

---

## Step 12: Test Missing Freeze File Behavior

Remove the freeze file temporarily:

```bash
mv /path/to/test-project/.claude/freeze /path/to/test-project/.claude/freeze.bak
```

Reload the session. Submit an in-scope edit prompt.

### With `missing_freeze_mode: open` (default)

The edit is permitted. A `plugin_warning` event is logged:

```
2026-03-22T10:12:00.001Z WARN [freeze-scope] [plugin_warning]
  code=freeze_file_missing
  path=/path/to/test-project/.claude/freeze
  mode=open
```

### With `missing_freeze_mode: closed`

The edit is blocked. A `scope_violation` is logged for every attempt.

Restore the freeze file:

```bash
mv /path/to/test-project/.claude/freeze.bak /path/to/test-project/.claude/freeze
```

---

## Common Issues Table

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| Edit to in-scope file is blocked | Pattern does not cover the file's exact path | Adjust the glob in `.claude/freeze` (e.g., add `**` for recursive matching) |
| Edit to in-scope file is blocked | Freeze file has a malformed glob line | Check `.claude/freeze` for unescaped `[` or `!` characters |
| All edits permitted even with freeze file | `missing_freeze_mode: open` and freeze file missing | Check that `.claude/freeze` exists and is readable |
| All edits blocked even with valid freeze file | `missing_freeze_mode: closed` with file present | Change `missing_freeze_mode` to `open` |
| YAML validation fails | Indentation or missing required field | Check `plugin.yaml` against schema |
| `freeze_path_not_file` error | `.claude/freeze` is a directory | Remove the directory and create a file at that path |
| Symlink traversal succeeds | See known issue #4 | Do not place symlinks inside the freeze scope |
| Agent bypasses freeze by renaming `.claude/freeze` | See known issue #3 | Disable shell permissions or use `chattr +i` on the freeze file |
| Minimatch escape confusion | See known issue #1 | Use [globtester](https://github.com/your-org/globtester) to validate patterns |
| `ERR_SCOPE_FROZEN` not returned | Plugin not loaded | Confirm symlink is present in `.claude/plugins/` |
| Override role bypass not working | Role name mismatch (e.g., `admin` vs `Admin`) | Check exact spelling of role name in identity provider; role matching is case-sensitive |

---

## Cleanup

Remove the symlink from the test project when done:

```bash
rm /path/to/test-project/.claude/plugins/molecule-freeze-scope
```

To remove the test freeze file as well:

```bash
rm /path/to/test-project/.claude/freeze
```

Do not delete the source repository clone.
