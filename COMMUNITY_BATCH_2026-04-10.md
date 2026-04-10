# Community Batch — 2026-04-10

Upstream state at time of application: `64e8dee` (SDK v0.1.58, CLI 2.1.100).
Branch: `community-fixes` (reset to upstream/main, then patched).

This document describes every change cherry-picked from open community PRs
on `anthropics/claude-agent-sdk-python` and applied as a single batch.

---

## 1. PR #806 — `setting_sources=[]` truthiness fix

**Issue:** #794
**File:** `src/claude_agent_sdk/_internal/transport/subprocess_cli.py`

### Problem

`_build_command()` used `if self._options.setting_sources:` — a truthiness
check. In Python, `[]` is falsy, so passing `setting_sources=[]` (meaning
"load no setting sources") silently did nothing. The CLI fell back to loading
all sources (user, project, local).

### Fix

Changed to `if self._options.setting_sources is not None:`. When the list is
empty, the SDK now sends `--setting-sources ""` to the CLI, which correctly
disables all setting sources.

### Test changes

- `test_build_command_setting_sources_omitted_when_empty` renamed to
  `test_build_command_setting_sources_empty_list_passes_empty_string` and
  updated to assert that `--setting-sources` IS present with value `""`.

---

## 2. PR #803 — `betas=[]` and `plugins=[]` truthiness fix

**File:** `src/claude_agent_sdk/_internal/transport/subprocess_cli.py`

### Problem

Same class of bug as #806. `betas` and `plugins` used truthiness checks,
making empty lists indistinguishable from `None` (unset).

### Fix

- `betas`: changed to `if self._options.betas is not None:`. Empty list sends
  `--betas ""`.
- `plugins`: changed to `if self._options.plugins is not None:`. Empty list
  simply iterates zero times (no behavioral change, but semantically correct).

---

## 3. PR #786 — ThinkingBlock missing `signature` field

**File:** `src/claude_agent_sdk/_internal/message_parser.py`

### Problem

`message_parser.py:110` used `block["signature"]` — a direct dict lookup.
The Anthropic API can return thinking blocks without a `signature` in certain
conditions (redacted thinking blocks, streaming edge cases, older cached
responses), causing a `KeyError` crash.

### Fix

One-line change: `block["signature"]` -> `block.get("signature", "")`.

### Test added

- `test_parse_assistant_message_with_thinking_missing_signature` — verifies
  parsing succeeds with an empty string default for the signature field.

---

## 4. PR #790 — Suppress `ProcessError` when result already received

**File:** `src/claude_agent_sdk/_internal/query.py`

### Problem

When using StructuredOutput (or any tool_use stop reason), the CLI exits with
code 1 because it called a tool but received no tool_result response. The
transport raises `ProcessError` after the process exits, even though a valid
`ResultMessage` was already streamed to the consumer.

### Fix

Added a `except ProcessError` handler in `_read_messages()` that checks
`_first_result_event.is_set()`. If a result was already received, the error
is logged as a warning and suppressed. If no result was received, the error
propagates normally.

### Import added

`from .._errors import ProcessError` — needed in `query.py` to catch the
specific exception type.

---

## 5. PR #658 — Capture real stderr in `ProcessError`

**File:** `src/claude_agent_sdk/_internal/transport/subprocess_cli.py`

### Problem

When the CLI subprocess exits non-zero, `ProcessError` was raised with
`stderr="Check stderr output for details"` — a hardcoded string. The actual
stderr was only piped when the user explicitly provided a `stderr` callback
or enabled debug mode. This made debugging `Command failed with exit code 1`
failures impossible without forking the SDK.

### Fix

Three changes:

1. **Always pipe stderr** — `stderr_dest` is now unconditionally `PIPE`,
   removing the conditional `should_pipe_stderr` check. The stderr stream
   setup is also unconditional.

2. **Buffer stderr lines** — Added `self._stderr_buffer: list[str] = []`.
   Every line read from stderr is appended to this buffer (in addition to
   being forwarded to the user callback if one exists).

3. **Use buffer in ProcessError** — When creating `ProcessError` on non-zero
   exit, `stderr` is set to `"\n".join(self._stderr_buffer)` instead of the
   hardcoded placeholder.

The buffer is cleared on `close()` to prevent memory leaks across reuse.

---

## 6. PR #791 — Suppress stale task notifications between turns

**File:** `src/claude_agent_sdk/client.py`

### Problem (Issue #788)

When a background task (spawned via `run_in_background=True`) completed
between turns, its `TaskNotificationMessage` sat in the shared message
buffer. The next `receive_response()` call would yield it as the first
message — before any response to the new query, causing the model to respond
to stale task context.

### Fix

Added turn-tracking state to `ClaudeSDKClient`:
- `_current_turn: int` — incremented each time a `ResultMessage` is yielded.
- `_task_turn_map: dict[str, int]` — maps task IDs to the turn they started.

