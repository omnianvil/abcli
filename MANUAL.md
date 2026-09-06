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
abcli pin                      # what does THIS repo pin?  (and is it behind?)
abcli pin --latest             # pin the newest release   (writes .abcli.lock)
abcli pin --version 0.5.7      # pin an exact version
abcli pin install              # materialize the pinned binary for this arch
abcli pin lag [--json]         # is this repo behind the latest release?
```

`.abcli.lock` is committed. It carries the version **and the sha256**, so:

- ten app repos move independently — one repo upgrading cannot drag the others;
- **your global install can never silently change what a repo's gate enforces.** Whoever runs `abcli check`
  in this repo — you, the hook, CI, a teammate on a different laptop — runs *the same* gate.

`abcli pin install` fetches the pinned binary into a git-ignored cache and **verifies it against the lock's
sha256**. The lock is the pin; the cache is disposable. CI bootstraps from the lock, so **CI needs no token
and cannot be tricked into running a different gate than yours.**

### A pin that goes stale is a gate that goes blind

The pin's strength is also its trap: **a pinned gate is a frozen gate.** Your repo does not see a new rule
until you bump, and nothing used to tell you the pin had aged — one app shipped for weeks blind to a verb
three releases newer, and only found out by running `abcli pin --latest` on a hunch.

So `abcli pin` (and the `pin install` your CI already runs) compares your lock against the latest release
and says so:

```
⚠️  pin is BEHIND — this repo pins abcli v0.5.13, latest is v0.5.29. The gate enforces the v0.5.13 rules;
    newer verbs/checks are invisible here until you bump.
```

It **never fails** — a stale pin exits 0, always. Reproducibility is the whole point of pinning, and a tool
that broke your build over a rule you never opted into would be a tool you were right to distrust. It only
tells you. And if it cannot reach the release feed it says *that*, rather than reassuring you.

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
not. It also feeds each step the environment the runner would: the workflow-level and job-level `env:` blocks
(a step-level `env:` wins), and the runner state files — `$GITHUB_OUTPUT`, `$GITHUB_ENV`,
`$GITHUB_STEP_SUMMARY`, `$GITHUB_PATH` — so a value a step writes to `$GITHUB_ENV`/`$GITHUB_PATH` reaches the
next step, exactly as on GitHub. A workflow abcli itself renders runs green on the bench.

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
abcli docs scaffold                 # docs/<persona>/<type>/ — tutorials, how-tos, reference, explanation
```

---

## Running it

```bash
abcli dev                           # this app's backend inside the closed devbox image
abcli dev --stack                   # …and its whole depends_on closure, behind the portal
abcli dev --app-only                # …and I know it has dependencies that will not be up
abcli doctor                        # preflight: docker, toolchain, image, ports, app config, and your STACK
```

`abcli dev` runs against the **compiled** platform and the public SDK — you build and run your app; you never
receive our source.

### If your app declares `depends_on`, read this

`abcli dev` boots **one** container. An app that declares

```json
{ "id": "people-hub", "depends_on": ["oracle-hcm-connector"] }
```

is not "the app minus a feature" when its dependency is absent — it is an app whose upstream calls fail
**invisibly**: the directory comes back **empty** and the guarded routes **401**, which reads exactly like
success. That is the worst failure a devbox can have, so `dev` now says it out loud before booting, and
`abcli doctor` prints your whole stack — every app in the `depends_on` closure, in boot order, with the port
and edge route each will answer on, and a ✗ against any dependency no checkout provides.

Dependencies are located by their **declared `id`** (never the folder name) in sibling checkouts — your
multiroot workspace. Add more places to look with `--app-root <dir>` or `$ABCLI_APP_ROOTS`.

Once you have accepted the gap, `abcli dev --app-only` silences the banner. **Treat an empty index in that
mode as unproven, not as data.**

### `abcli dev --stack` — your app AND its dependencies, behind the portal

```bash
abcli dev --stack        # generate the whole closure as a compose overlay + edge routes
```

It writes three files under `.abcli-devstack/`:

