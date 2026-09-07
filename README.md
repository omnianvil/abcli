# abcli

> **Status:** shipped, standalone, closed. Consumers run the **compiled binary** from
> [`omnianvil/abcli`](https://github.com/omnianvil/abcli) (public releases, no wheels, no source).

`abcli` is the architectural enforcement and code-generation toolchain. It validates that committed
code conforms to a repository's architectural decisions (ADRs), and generates applications, services
and tests that already conform out of the box.

> ### [`MANUAL.md`](MANUAL.md) is the PUBLIC manual — editing it publishes
>
> It is the reference external developers read, mirrored to `omnianvil/abcli`
> **with each release**, so it describes the binary a consumer can actually
> install. It lived only on the public mirror once, hand-maintained, and fell
> **four releases behind** — three shipped verbs with no mention at all. It is
> sourced here so a change to a verb and a change to its documentation land in
> the same PR, and `test_manual_documents_the_binary.py` refuses a verb the
> binary exposes and the manual never names.

## The boundary — and the thing that verifies it

`abcli` is standalone. One verb, and one only, depends on a consumer's code:

| module | why |
|---|---|
| `seed_cmd.py` | `seed` operates on the platform's own manifests (`omnianvil.core.seed`) |

The rules that hold it there, each with a test in
[`agents/tests/test_boundary.py`](agents/tests/test_boundary.py):

1. **nothing imports a consumer's package at module level** — `abcli` must start on a box where that
   package does not exist, which is every box a consumer runs it on;
2. **the coupled modules are a declared list** — a new one fails the build until someone writes down
   the reason. Without this the first rule is satisfied by any lazy import, and the coupling grows one
   module at a time, each step defensible, with no place anyone has to say it out loud;
3. **the coupled verb refuses with a sentence**, not a `ModuleNotFoundError`. A closed binary has no
   source beside it to undo the misreading, and *"abcli is broken"* is what a traceback says when the
   truth is *"this verb is not for here"*.

> This section replaces one that promised a boundary and had **no verifier**. It claimed nothing
> imported from `apps/`, `backend/` or `packages/` while `seed_cmd.py` imported the platform, and it
> named a `backend/` folder that had been deleted. **A promise without a test is a declaration, and
> every new import makes it dearer in silence.** (Named by @nexo, who measured it.)

## Directory layout

```
rules/         # YAML rule definitions (one per ADR)
fixtures/      # Pass/fail fixtures for each rule (regression suite)
runner/bash/   # the Bash runner
tests/         # run-fixtures.sh and friends
agents/        # the Python CLI (click) — check, vix, px, pin, hooks, scaffolding
clients/       # the VS Code extensions (Vix)
docs/          # the CLI's own documentation
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
