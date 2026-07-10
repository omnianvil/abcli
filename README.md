# abcli

**The developer CLI for teams building on OmniAnvil.** One tool, installed as one package. It runs the
architecture quality-gate on your repo before CI does, tells you *why* a rule exists when it stops you, and
carries a feedback channel back to us — a defect or a capability request, filed from your terminal.

```bash
pip install omnianvil-agents
abcli --version
```

The package is `omnianvil-agents`; the command you type is **`abcli`**.

## What you'll actually run

```bash
abcli check              # the architecture gate — run it before you push; CI runs the same one
abcli explain <rule>     # what a rule wants, why it exists, and how to argue with it
abcli install-hooks      # a pre-commit hook that runs the gate on staged files
abcli feedback new       # report a defect or ask for a capability (see below)
abcli doctor             # preflight your dev environment (docker, toolchain, ports)
abcli --help             # everything else
```

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
