# Changelog

All notable changes are recorded here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

Consumers pin a commit rather than a tag, so this file is how a reviewer
decides whether a bump is worth taking. The `# vX.Y.Z` comment beside each pin
is what Dependabot rewrites; this is what says whether it matters.

## [Unreleased]

## [1.0.0] — 2026-09-20

First release. Consumed by `sigrix-io/sigrix-mcp`, `sigrix-io/mullion` and
`sigrix-io/postern`.

### Added

- `setup-python-project` — checks out the caller, installs a Python, and
  pip-installs the project. A consuming job is one `uses:` line carrying no
  upstream pin of its own.
  - `checkout: false` for a caller that has already checked out, because it
    needs options this action does not expose or because it is this repository
    testing its own actions.
  - `install: ""` skips the install, for a repository with a Python script and
    no installable package at its root. Found by wiring the third consumer:
    postern's link checker is standard library only, and `pip install` with no
    arguments is an error rather than a no-op.
- `verify-published-package` — installs a release from PyPI by name and checks
  it is the version just published, resolved from site-packages, carrying
  `py.typed`. Its `python` output exists so a caller can assert what its own
  package claims about itself, which is not this action's business.
