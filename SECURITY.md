# Security

## Reporting

Email security@sigrix.io with what you found and how to reproduce it. Do not
open a public issue for a vulnerability. You will get an acknowledgement
within three working days.

## What these actions do

Both run inside the calling job, with that job's permissions, and neither
takes a secret as an input.

- `setup-python-project` checks the caller out with `actions/checkout` at its
  defaults, which use the job's `GITHUB_TOKEN`, installs a Python, and runs
  `pip install` with the caller's `install` input. That input reaches the
  shell through an environment variable, never pasted into the script, and
  with globbing off.
- `verify-published-package` installs one exact version of one package from
  PyPI into a throwaway environment, imports it from outside the checkout, and
  checks it is that version.

Every upstream action they use is pinned to a commit.

## What is in scope

- A caller's input executed as code rather than passed as arguments.
- Anything that makes an action fetch or run code other than its pinned
  upstream actions and the package the caller named.
- `verify-published-package` passing a release that is not what the index
  serves: satisfied by the checkout, by an older version, or by a package it
  never installed.

## What to report elsewhere

- A vulnerability in an upstream action (`actions/checkout`,
  `actions/setup-python`) belongs to its own repository.
- A workflow of a repository that uses these actions belongs to that
  repository's own `SECURITY.md`.
