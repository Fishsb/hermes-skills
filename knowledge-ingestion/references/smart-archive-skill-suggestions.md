# Smart Archive → Skill Factory Suggestions Contract

Session-derived reference for the smart-archive / skill-optimization / Skill Factory integration. Use this when maintaining or extending the intelligent archive pipeline.

## Goal Standard

Preserve the pre-existing archive outcome while adding Skill Factory suggestions:

```text
scan → user selects number → archive/export → stage JSON + .meta → ingest queue → delete Hermes session → skill-optimization (knowledge extraction + SF inline)
```

The old success criterion remains the baseline: a selected session must either be safely staged+queued and then deleted, or the original Hermes session must remain untouched. Any Skill Factory integration is additive and must not weaken this baseline.

## Deterministic Front Half

`smart-archive.py` is the deterministic front half. It may:

- scan sessions and build `archive-tracker.json`
- export full session JSON
- write `_temp_session/<session>.json`
- write `<session>.json.meta`
- update `ingest-queue.json`
- delete the Hermes session only after JSON + meta + queue update succeed
- show queue/suggestion status

It must not:

- summarize session content
- judge knowledge value
- decide which skill needs optimization
- generate patch suggestions
- patch `SKILL.md`

## Current Safety Fixes to Preserve

### Stage/Queue/Delete Ordering

Deletion is allowed only after:

1. export succeeds
2. staging JSON is written
3. `.meta` is written
4. queue item is written

If staging or queue fails, the Hermes original session remains and any just-created orphan staging JSON/meta is cleaned up.

### Export Validation Boundary

Treat `hermes sessions export` as untrusted until validated. Before staging or deleting, require:

- command exits with code 0
- stdout is non-empty
- stdout parses as a JSON object
- JSON contains an `id` or `messages`-like payload sufficient for later ingestion

If validation fails: do not stage, do not queue, and do not delete the original session.

### Queue Corruption Boundary

If `ingest-queue.json` cannot be parsed, back it up before rebuilding:

```text
ingest-queue.corrupt.YYYYMMDD_HHMMSS.json
```

Never silently overwrite a damaged queue without backup.

### Archive Result States

Archive results should distinguish:

```text
complete                    # staged + queued + Hermes deletion succeeded
partial_delete_failed       # staged + queued, but Hermes deletion failed
False / failed_before_delete # export/stage/meta/queue failed; original session preserved
```

`archive-all` summaries should report these separately.

When deletion fails after staging/queue succeeds, both `.meta` and queue item should record `delete_status=failed`, `status=queued_delete_failed`, and a short `last_error`. `ingest-status` should expose this count so the user can retry or manually resolve.

### Completed Queue Items

When moving an item to completed, preserve the original pending item fields (`title`, `profile`, `staged_file`, `meta_file`, `message_count`, `model`, `optimization`, etc.) and append `status=completed` + `completed_at`. Do not reduce completed records to only `{id, completed_at}`.

### CLI Compatibility Boundary

The archive command-line interface is user-facing state, not an internal detail. Preserve old shortcuts when extending behavior:

- bare `0` archives all pending sessions
- bare `1`, `2`, etc. archive selected session numbers
- bare `1 2 3` archives multiple selected sessions
- `archive 0` and `archive 1 2 3` remain supported
- nonnumeric input such as `archive abc` fails with a friendly message, not a traceback
- empty tracker / no pending sessions returns a clean “nothing to do” result

Do not add subcommands in a way that breaks the old scan → numeric reply flow.

## Suggestion Queue Contract

Skill Factory suggestions are separate artifacts, not Wiki knowledge and not immediate patches.

Recommended layout:

```text
D:/lk/Obsidian/Hermes/wiki/记忆/_temp_session/
  ingest-queue.json
  skill-suggestions.json
  skill-suggestions/
    SF-YYYYMMDD-sessionid-01.json
    SF-YYYYMMDD-sessionid-01.md
```

Canonical suggestion schema:

```json
{
  "suggestion_id": "SF-YYYYMMDD-sessionid-01",
  "fingerprint": "hash(target.skill_name + suggestion_type + suggested_section + normalized proposal.summary)",
  "source": {
    "session_id": "...",
    "session_title": "...",
    "staged_file": "...",
    "evidence": "short quote or tool-observed fact"
  },
  "target": {
    "skill_name": "...",
    "skill_path": "..."
  },
  "suggestion_type": "workflow|pitfall|fallback|prerequisite|trigger|verification",
  "suggested_section": "Overview|When to Use|Workflow|Common Pitfalls|Verification Checklist|References|needs_human_design",
  "proposal": {
    "summary": "...",
    "rationale": "...",
    "suggested_patch_outline": []
  },
  "status": "pending_confirmation|duplicate|blocked_missing_skill|blocked_insufficient_evidence|needs_manual_review",
  "confidence": "high|medium|low"
}
```

Rules:

- MiMo child agents may return suggestions or write per-suggestion sidecars.
- MiMo child agents must not write `skill-suggestions.json` directly.
- MiMo child agents must not patch any `SKILL.md`.
- The main Agent serially unwraps Skill Factory batch output, normalizes simplified suggestions into the canonical queue item schema, dedupes by `fingerprint`, and merges into the master suggestion index.
- If evidence is insufficient, use `blocked_insufficient_evidence` or `needs_manual_review`, not low-confidence patch proposals.
- User approval is required before loading `hermes-agent-skill-authoring` and editing a skill.

## Skill Factory Plugin Boundary

Skill Factory is an archive-time suggestion interface, not a live task observer. Preserve these boundaries:

- no live `tool_call` / `command` session hooks for observing every task
- no current-conversation skill generation
- no direct `SKILL.md` or plugin file writes from the plugin
- plugin commands may propose/list/review/approve/reject/defer queue items only
- `approve` means user approved the suggestion; it does not mean the skill has already been patched
- actual patching belongs to the normal Agent flow after user confirmation and, when applicable, `hermes-agent-skill-authoring`

## Suggested Future Commands

These are design targets, not all necessarily implemented:

```bash
smart-archive.py ingest-claim --limit N
smart-archive.py ingest-complete <session_id> --notes ... --suggestion-file ...
smart-archive.py ingest-fail <session_id> --error "..."
smart-archive.py ingest-retry <session_id>
smart-archive.py ingest-validate
smart-archive.py ingest-repair
smart-archive.py suggestions-status
smart-archive.py suggestions-export
```

## Regression Checklist

Before changing this workflow, verify:

- `archive <N>` still stages JSON + meta + queue item.
- bare numeric replies (`0`, `1`, `1 2`) still work.
- range replies (`N+`, for example `2+`) archive from N through the last tracked session and reject mixed forms like `2+ 4`.
- nonnumeric input reports a friendly error.
- Hermes session deletion occurs only after export validation, staging, meta write, and queue update succeed.
- invalid export output does not stage, queue, or delete.
- queue failure does not delete the Hermes session and does not leave orphan staging files.
- delete failure becomes `partial_delete_failed`, not a full success.
- delete failure is visible in both `.meta` and `ingest-queue.json`.
- corrupt queue files are backed up before rebuilding.
- old list-shaped queues migrate without losing pending items.
- completed queue entries preserve original pending fields.
- unnamed cleanup still deletes only the source session and does not create staging/queue artifacts.
- `ingest-status` works with old and v2 queue shapes.
- `suggestions-status` is read-only and does not generate suggestions.
- Skill Factory plugin has no live observer hooks and no direct skill-writing functions.
- Markdown fences in SKILL.md stay balanced.