`receive_response()` now defers task-lifecycle events that arrive before the
first non-task message of the current turn. Once a non-task message arrives
(proving the CLI is processing the latest query), deferred events are
re-evaluated:
- Notifications for tasks started in a **previous turn** are **dropped**.
- Notifications for tasks started in the **current turn** (or unknown IDs)
  are **yielded**.

### Tests added (4)

- `test_stale_notification_before_turn2_is_suppressed`
- `test_notification_arriving_mid_turn_is_yielded`
- `test_turn_counter_increments_and_cleans_map`
- `test_unknown_task_id_notification_is_yielded`

---

## 7. PR #763 — Guard against malformed `CLAUDE_CODE_STREAM_CLOSE_TIMEOUT`

**File:** `src/claude_agent_sdk/client.py`

### Problem

`ClaudeSDKClient.connect()` used `int(os.environ[...])` to parse the timeout
env var. If the value was malformed (e.g., `"60s"`), connection setup raised
`ValueError` before the SDK could initialize.

### Fix

Added `_parse_timeout_ms_from_env()` helper that:
- Returns the default (60000ms) when the env var is unset.
- Uses `float()` parsing with fallback on `TypeError`/`ValueError`.
- Rejects non-finite values (`inf`, `nan`).

### Test added

- `test_connect_with_invalid_timeout_env_falls_back_to_default`

---

## 8. PR #805 — `delete_session()` cascades subagent transcript directory

**File:** `src/claude_agent_sdk/_internal/session_mutations.py`

### Problem

`delete_session()` only removed the `{session_id}.jsonl` file. The sibling
`{session_id}/` subdirectory (which holds subagent transcripts) was left
behind, leaking disk space.

### Fix

After removing the `.jsonl` file, `shutil.rmtree(path.parent / session_id,
ignore_errors=True)` cleans up the subagent directory. `ignore_errors=True`
because most sessions never spawn subagents, so the directory usually doesn't
exist.

### Import added

`import shutil`

### Test added

- `test_removes_subagent_transcript_dir` — creates a session with a sibling
  subagent dir, deletes the session, asserts both are gone.

---

## 9. PR #804 — Top-level `skills` option on `ClaudeAgentOptions`

**Files:**
- `src/claude_agent_sdk/types.py`
- `src/claude_agent_sdk/_internal/transport/subprocess_cli.py`
- `src/claude_agent_sdk/_internal/query.py`
- `src/claude_agent_sdk/_internal/client.py`
- `src/claude_agent_sdk/client.py`

### Problem

Enabling Skills required two non-obvious steps in unrelated fields:

```python
options = ClaudeAgentOptions(
    allowed_tools=["Skill"],
    setting_sources=["user", "project"],
)
```

### Fix

Added `skills: list[str] | Literal["all"] | None = None` to
`ClaudeAgentOptions`. The SDK now handles the wiring automatically:

- `skills="all"` -> injects `"Skill"` into `allowed_tools`, defaults
  `setting_sources` to `["user", "project"]`.
- `skills=["pdf", "docx"]` -> injects `"Skill(pdf)"`, `"Skill(docx)"` into
  `allowed_tools`, defaults `setting_sources`.
- `skills=None` -> no-op (default).

The `_apply_skills_defaults()` method in `SubprocessCLITransport` computes
effective values without mutating the original options object.

The skills list is also sent on the `initialize` control request so
supporting CLIs can filter which skills appear in the system prompt.

### Tests added (10)

- Transport tests for all skills scenarios (none, all, empty, named, merge,
  preserve user setting_sources, no mutation, no duplicates).
- Query initialize tests for skills list vs. all/None serialization.

---

## 10. PR #691 — `PostCompact` hook event support

**Files:**
- `src/claude_agent_sdk/types.py`
- `src/claude_agent_sdk/__init__.py`

### Problem

The CLI (v2.1.76+) fires a `PostCompact` event after context compaction
completes, but the Python SDK had no type for it.

### Fix

Added to `types.py`:
- `PostCompactHookInput` — with `trigger` (`"manual"` | `"auto"`) and
  `compact_summary` fields.
- `PostCompactHookSpecificOutput` — with `additionalContext` field.
- Added `"PostCompact"` to the `HookEvent` union.
- Added both types to the `HookInput` and `HookSpecificOutput` unions.

Exported both new types from `__init__.py` and added them to `__all__`.

### Tests added (3)

- `test_post_compact_hook_input` (auto trigger)
- `test_post_compact_hook_input_manual_trigger`
- `test_post_compact_hook_specific_output`

---

## Verification

After applying all changes:

| Check | Result |
|-------|--------|
| `ruff check` | Clean |
| `ruff format` | Clean |
| `mypy src/` | Success: no issues found in 15 source files |
| `pytest tests/` | **479 passed** (20 new tests) |
