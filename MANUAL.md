# abcli — manual

The developer CLI for teams building on OmniAnvil. This is the reference: what each verb is for, what it
actually protects, and the traps that cost somebody a day.

**abcli is a tool you run, not a library you vendor.** It ships as a **closed, per-architecture binary** —
there is no wheel, no source tree, and you never need either to use it. A copy of a tool is a fork of it.

---

## Install

Grab the binary for your architecture from [Releases](../../releases/latest), verify it, and put it on your
`PATH`:

```bash
ARCH=$(uname -m | sed 's/aarch64/arm64/;s/x86_64/amd64/')
gh release download --repo omnianvil/abcli --pattern "abcli-linux-$ARCH*" --clobber
sha256sum -c "abcli-linux-$ARCH.sha256"          # never skip this
chmod +x "abcli-linux-$ARCH" && sudo mv "abcli-linux-$ARCH" /usr/local/bin/abcli

abcli --version
```

That is the *global* install — enough to bootstrap. **What actually governs a repo is its pin.**

---

## `abcli pin` — one abcli per repo, not one per developer

```bash
abcli pin                      # what does THIS repo pin?
abcli pin --latest             # pin the newest release   (writes .abcli.lock)
abcli pin --version 0.5.7      # pin an exact version
abcli pin install              # materialize the pinned binary for this arch
```

`.abcli.lock` is committed. It carries the version **and the sha256**, so:

- ten app repos move independently — one repo upgrading cannot drag the others;
- **your global install can never silently change what a repo's gate enforces.** Whoever runs `abcli check`
  in this repo — you, the hook, CI, a teammate on a different laptop — runs *the same* gate.

`abcli pin install` fetches the pinned binary into a git-ignored cache and **verifies it against the lock's
sha256**. The lock is the pin; the cache is disposable. CI bootstraps from the lock, so **CI needs no token
and cannot be tricked into running a different gate than yours.**

---

## The gate

```bash
abcli check                    # the ADR quality-gate on this repo
abcli check --staged           # only what git has staged  (what the hook runs)
abcli check --format json      # for agents and CI
abcli explain adr-031          # what this rule wants, why it exists, how to argue with it
abcli install-hooks            # a pre-commit hook that calls the INSTALLED cli, not a copy of it
```

`abcli check` is the same gate in your editor, your pre-commit hook, and CI — because it *is* the same gate
in all three. It exits non-zero on a violation.

**`abcli explain` prints the rule's INTENT, never its detector.** That is deliberate and it is not
gatekeeping: a `require:` clause is only a *proxy* for the intent, and a developer who can read the proxy can
satisfy the proxy **without** the intent — `adr-031` matches an import line, so publishing the detector makes
a dead import the way past. **You publish the dish, not the recipe.** The waiver comment stays public,
though: a declared exception lands in a diff where a reviewer sees it, and an evasion never does.

> **If a gate misfires on your code, that is not something to work around quietly.** It is a `question`
> issue, and it is the fastest way to get the rule fixed. See [feedback](#the-feedback-channel).

---

## `abcli ci` — run YOUR CI, here, before you push

```bash
abcli ci                            # every job in .github/workflows/ci.yml
abcli ci --job test                 # just one
abcli ci --workflow .github/workflows/other.yml
```

**This is the bench.** It executes each job's `run:` steps from your actual workflow file — not an
approximation of them, not a second implementation that can drift. It skips the `uses:` setup steps (the repo
and the toolchain are already here) and covers the plain `test` job that `check` and `federation-check` do
not.

And it does the one thing that makes a local green *worth something*:

> ### A rich dev box lies.
> Your ambient venv and `$PYTHONPATH` leak onto `sys.path`, so `pytest` passes on your machine and the clean
> runner fails on a dependency it never had. **`abcli ci` runs the steps with that leak STRIPPED.** A green
> here is a green there.

That is the difference between a bench and a wish. Iterate here at three seconds a loop instead of pushing to
find out at six minutes a loop — and never in billed CI minutes.

---

## Wiring a repo to the engine

```bash
abcli engine init                   # plant .npmrc, scripts/fetch-sdk.sh, .github/workflows/ci.yml
abcli engine init --force           # overwrite them (review the diff before you commit)
abcli engine sync                   # refresh the vendored engine wheel in sdk/
abcli federation-check              # is this app ready to federate into the portal AND stand alone?
```

`engine init` is safe to re-run: existing files are **skipped** unless you pass `--force`. With `--force` it
**overwrites** your `ci.yml` and `fetch-sdk.sh` — if your CI is customized, read the diff.

**`federation-check` is strict, and it fails early on purpose.** It exits 1 on the four gaps that only ever
surface later as a blank menu in the portal: an `@omnianvil/*: workspace:*` dependency, a relative `tsconfig`
`extends`, a misplaced `.npmrc`, or an absolute `http://localhost` API baseURL. Wire it into CI beside
`check`.

---

## Scaffolding

```bash
abcli new app                       # a new MFE application from an AppSpec
abcli new entity                    # a DDD N0/N1 entity inside an existing backend
abcli new app-registration          # registration seed files for an app
abcli design-app                    # free text → an app + backend spec
abcli scaffold-app                  # scaffold from an AppSpec JSON
abcli gen-api                       # typed openapi-fetch client wiring for an MFE
```

---

## Running it

```bash
abcli dev                           # this app's backend inside the closed devbox image
abcli doctor                        # preflight: docker, toolchain, image, ports, app config
```

`abcli dev` runs against the **compiled** platform and the public SDK — you build and run your app; you never
receive our source.

---

## The feedback channel

This repository **is** the channel. Two doors, and both run the same completeness check before an issue is
created:

```bash
abcli feedback new --repo omnianvil/platform \
  --kind bug \
  --evidence "the path, the traceback, or the command that failed"

abcli feedback new --repo … --json -        # the AGENT path: a JSON report on stdin. No prompts, ever.
abcli feedback templates                    # generate .github/ISSUE_TEMPLATE (the browser door)
abcli feedback units                        # what this repo is made of — DERIVED from the manifests
```

**You do not classify the report.** You do not need to know which piece of the system it belongs to — paste
the evidence, and the routing is derived from it. `--evidence` is not a formality: it is the routing signal,
and it is what a duplicate search is run against.

Or [open an issue](../../issues/new/choose) and pick a form.

---

## The rest

```bash
abcli docs                          # public app docs (Docs Generation Charter)
abcli seed                          # declarative seed manifests (Seed Delivery Charter)
abcli rag                           # index sources and query them
abcli logo                          # tokenize a flat SVG into the brand logo set
abcli serve                         # the MCP server (stdio) — abcli's verbs, as agent tools
abcli --help                        # everything, always current
```

`abcli --help` is the source of truth. **This manual is written from the binary, not from memory** — if the
two ever disagree, the binary is right and the manual is a bug. [File it.](#the-feedback-channel)

---

## Exit codes

| code | meaning |
|---|---|
| `0` | the gate passed |
| `1` | a violation, or the gate could not run |

Every verb that gates something exits non-zero when it refuses. **A verb that cannot run must never look like
a verb that passed** — if you ever catch one doing that, it is a bug and we want it.
