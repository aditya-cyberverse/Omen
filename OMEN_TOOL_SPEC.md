# Spec: Omen tool for Bruce

Implement `bruce/tools/omen.py` and register it. One commit. Do not modify
anything else.

**Read first (only these):** `bruce/tools/base.py`, `bruce/tools/registry.py`,
`bruce/models/router.py`. Match their existing conventions for tool definition,
result shape, and error handling. Do not survey the rest of the repo.

---

## What it does

Bruce delegates build tasks to Omen (a Claude Code wrapper). Flow:

```
request → generate acceptance criteria → invoke claude -p in target repo
        → parse report → check criteria → return structured result
```

Bruce never lets Omen push. Bruce pushes separately, on the operator's command.

---

## Tools to expose

### `omen_build`

Arguments:

| arg | type | required | notes |
|---|---|---|---|
| `request` | str | yes | what to build, in the operator's words |
| `repo_path` | str | yes | absolute path to the target repo |
| `criteria` | list[str] | no | if omitted, generate them (see below) |
| `scope` | list[str] | no | files/dirs in bounds |
| `off_limits` | list[str] | no | files/dirs not to touch |

Returns the parsed report plus a per-criterion verdict.

### `omen_status`

Argument: `build_id`. Returns state of a running or finished build.

Builds run in a background thread, like `TaskOrchestrator` does — `omen_build`
returns a `build_id` immediately rather than blocking. Reuse the orchestrator's
threading approach if it fits; do not invent a second concurrency model.

---

## Generating acceptance criteria

When `criteria` is not supplied, generate 3–5 checkable statements from
`request` using the existing `ModelRouter`, classified as a **planning** task so
Stage 3 routing sends it to the hosted model when available.

Constraints on generated criteria:
- Each must be objectively checkable, not a matter of taste. Good: "exposes a
  `run()` entrypoint". Bad: "code is clean".
- Never include lint, formatting, or style criteria.
- If the model returns fewer than 3 or more than 5, retry **once**, then fail
  with a clear error rather than proceeding with bad criteria.

Return the generated criteria in the result so the operator can see what Omen
was actually held to.

---

## Invoking Omen

Build the prompt as:

```
<contents of OMEN_CLAUDE_MD>

TASK: <request>

ACCEPTANCE CRITERIA:
1. <criterion>
...

SCOPE: <scope, or "the current repository">
OFF-LIMITS: <off_limits, or ".env, credentials, .git internals">
```

`OMEN_CLAUDE_MD` is read from a path in config (`omen_claude_md_path`,
defaulting to `~/omen/CLAUDE.md`). If the file is missing, fail immediately with
a clear error — do not fall back to an inline prompt.

Run via `subprocess.run`:

- command: `claude -p <prompt> --output-format json`
- `cwd=repo_path`
- `timeout` from config `omen_timeout_seconds`, default **600**
- capture stdout and stderr
- on `TimeoutExpired`: kill, return `status="timeout"` with whatever was captured

Hard requirements:
- Pass the prompt via stdin or argv, never through a shell string. `shell=False`.
- Do not pass any token, key, or `.env` content into the prompt or environment
  beyond what the parent process already has.
- Never run `git push`, `git push --force`, or any remote-mutating git command
  from this tool.

---

## Parsing the report

Omen ends its output with:

```
STATUS: done | stopped | blocked
CRITERIA:
  - <criterion>: met | not met
CHANGED:
  - <path> (+N/-M)
TESTS: <passed>/<total>
NOTES: <text>
```

Parse into a dict: `status`, `criteria` (list of `{text, met: bool}`),
`changed` (list of `{path, added, removed}`), `tests` (`{passed, total}`),
`notes`.

Parsing must be defensive — Omen is a language model and may deviate:

- Take the **last** occurrence of the block if it appears more than once.
- A missing section becomes `None`, not a crash.
- If `STATUS:` is absent entirely, set `status="unparseable"` and include the
  raw tail (last 2000 chars) in the result. Never raise on malformed output.

---

## Verdict

After parsing, Bruce computes its own verdict — it does not trust Omen's
`STATUS` alone:

- `passed` — status is `done` **and** every criterion is `met` **and** tests
  are not failing (`passed == total`, when tests were reported)
- `failed` — status is `done` but a criterion is not met, or tests fail
- `stopped` — Omen stopped itself (loop guard, iteration cap, scope breach)
- `timeout` / `unparseable` — as above

Return the verdict, the criteria table, and `notes`. Bruce reports this upward;
the operator decides whether to push.

---

## Security classification

Classify `omen_build` as **Tier 2** under the existing policy in
`bruce/security/policy.py` — it executes an external process that writes files.
It must route through the existing approval machinery.

`omen_status` is **Tier 0** (read-only).

Do not create a parallel approval path. Wire into what's already there.

---

## Config additions

Add to `config/config.json` with these defaults:

```json
"omen_claude_md_path": "~/omen/CLAUDE.md",
"omen_timeout_seconds": 600,
"omen_max_concurrent": 1
```

`omen_max_concurrent: 1` — reject a second concurrent build with a clear error.
Parallel Claude Code invocations burn usage fast.

---

## Tests

Add `tests/test_omen_tool.py`. **Mock `subprocess.run` — never invoke `claude`
in tests.**

Cover:
1. Well-formed report parses correctly.
2. Malformed report → `status="unparseable"`, no exception.
3. Report with a criterion `not met` → verdict `failed` even when
   `STATUS: done`.
4. Timeout → verdict `timeout`.
5. Missing `CLAUDE.md` → clear error, no subprocess call.
6. Second concurrent build rejected.
7. `omen_build` is classified Tier 2.

---

## Done when

- `bruce/tools/omen.py` exists and is registered in the tool registry
- `tests/test_omen_tool.py` passes
- The full existing suite still passes
- One commit: `Add Omen build tool`

**Stop conditions:** if two attempts fail the same way, stop and report. Do not
exceed 5 iterations. Do not widen scope beyond the files named here.
