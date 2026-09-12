# Omen

A build-only agent brick. Drop it next to any codebase, point Claude Code at
it, and you get a builder with fixed rules, hard stop conditions, and a terse
reporting format.

No runtime. No agent loop of its own. Claude Code is the engine; this repo is
the behaviour.

## Layout

```
omen/
├── CLAUDE.md     # the persona, rules, stop conditions, report format
├── skills/       # reusable build patterns
├── tools/        # validators, scaffolding scripts
└── README.md
```

## Use

Standalone, from a target repo:

```bash
cd /path/to/target-repo
claude -p "$(cat /path/to/omen/CLAUDE.md)

TASK: <what to build>

ACCEPTANCE CRITERIA:
1. <checkable statement>
2. <checkable statement>
3. <checkable statement>

SCOPE: <files or dirs in bounds>
OFF-LIMITS: <files or dirs not to touch>"
```

Via Bruce: Bruce generates the acceptance criteria from the operator's request,
invokes Omen in the target repo, checks the report against those criteria, and
reports up.

## Rules that matter

- **Acceptance criteria are required.** Omen asks for them if missing. They are
  the definition of done and the thing Bruce validates against.
- **Works in place.** Omen operates in the repo it's pointed at. It creates a
  new repository only when explicitly told to.
- **Stops early on purpose.** Same failure twice, or five iterations, and it
  stops. Looping is the expensive failure mode.
- **Builds only.** No documentation, no publishing, no pushing.
