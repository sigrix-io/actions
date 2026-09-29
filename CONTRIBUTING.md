# Contributing

These are composite actions the Sigrix repositories share, so a change here
reaches every repository that bumps its pin. The bar is that an action stays a
mechanism every one of them wants.

## What fits

- A fix to how an action checks out, installs a Python, installs a project, or
  verifies a release.
- Something every Sigrix package wants, done once instead of in each caller.
  `verify-published-package` is the worked example: see *What belongs here, and
  what does not* in [README.md](README.md).

## What does not

- A claim one package makes about itself, such as the tools `sigrix-mcp`
  serves or what `mullion` exports. That stays in the caller, which is what
  `verify-published-package`'s `python` output is for.
- A reusable workflow instead of an action. A `workflow_call` job cannot publish
  to PyPI through trusted publishing; README.md says why.

## Working on it

`.github/workflows/ci.yml` runs the actions rather than asserting about them,
against the throwaway project in `tests/fixture/`, on each supported Python, and
against a package already on PyPI. A pull request's CI is the test run, and a
change comes with a step there that fails without it.

## Pull requests

- One change per pull request.
- Every `uses:` inside an action names a 40-character commit, with its release
  in a `# vX.Y.Z` comment beside it.
- A caller's input reaches a shell through `env:`, never written into `run:`.
- `CHANGELOG.md` gets a line under *Unreleased*. Consumers pin a commit, so this
  is how whoever reviews their bump decides whether it matters.