| file | what it is |
|---|---|
| `apps.compose.yml` | one unit per app in the closure — a backend (your repo's uvicorn inside the devbox image) and a static MFE host |
| `apps.nginx.conf` | the edge routes: `/api/<id>` and `^~ /apps/<id>/` per app |
| `apps.db-init.sql` | `CREATE DATABASE` per app — each app owns its migrations, none owns its own database |

**The platform base is not generated, and that is deliberate.** The base (postgres, keycloak, the compiled
services, and the portal on `:8000`) is the platform's own `docker-compose.devbox.yml`, and it **travels
inside the published devbox image**. `dev --stack` prints the `docker cp` that extracts it. abcli does not
carry a copy, because a copy is a second source of truth that drifts from the first — silently, which is the
one failure mode this whole toolchain is built to refuse.

The MFE is served at **`/apps/<id>/`** — the app-id canon (#39), derived from your `id`, never read from a
registry. A live registry was once observed serving `/apps/<id>-service/`; that registry was the violation,
and reading the path from it would have baked one service's bug into every app this generator ever writes.

**Your devbox image must be recent.** The stack's edge needs three *global* nginx directives that live in
the base conf shipped inside the image (the fat-JWT header buffer, the exact-match OIDC callback, and the
same-origin flag). They landed on 2026-07-17; an older image does not carry them, and the generated routes
being perfect will not save you. It is the same frontier as the app-id canon — a pre-fix image also carries
the registry that serves `/apps/<id>-service/` — so one pull closes the whole bundle.

Recognise it by the symptom, because none of these say "old image":

| you see | it is |
|---|---|
| the portal loads but lists **0 apps** | the MFE path canon — old registry |
| every `/api` call with a Bearer returns **400** | the header buffer — the persona JWT is ~10KB |
| login redirects to **"Page not found"** | the OIDC callback proxying to Keycloak instead of the SPA |
| the SPA calls `localhost:<port>` your browser cannot reach | the same-origin flag missing |

All four: `docker pull` the devbox image and re-run.

> **Honest status:** the artifacts mirror a stack that was proven by hand on a real docker host — every
> nginx line in them was falsified there, and each one silently breaks the portal when dropped. abcli has
> **not** booted them itself. Treat your first run as the falsification, not as a formality, and file what
> you find.

---

## `abcli publish` — put your app on the shared dev environment

```bash
abcli publish                       # both halves: API + MFE
abcli publish --back                # the API only:  git push --force-with-lease HEAD:<branch>
abcli publish --front               # the MFE only:  build from HEAD, push the packaged dist
abcli publish --dry-run             # resolve + report what would go where; write nothing
abcli publish --status [<sha>]      # the receipt: did it land? (default: your last publish)
```

**Publish publishes your last COMMIT, not your working tree.** Both halves share that semantic: `--back`
pushes HEAD, and `--front` builds inside a clean export of HEAD — so the sha it stamps is provably what
shipped. Uncommitted changes never ship; when you have any, publish says so by name
(`⚠ N uncommitted change(s) did NOT ship (...) — commit to include them`).

One command puts your work on the shared dev environment — **front and back** — behind a single door. The
destination is resolved from your repo's own config; you never name a bucket, a storage credential, or a
git remote on the command line:

| source | value |
|---|---|
| `omni.config.json` | the app **id** → names `/apps/<id>` and `/api/<id>` |
| `DEVGATE_REMOTE` (`.env`) | git remote for the dev-gate refs — required |
| `DEVGATE_BRANCH` (`.env`) | default `dev-gate` |
| `DEVGATE_DIST_REF` (`.env`) | optional; default `<branch>-dist` |

**`--back`** force-pushes (with lease) your HEAD to the dev-gate branch; the host converges it — pull,
`uv sync --frozen`, reload. **`--front`** builds the MFE with your own toolchain (`scripts/vendor-types.sh`
then `pnpm build`) and pushes the packaged `frontend/dist` to a one-commit-deep dist ref; the host unpacks
and serves it. Force-updates are expected on both — these refs are live pointers, not history.

**The only credential is your GitHub identity.** Both halves are a `git push`, so being able to publish
*is* being a member of the org — nothing else to issue, carry, or leak. abcli never touches the storage
behind the environment, and a CI test on our side keeps it that way.

`--env` accepts only `dev-gate` for now — it refuses anything else by name. Higher environments promote
through CI, not through this verb.

### The receipt — `--status`

A publish is **asynchronous**: the push returns immediately, and the environment converges after it. The
publish id printed at the end **is the commit sha** you published — query its receipt any time:

```bash
abcli publish --status              # your last publish (remembered per-repo, in .git/)
abcli publish --status <sha>        # any publish id
```

The environment posts its verdict as **commit statuses** on that sha — also visible on the commit in the
GitHub UI:

| context | green means |
|---|---|
| `devgate/back` | pulled, installed, reloaded — **and the app answers its health probe** |
| `devgate/front` | the packaged dist was unpacked and served |

Exit codes are script-friendly: `0` every posted verdict succeeded · `1` any failure · `2` still pending
(no verdict yet is *pending*, never success). Reading the receipt uses the same GitHub identity that
pushed — no extra token, no extra surface.

#### An EMPTY receipt is the one thing `--status` cannot resolve for you

If the receipt carries **no verdict at all**, two states look byte-identical on the wire:

- **not yet** — the environment converges on a cycle and has not reached your sha;
- **never** — your app is not one of the environment's discovery *candidates*, so no cycle will ever post
  a verdict on it.

`--status` says so plainly, and tells you **how long** it has been empty — the one datum abcli owns. What it
will **not** do is guess: only the host knows its candidate list, and rebuilding that list on the client
would be a *second verdict*, the exact failure mode the conformance plane exists to prevent. If the receipt
stays empty across a few cycles, that is your answer — **ask iter-ops to confirm your app is in the dev-gate
discovery list**, rather than waiting longer.

### Two ways `--front` alone used to hand you a receipt that could never arrive

Both are now refusals, **before** the build, because the alternative was a green followed by a `pending` you
could poll forever. They exist only for `--front` **by itself** — a full `abcli publish` is immune to both,
because its `--back` half does the very thing that was missing.

**1. The app was never in discovery.** The environment finds apps by the **dev-gate branch**; the dist ref
is only its satellite. A first-ever front-only publish creates the satellite and nothing to anchor it, so
there is nothing to converge — ever:

```
publish: --front would push to dev-gate-dist, but 'dev-gate' does not exist on <remote> …
  Run `abcli publish` (both halves) or `abcli publish --back` first to make the app discoverable.
```

**2. The commit never left your machine.** `--back` pushes your HEAD, so the sha lands on the remote as a
side effect and the receipt always resolves. `--front` pushes only the **orphan** dist ref — your sha never
travels. Publishing front-only from an unpushed commit therefore put the dist in the bucket and left the
proof orphaned: `--status` 404s **forever**, until somebody runs `git push`.

```
publish: HEAD (36745cf) is not on <remote> — a front-only publish pushes only the orphan dev-gate-dist,
so this sha would never reach the remote and the receipt could NEVER resolve.
  Fix: `git push` the commit, then re-run `abcli publish --front`.
  Override with --allow-unpushed if you know the sha reaches the remote another way.
```

`--allow-unpushed` exists for the dev whose sha arrives by another route — but it is a claim you make out
loud, not a default. And if abcli cannot ask the remote at all (no network, no `gh`), it does **not** block
you: its own diagnostic's trouble is not your problem.

---

## `abcli px` — the committee radio

```bash
abcli px inbox                     # what is waiting on ME
abcli px send --to vero "…"        # open a thread            (born `over`)
abcli px get <msg-id>              # read ONE message         (marks it read)
abcli px reply <msg-id> "…"        # answer in the thread     (born `over`)
abcli px out <msg-id> "…"          # câmbio, desligo — closes the thread
abcli px listen --last 20          # the open frequency
abcli px who [--expertise rls]     # profiles — who to address
```

A direct channel between first-level agents, who otherwise have none. **`abcli px --help` is the
etiquette** — read it before your first message; it is the protocol, not a preference. In short:

- **Two statuses, and only two.** `over` (*câmbio* — I am waiting for your answer) and `out` (*câmbio,
  desligo* — closed, nobody owes anybody anything). "Over and out" is a contradiction that exists in films.
  A thread is open while its last message is `over`, and that state is derived, never stored.
- **Reading without replying is allowed**, explicitly. If a message needs nothing from you, `px out` it.
  What gets chased is not-reading, and reading-then-shelving.
- **The frequency is open.** `--to` says who a message is *addressed* to, never who may read it. But
  nobody is obliged to listen — listening is always pull.
- **PX or an issue?** If nobody has to answer, it is not a PX. If the outcome must be findable in six
  months by somebody who was not here, PX is not enough — land it in an issue too.

### Two surfaces that reach you: the footer, and the Stop hook

**The footer** — every abcli command ends with a nudge when something is waiting:

```
📬 2 mensagens aguardando — `abcli px inbox`
```

It rides commands you already run — including your pre-commit hook — so the channel needs no daemon to
reach you. Three things it will never do: **change your exit code** (the radio being down leaves your
`abcli check` exactly as it was, and silent), **make you wait** (~1.5s ceiling; a refused connection is
instant), or **appear in machine output** (no TTY, or `--json`/`--quiet`/`--format json`, and it does not
exist). Set `ABCLI_PX_FOOTER=0` for silence.

Configure the **transport** with two environment variables. Your **identity is NOT an env var** — it is
recorded per worktree (see the hook section below), because nexo and vero share one environment and a
process-wide variable cannot tell them apart. abcli **never guesses who you are**: no worktree identity means
silence, because reporting somebody else's inbox is worse than reporting none.

| var | meaning |
|---|---|
| `PX_BASE` | the API root, e.g. `http://localhost:8090/api/px/internal` |
| `PX_KEY` | the service key (or `INTERNAL_SERVICE_KEY`) |

**From outside the box** (an external agent, e.g. behind Cloudflare Access) the radio is reached over the
tunnel, which puts an Access wall in front of the whole domain. Add the service-token pair and abcli sends
it on every call:

| var | meaning |
|---|---|
| `CF_ACCESS_CLIENT_ID` | Cloudflare Access service-token id |
| `CF_ACCESS_CLIENT_SECRET` | Cloudflare Access service-token secret |

Both, or neither — a half-set pair is a confusing 401, not partial auth. On the box they are unset and
nothing changes. If you hit the wall without them, the error names Cloudflare rather than blaming the API.

**The Stop hook** — the footer needs a command to ride; a session with no terminal (an extension session,
no TTY, no tmux) has none. Claude Code hooks reach it. Because ONE agent has MANY live sessions (an
extension AND a tmux), the install wires **two** hooks and the radio arbitrates so a message wakes only ONE:

```bash
abcli px hook install --agent nexo                 # SessionStart→claim + Stop→listen; mode nudge (default)
abcli px hook install --agent nexo --mode listen   # HOLD mode, for a headless/idle fleet agent
abcli px hook uninstall
```

- **SessionStart → `px claim`** — each session, at start, claims the listen (`PUT /agents/<you>/session`).
  Last to claim wins; the others do not die, they go quiet (the `attach -d` model). Same in both modes.
- **Stop → `px listen`** — checks `/wake` and, if mail is waiting for THIS active session, wakes with a
  **bell** (a count and a command, never a body — that text is injected into the woken session, so a body
  would be an injection vector). A superseded session releases at once and stays quiet. Two modes, chosen by
  `install --mode`:
  - **`nudge` (the default)** — check `/wake` ONCE and release. For an **interactive, Principal-facing
    session**: a holding poll would FREEZE it between turns (it looks stuck to the human). Fail-open — an
    empty check, a superseded session, or a down radio all release at once.

    **It is ADAPTIVE**, because "no mail" means two opposite things. *No mail, conversation dead* → release,
    as always. *No mail, but a thread of yours is still awaiting its answer* → **poll instead of releasing**
    (every `--conv-every`, default 60s, for up to `--conv-timeout`, default 15 min). Without this, you answer
    with `over`, the hook checks, the peer replies 40 seconds later — and the reply sleeps until a human says
    *"check your px"*. The channel exists so the human is not the postman; a nudge that cannot wait makes him
    the **alarm clock**, which is worse.

    It disarms on any of four: the reply lands (bell), the peer closes the thread (`out`), another session
    claims the listen, or the ceiling is reached. It is **not** `listen` — it holds only while a real exchange
    is open, and never past the ceiling. `install` also raises the settings `timeout` above that ceiling: a
    hook killed before its loop could deliver would be a mode that looks installed and does nothing.
  - **`listen` (hold)** — HOLD and poll `/wake` until woken. For a **headless/idle fleet agent** that would
    otherwise be dead and unreachable. An empty check does not release — it polls again, bounded by Claude
    Code's `--timeout` on the hook (`--every`/`--timeout` apply to this mode only). `install --mode` switches
    modes idempotently.

    ⚠️ **It covers an idle session, not a stopped one** — once the ceiling expires there is no next `Stop`
    to re-arm it. See *A hold does NOT survive silence* below.

### ⚠️ A hold does NOT survive silence — use `abcli px monitor` for that

Measured on two agents on the same day: one sat stopped for **5h30**, the other for **1h00**, both with
`--mode listen` armed and a working radio, and **neither was woken by the hook**.

The reason is structural, not a bug: a hold only exists *between turns*. When it expires, **no process is
left attached to the session** — so nothing can reach it, and there is no next `Stop` to re-arm it. The hold
is strongest where it is least needed (a session already taking turns) and absent where it matters.

```bash
abcli px monitor          # arm with the harness's Monitor tool, persistent: true
```

`px monitor` polls the radio in a loop and prints one line per event. Armed as a background monitor, it
**leaves a live process tied to the session**, which is what makes the wake possible with no turn to
piggyback on. Both agents above were eventually woken by a background task, not by the hook.

**The filter is emitted by abcli, not written by whoever arms it**, and that is deliberate. Both agents who
designed these rules broke them in their own hand-written monitors within minutes of stating them — one of
them twice in a day, in his live implementation. Three rules, each one asserted by the suite:

1. **Seed without emitting.** The first pass records what is already waiting and says nothing; emitting it
   fires one event per message already in the box, and monitors that emit too much are **stopped
   automatically** — the new failure mode would kill the new watch in its first second.
2. **One line per NEW id**, never per tick.
3. **The radio's own failure is an event**, on the *transition*, both ways. Without it, radio-down and
   inbox-empty are the same silence. On the transition only: a radio down for an hour would otherwise emit
   80 events and disarm the watch that exists to notice it.

Disarm with `TaskStop` on your own monitor — it has an author by construction, and it silences nobody else.
That is the other half of what it fixes: the hook lives in **one user-level file shared by the whole box**,
so any agent switching modes changes everyone, with no record of who. A monitor is per session already.

The agent name is **not** in the commands (nexo and vero share this structure); `install --agent <you>`
records it in **this worktree's** `.claude/px-agent` and the hooks resolve it from there at run time — never
from a shared env var or `~/.claude`, so a name is never one agent's leaked onto the whole box. The file is
**gitignored by the installer** (`.claude/.gitignore` gains `px-agent`/`px-session`), so a committed name can
never resolve on someone else's checkout. No file, no identity → silence (**fail closed**): abcli reports the
wrong agent's inbox to no one. If `install` cannot record the identity it says so **loudly** — it never
claims success over hooks that would be mute. This works because each agent lives in its own worktree; two
different names sharing one worktree is the one case it cannot separate. `install` appends to both hook arrays
and preserves every hook already there (the platform's `claude-persist` among them); idempotent, clean
`uninstall`.

You do not run `abcli px listen` or `abcli px claim` by hand — the hooks run them.

---

## `abcli bundle-check` — the build is green and it ships a `throw`

```bash
abcli bundle-check frontend/dist          # 0 clean · 1 contaminated · 2 could not verify
abcli bundle-check ./unpacked --json      # for CI and the host
```

**A missing federation share does not fail your build. It ships.** If `vite.config.ts` stops spreading the
external shared set, the bundler quietly keeps the local fallback and packages the `@omnianvil` **types-only
stub** — a module whose only statement is a `throw`. The build exits 0, the chunks are emitted, everything
is green, and the app dies on the **first render, in a user's browser**, with nothing in the toolchain
saying a word.

`bundle-check` reads the **emitted bundle** and refuses one carrying that stub. Reading the bundle, and not
`vite.config.ts`, is the whole point: a config can name the right symbol and still produce the wrong output
(imported from a module that no longer exports it, an empty array, a later `mergeConfig`). **The artifact is
the only witness that cannot be talked out of it.**

It needs no repo around it — point it at an unpacked `dist.tar.gz` with no git, no `omni.config.json`, no
`node_modules` — and it **fails closed**: a missing, empty or unreadable dist is `cannot verify` (exit 2),
never a pass. `abcli publish --front` runs the same scan on the dist it just built, so a bundle that would
die on first render never reaches the shared environment.

---

## `abcli plan` — the nine delivery files, owned by the tool

```bash
abcli plan                     # what plan does this repo carry, and is it intact?
abcli plan check               # THE GATE — red on drift  (run it in your CI)
abcli plan upgrade             # adopt the plan this abcli ships (this is also the migration)
```

Nine files take an app from a clone to a running image — `scripts/fetch-sdk.sh`, `scripts/vendor-types.sh`,
both workflows, the `Dockerfile`, `entrypoint.sh`, `.npmrc`, `.dockerignore`, `.gitignore`. Their owner used
to be copy-and-paste, which means **one bug was N bugs and one fix was N fixes** — each found alone, later,
in somebody else's red PR. A repair could not travel.

Now the plan is a versioned artefact: your repo pins one in `.abcli-plan.lock`, a fix lands upstream, and it
reaches every repo that upgrades. **Do not hand-edit a planned file** — `plan check` refuses, on purpose.

The lock cannot be its own witness: re-blessing a hand-edit inside the lock is reported as a **forged** lock,
not as integrity, and a plan version this binary never published is reported as **unverifiable** rather than
intact. A gate with no anchor must refuse, not pass.

---

## `abcli template` — the mechanical half of a shared file, owned by the tool

```bash
abcli template                 # show drift against the gold
abcli template check           # THE GATE — fails closed
abcli template sync            # write the owned parts; never touches yours
```

`plan` owns whole files. This owns what `plan` cannot: files that weld a **mechanical part** (the tool's) to
a **per-app part** (yours) — and it owns them **per key**, never per file.

Today that is `.devcontainer/devcontainer.json` and `AGENTS.md`:

| file | abcli owns | you own (asserted present, never touched) |
|---|---|---|
| `.devcontainer/devcontainer.json` | `image` (the digest), `remoteUser`, `workspaceFolder`, `features`, `mounts` | `name`, `forwardPorts`, `portsAttributes`, `postAttachCommand` |
| `AGENTS.md` | the `<!-- abcli:section:workflow -->` block | every other section, and all your prose |

**Why per key and not per file.** The first attempt at this owned the devcontainer whole, and it would have
**erased every app's ports** — the header is identical across apps, the footer is not. Ownership stops
exactly where your content starts, and a test freezes that boundary as something that cannot happen.

`sync` splices the gold's **raw text** over yours, so every comment in your file — including the ones you
wrote — survives byte for byte. `check` fails **closed**: if it cannot resolve the gold, that is not a pass.

**The escape hatch, and its price.** A repo that must diverge declares it:

```jsonc
// .abcli-template.waivers      (its own file — a sync would erase it, and the pin rewrites .abcli.lock)
{ ".devcontainer/devcontainer.json::image": { "reason": "patched devbox for the on-prem POC", "owner": "vero" } }
```

A waiver **without a reason and an owner is not a waiver** and is ignored — that is the `# noqa` we refuse.
Active waivers print on **every** check run, not behind a flag: a waiver nobody sees is the drift it was
meant to declare.

---

## `abcli agents-sync` — the AGENTS.md workflow section, reconciled and freshness-stamped

```bash
abcli agents-sync              # SYNC the workflow section from the gold + print the freshness stamp
abcli agents-sync --check      # THE GATE — fails on section drift (teeth); the stamp is advisory
```

The onboarding runbook rots one level above the scaffold: `AGENTS.md` carries a **copy** of "how to use
`abcli` in this repo" that never re-syncs. `agents-sync` closes that — it is the AGENTS.md focus of the same
per-section machinery `template` runs, welded to the **freshness plane**:

- **The delta (local, deterministic).** The `<!-- abcli:section:workflow -->` block vs the bundled gold.
  `--check` **fails** on drift — this has teeth, and it works offline, so it is safe as a CI gate. Without
  `--check`, it **syncs** the section from the gold (every other section and all your prose untouched).
- **The stamp (freshness).** abcli pins the contract version its golds were generated from and compares it to
  the platform's published `contracts/VERSION` — the freshness oracle at `GET /api/platform/public/contracts`.
  A `behind` stamp means the golds this abcli emits may be stale: bump abcli. The stamp is **always advisory**
  — it **never** reddens the gate. It **fails open**: a `503` ("contract surface not shipped" — the server
  fails closed, which is "don't know", not "zero drift"), a `404`, or no gateway at all (CI) all degrade to
  "could not tell", never a false "up to date". Point it elsewhere with `--contracts-url` or
  `$ABCLI_CONTRACTS_URL`.

The teeth are on the deterministic local delta; the network-derived freshness is a *cutucão*, never a wall.

---

## `abcli vix` — verify an ADR-113 metadata set, and draw it

The metadata of ADR-113 carries **N graphs of different shapes over one set of nodes** — composition
(a tree, capped at four levels), type inheritance (a tree), field dependency (a DAG), and entity
inheritance (multi-parent). In JSON they do not read. `vix` verifies the set and draws it.

```bash
abcli vix apps/<app>/metadata              # verify the set
abcli vix apps/<app>/metadata --fixtures   # prove it REFUSES (the invalid/ corpus)
abcli vix apps/<app>/metadata --graph      # the graphs, as mermaid
```

**It is a VISUALISER**, and it is heading for an editor. The Principal asked for a way to SEE the
compositions — *"não vou ficar chafurdando um monte de arquivo json pra entender a estrutura"* — and
that is the requirement.

⚠️ I INVERTED THIS ONCE AND IT WAS NOT MINE TO INVERT. I argued that the gate was what made it worth
having and that the drawing was how the gate explained itself; the issue got renamed from *visualizador*
to *verificador* on the strength of it, without asking him. The argument was not wrong on its own terms —
a viewer that only ever draws valid metadata is a shop window — but it answered a question nobody had
asked, and it quietly replaced a requirement. Verification is a FEATURE of the viewer, not the other way
round.

**Three verdicts, never two:** `✗` refused · `?` could not verify · `✓` passed. The third is not
politeness. One fixture asserts a defect the model has no construct to express (`atLeastOneOf` catches
NONE and is blind to BOTH), so approving it would be green over a real defect and refusing it would be
a refusal with no grounds. It comes out `UNRESOLVED`, said out loud, and it moves on its own the day
the model gains the construct.

### The two laws

**A reference that does not resolve is a defect, never "no constraints."** Two of the known defects
fail OPEN: an `extends` that does not resolve, or a type with no `primitive`, constrains nothing — so
every instance test passes. The typo does not produce an error; it produces the *absence of
verification*.

**Not being reached is no excuse not to look.** Measured on the first corpus: 26 of 49 types were
islands, referenced by nothing. A verifier that walks down from the views visits 23 of 49 and calls the
whole thing clean.

### `--fixtures` judges the rule the fixture NAMES

Not the worst verdict anywhere in the set. A fixture asserts ONE defect; if the set happens to be
incomplete elsewhere, that must show up as *collateral*, reported separately — never as the fixture's
verdict. Red for the wrong reason and green for the wrong reason are the same bug.

## Your 1x1 with the principal — and why it is NOT an abcli verb

There is a durable channel of commitments between the principal and **one named agent** — you. PX dies
at `out`, a decision dies at the answer; a commitment lasts.

⚠️ **The verb belongs to the motor, not to this tool**, and that is a decision rather than an omission.
The placement doctrine says *first match wins, top down*: this channel works for any work-type and
carries no domain, so it is layer 1. An `abcli backlog` would be this layer mirroring the other one's
state — and a mirror is one more thing that drifts.

```bash
abconvoy backlog <your-name>                   # what you two have pending, and how long since you met
abconvoy backlogs                              # every pair
abconvoy backlog-add <your-name> --title "…"   # ONLY when he said to put it there
abconvoy backlog-close <id> --status retired --reason "…"
abconvoy backlog <your-name> --review          # stamp it AFTER you have shown him
```

**If you get `coord database not reachable`**, your worktree does not know which coord DB to use. Point
it at the square you serve:

```bash
COORD_DOTENV=/workspaces/platform/.abconvoy.env abconvoy backlogs
```

### The three ways to poison it

**It never interrupts.** No bell, no emission, no nudge, no badge. If you find yourself wanting to page
somebody about an item, that item was a PX message or a decision — not this.

**An item enters only when he says so**, in a live turn. ⚠️ **The engine cannot witness that exchange**
— it records who *claims* to have written the line. Filing what you think you are owed poisons the
channel quietly, and nothing will stop you.

**`--review` means he SAW it, not that you read it.** The stamp is on the **pair**: once set, no other
session of yours brings him the list again today. Stamping after a look you never showed him buys a day
of silence and he never finds out.

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
