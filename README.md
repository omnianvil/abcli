# abcli

**The developer CLI for teams building on OmniAnvil.** One tool, installed as one package. It runs the
architecture quality-gate on your repo before CI does, tells you *why* a rule exists when it stops you, and
carries a feedback channel back to us — a defect or a capability request, filed from your terminal.

```bash
ARCH=$(uname -m | sed 's/aarch64/arm64/;s/x86_64/amd64/')
gh release download --repo omnianvil/abcli --pattern "abcli-linux-$ARCH*" --clobber
sha256sum -c "abcli-linux-$ARCH.sha256"
chmod +x "abcli-linux-$ARCH" && sudo mv "abcli-linux-$ARCH" /usr/local/bin/abcli

abcli --version
```

**abcli ships as a closed, per-architecture binary — there is no wheel and no source tree.** A copy of a
tool is a fork of it. Once installed, the thing that actually governs a repo is its **pin** (`abcli pin
--latest`), so your global install can never silently change what a repo's gate enforces.

📖 **[Read the manual](MANUAL.md)** — every verb, what it protects, and the traps.

## What you'll actually run

```bash
abcli pin --latest       # pin THIS repo's abcli (.abcli.lock) — the gate stops depending on your laptop
abcli check              # the architecture gate — run it before you push; CI runs the same one
abcli ci                 # run THIS repo's CI jobs here, with the dev box's leak STRIPPED
abcli explain <rule>     # what a rule wants, why it exists, and how to argue with it
abcli install-hooks      # a pre-commit hook that runs the gate on staged files
abcli feedback new       # report a defect or ask for a capability (see below)
abcli doctor             # preflight your dev environment (docker, toolchain, ports)
abcli --help             # everything else — and see the MANUAL
```

**`abcli ci` is the bench.** A rich dev box lies: your ambient venv and `$PYTHONPATH` leak onto `sys.path`,
so `pytest` passes here and the clean runner fails on a dependency it never had. `abcli ci` runs your real
workflow's steps with that leak stripped — **a green here is a green there.** Iterate at three seconds a
loop instead of six minutes of billed CI.

`abcli check` is the gate. It fails the same way in your editor, your pre-commit hook, and CI — because it is
the same gate in all three. When it stops you and the reason isn't obvious, `abcli explain <rule>` says what
the rule is protecting and how to opt out on purpose.

## Found a bug, or want it to do something it can't?

This repository **is** the channel. There are two doors, and both run the same completeness check before an
issue is ever created:

- **From your terminal** — `abcli feedback new`. It asks for evidence (a path, a traceback, the command that
  failed), searches for duplicates, and files it. Agents use `--json`.
- **From the browser** — [open an issue](../../issues/new/choose) and pick a form.

You don't classify the report — you don't need to know which piece of the system it belongs to. Paste the
evidence; the routing is derived from it.

## What this repo is, and isn't

The tool ships as a **compiled artifact** — you install it, you don't build it, and you never need its source
to use it. Releases and issues live here; that's the whole surface. If a gate misfires on your code, that's
not something to work around quietly — it's a `question` issue, and it's the fastest way to get the rule
fixed.

## License

See [LICENSE](LICENSE).
