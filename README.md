# abcli

> **Status:** Phase 0 (early development) — internal-only.
> **Future home:** standalone repository (see [`../../internal-docs/ITER-CLI-AGENTS-PLAN.md`](../../internal-docs/ITER-CLI-AGENTS-PLAN.md)).

`abcli` is the architectural enforcement and code-generation toolchain
for the ITER SUITE. It validates that committed code conforms to the
project's architectural decisions (ADRs) and — in later phases — generates
new applications, services and tests that already conform out of the box.

> ### [`MANUAL.md`](MANUAL.md) is the PUBLIC manual — editing it publishes
>
> It is the reference external developers read, mirrored to `omnianvil/abcli`
> **with each release**, so it describes the binary a consumer can actually
> install. It lived only on the public mirror once, hand-maintained, and fell
> **four releases behind** — three shipped verbs with no mention at all. It is
> sourced here so a change to a verb and a change to its documentation land in
> the same PR, and `test_manual_documents_the_binary.py` refuses a verb the
> binary exposes and the manual never names.

## Why this directory exists separately

This folder is treated as if it were already a standalone repository. From
day 1, no code outside `tools/abcli/` imports from it, and nothing
inside `tools/abcli/` imports from `apps/`, `backend/`, or `packages/`.

When the spinoff happens, the migration is a single command:

```bash
git filter-repo --subdirectory-filter tools/abcli
```

See **§3 — Target Architecture** of the implementation plan for the full
boundary contract.

## Directory layout

```
tools/abcli/
├── rules/         # YAML rule definitions (one per ADR)
├── fixtures/      # Pass/fail fixtures for each rule (regression suite)
├── runner/
│   ├── bash/      # Phase 0 — Bash + ripgrep + yq implementation
│   └── go/        # Phase 0.5 — single static binary (deferred)
├── tests/         # run-fixtures.sh and friends
├── agents/        # Phase 1+ — Python + LangGraph agentic factory
└── docs/          # CLI's own documentation
```

## Phase status

| Phase | Status | Description |
|---|---|---|
| 0   | Shipped | Bash runner + YAML rules-as-data |
| 0.5 | Deferred | Go runner (drop-in replacement) |
| 1   | Shipped | `/agents/` skeleton + RAG over ADRs |
| 2   | Shipped | App scaffolder agent (`scaffold-app` / `new app` — GOFAI renderer, page templates, gen-api, docs, engine files) |
| 3   | Partial | Full pipeline (Architect → Scaffolder → Dev → QA) — `new entity` + the ADR/federation gates land; the QA leg is open |
| 4   | Not started | Cortex integration |
| 5   | Shipped | Spinoff to standalone repo (`omnianvil/abcli`) |

## Running locally

**Install it. Do not copy it.** (ABCLI-ADR-010 — the wheel is the whole tool.)

```bash
pip install omnianvil-agents        # gives you BOTH `omnianvil-agents` and `omni`

omnianvil-agents check              # the ADR gate, over the whole repo
omnianvil-agents check --staged     # only what git has staged — what the pre-commit hook runs
omnianvil-agents install-hooks      # writes a hook that calls the INSTALLED cli
omni doctor                         # preflight the external-dev environment
```

A copy of a tool is a fork of it: the platform's vendored `tools/abcli/` sat at `VERSION 0.0.0` while abcli
shipped `0.4.0` — still named `iter.sh`, still carrying bugs fixed months earlier. And an external app has no
monorepo and no source tree, so nothing may reach for one.

<details><summary>Legacy: the vendored bash runner (it dies with the last <code>tools/abcli/</code>)</summary>

```bash
# These reach into agents/src/… — they only work where the SOURCE has been copied.
tools/abcli/runner/bash/run.sh --all
tools/abcli/runner/bash/run.sh --staged
tools/abcli/tests/run-fixtures.sh
```
</details>

## License

See [`LICENSE`](LICENSE).
