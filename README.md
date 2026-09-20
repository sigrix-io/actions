# sigrix-io/actions

Composite actions shared by the Sigrix repositories. One place for the upstream
action pins.

## Why this exists

`sigrix-io/postern`, `sigrix-io/mullion` and `sigrix-io/sigrix-mcp` ran the
same two `uses:` lines in eleven places between them. Every one named a commit,
which is right — a tag is moved by whoever owns the action, and nothing in a
consuming repository would record that it had.

But a pin only stays correct if something bumps it, and eleven pins meant three
Dependabot queues editing the same two actions. That is how mullion came to run
floating tags while postern was pinned: nothing failed, the divergence simply
accumulated between review cycles. One compromised action was three pull
requests under time pressure.

## Why composite actions and not reusable workflows

**A reusable workflow cannot publish to PyPI.** Trusted publishing matches the
OIDC claim against the workflow that ran, and a `workflow_call` job's token
names the *called* workflow rather than the caller, so PyPI answers
`invalid-publisher` — see [pypa/gh-action-pypi-publish#166][1].

A composite action runs **inside the caller's job**, so the caller's identity
is unchanged and the publish step stays where it is.

It also fits postern, which shares no job graph with the other two and would
never call a shared CI workflow — but does run the same upstream actions.

[1]: https://github.com/pypa/gh-action-pypi-publish/issues/166

## Using them

Pin by commit with the version in a comment, the same way every other `uses:`
in these repositories is written. `sigrix-io/mullion`'s
`tests/test_workflow_pins.py` enforces exactly that shape.

### `setup-python-project`

Checks out the caller, installs a Python, and pip-installs the project. The
default case is one line carrying no upstream pin at all:

```yaml
- uses: sigrix-io/actions/setup-python-project@<sha>  # v1
```

```yaml
- uses: sigrix-io/actions/setup-python-project@<sha>  # v1
  with:
    python-version: ${{ matrix.python-version }}
    install: build twine      # default: -e .[dev]
```

Pass `install: ""` to skip the install and get only a Python — the shape
`sigrix-io/postern`'s link checker needs, where the script is standard library
only and the repository root holds no installable package.

Pass `checkout: false` when the caller has already checked out — because it
needs options this action does not expose (a `ref`, a `fetch-depth`,
submodules), or because it is this repository testing its own actions.

### `verify-published-package`

After a release, installs the package from PyPI **by name** and checks it is
the version just published and carries its type marker.

```yaml
- id: published
  uses: sigrix-io/actions/verify-published-package@<sha>  # v1
  with:
    package: sigrix-mcp
    version: ${{ steps.tag.outputs.version }}
    import-name: sigrix_mcp

- run: ${{ steps.published.outputs.python }} -c "..."
```

An upload that succeeds is not the same as a package anyone can install, and
the two look identical from the publish step: a green tick.

## What belongs here, and what does not

`verify-published-package` is the worked example.

Waiting out the index, installing an exact version, and checking the package is
that version and carries `py.typed` are things **every** Sigrix package wants.
That is the action.

What tools `sigrix-mcp` serves, and what `mullion.__all__` resolves to, are
claims each package makes about *itself*. Those stay in the caller — which is
what the `python` output is for. A shared action that knew about `list_tools()`
would be a copy of one repository wearing a shared name.

## Testing

`.github/workflows/ci.yml` runs these actions rather than asserting about them:
against a throwaway project in `tests/fixture/`, on each supported Python, and
against a package that is already on PyPI.

`tests/fixtured` is not a stray file. `setup-python-project` expands its
`install` input unquoted, because word splitting is what lets `build twine`
arrive as two arguments — and unquoted also means bash globs, where `[dev]` is
a character class. That file is named to be exactly what `tests/fixture[dev]`
globs to, so removing `set -f` from the action turns CI red instead of turning
an extra into a filename.

## Licence

Apache-2.0. The Sigrix name and logo are not covered by the licence.
