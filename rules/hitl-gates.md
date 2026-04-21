# HITL Gates

**Status: Not applicable.**

`molecule-freeze-scope` does not implement Human-in-the-Loop (HITL) gates. The plugin operates as a hard enforcement mechanism at the `pre-edit-freeze` hook stage. When an edit falls outside the allowed scope, the operation is blocked unconditionally — there is no opportunity for a human to override or approve it inline.

## Why No HITL Gates Here

HITL gates are appropriate when the plugin needs to pause for human input before allowing an operation to proceed. `molecule-freeze-scope` is designed to enforce a policy boundary rather than seek approval. The distinction:

| Scenario | Mechanism |
|----------|-----------|
| Agent attempts to edit `src/prod-config/secrets.yaml` when not in scope | Hard block, no human approval available |
| Agent attempts a destructive tool call (e.g., shell `rm`) | `molecule-prompt-watchdog` advisory warning; no block |
| Agent attempts to edit a file that is in scope but the operation is risky | Consider a separate HITL plugin at `pre-tool-use` for that specific pattern |

This plugin's philosophy is "scope is defined, scope is enforced." Introducing a HITL approval step at the scope level would undermine the guarantee the plugin provides.

## If You Need an Approval Flow for Scope Violations

If your environment requires that out-of-scope edit attempts trigger a human approval request rather than a hard block, implement a companion plugin that:

1. Monitors for `scope_violation` log events from this plugin.
2. Opens a ticket or sends a notification to a human approver.
3. Temporarily extends `.claude/freeze` upon approval.

This is out of scope for this plugin but is a reasonable extension. See GitHub issue [#31](https://github.com/your-org/molecule-ai-plugin-molecule-freeze-scope/issues/31) for a tracking issue on a possible future `freeze-scope-request` companion plugin.

## Relationship to Other Plugins

The `molecule-prompt-watchdog` plugin is complementary. It injects warnings for destructive keyword patterns at the prompt level. Together, the two plugins provide:
- **Prompt-level guardrail** (`molecule-prompt-watchdog`): Warns before dangerous-looking prompts
- **Edit-level enforcement** (`molecule-freeze-scope`): Blocks edits outside the configured scope

Both plugins are `claude_code`-only and share the same `molecule` and `guardrails` tags.

## Future Considerations

A future version of this plugin may support a `soft_block` mode that:
1. Blocks the immediate edit.
2. Presents the operator with a confirmation prompt.
3. If the operator confirms, allows the edit and logs an `scope_explicitly_approved` event.

Track progress in GitHub issue [#31](https://github.com/your-org/molecule-ai-plugin-molecule-freeze-scope/issues/31).
