# Setting up a new public repository

How the Sigrix public repositories are configured, written down so the next one
starts from the same place rather than from memory. `sigrix-mcp`, `mullion` and
`postern` follow it, and so does this repository.

Most of a repository's setup is files: they go through pull requests, and CI
notices when one is wrong. **The branch ruleset is the exception.** It lives in
the repository's settings, where no pull request reviews it and nothing notices
when it drifts. So this document writes it out in full, together with the `gh`
commands that apply it, change it and check it.

The main commands here (apply, look, change in place, verify, delete) were used
to bring `sigrix-mcp`, `mullion`, `postern` and `actions` into line. Each of the
example filters was also run over a copy of what the API returns, to check its
output.

You need [`gh`](https://cli.github.com/), signed in as an org admin.

## The branch ruleset

Every public repository has **exactly one** ruleset, named `default-branch`,
targeting the default branch. [`default-branch.ruleset.json`](default-branch.ruleset.json)
is the canonical copy.

| Rule | Setting | Why |
|---|---|---|
| Restrict deletions | on | `main` cannot be deleted. |
| Block force pushes | on | History on `main` is never rewritten. |
| Require a pull request | on, **0 approvals** | Every change lands through a reviewable PR. Approvals are 0 because a PR's author can't approve their own PR, and one maintainer can't produce a second person. |
| Require review from Code Owners | **off** | Each `CODEOWNERS` file assigns every path to a team. With one maintainer, that asks for an approval nobody else can give, so every PR they open is blocked. That includes every PR an agent opens on their behalf, because those are opened under the maintainer's account. |
| Require approval for unattributed changes | **off** | Same problem. When this rule applies, it asks for an approval from someone other than the author. Going by its name, it applies to commits that aren't attributed to the PR's author, such as those an agent writes as `Claude <noreply@anthropic.com>`. |
| Require approval of the most recent push | off | Same reason: it needs someone other than the pusher. |
| Required status check | the repository's aggregate job, bound to GitHub Actions (`integration_id: 15368`) | **One** required check, so adding a CI job never means editing the ruleset. Binding it to the Actions app means no other app can post a green check with that name. |
| Require branches to be up to date | on | Without it, a PR can pass against an old `main` and break the new one on merge. |
| Bypass | team `19595700`, **`pull_request` mode** | An escape hatch for merging past a broken check. `always` mode would also let that team force-push and delete `main`, which quietly switches off the first two rules for exactly the people who push. |

To see which team `19595700` is:

```sh
gh api orgs/sigrix-io/teams --jq '.[] | select(.id == 19595700) | .slug'
```

**Once there is a second maintainer,** turn code-owner review and the
unattributed-changes approval back on (see "Change an existing ruleset"). They
are the right rules as soon as someone other than the author can satisfy them,
and the second one in particular is exactly the guard you want on agent-authored
changes.

### Name the aggregate CI job `ci-passed`

`sigrix-mcp`, `mullion` and this repository all do, so the ruleset applies to
them unchanged. `postern` is older than the convention and requires `validate`
and `conformance-status` instead. A new repository should use `ci-passed`: one
job that `needs:` every other job and fails if any of them failed or were
cancelled. `.github/workflows/ci.yml` here has one to copy.

A required check with a name that nothing reports never turns green, so every PR
waits on it forever. Don't require individual matrix jobs like `test (3.12)`:
the next Python you add to the matrix won't be required until someone edits the
ruleset.

## Apply it to a new repository

Check that the repository doesn't already have a ruleset:

```sh
NEW=my-new-repo
gh api repos/sigrix-io/$NEW/rulesets --jq '.[] | "\(.id)  \(.name)  \(.target)"'
```

**If that prints a ruleset, stop here and don't `POST`.** A `POST` next to an
existing ruleset creates a second one, and `mullion` once ended up with two
exactly this way. Rulesets stack, and the stricter rule from either one wins.
Replace the existing ruleset instead: see "Change an existing ruleset" below.

If it prints nothing, create the ruleset. From a clone of this repository:

```sh
gh api -X POST repos/sigrix-io/$NEW/rulesets --input docs/default-branch.ruleset.json
```

Or, without a clone, copy it from a repository that already has it. This is how
`mullion` got its ruleset. It looks up the source ruleset's ID by name:

```sh
SRC=$(gh api repos/sigrix-io/sigrix-mcp/rulesets --jq '.[] | select(.name == "default-branch") | .id')
gh api repos/sigrix-io/sigrix-mcp/rulesets/$SRC --jq '{name, target, enforcement, conditions, rules, bypass_actors}' | gh api -X POST repos/sigrix-io/$NEW/rulesets --input -
```

The copy takes whatever the source repository has *now*, including any change
made in its settings since. The file in this repository is the version that
went through review. If the two ever disagree, that's drift, and the Verify
section below is how you'd see it.

If the aggregate job isn't called `ci-passed`, apply the ruleset anyway and then
swap the check using the first example below.

## Change an existing ruleset

### Look first

`PUT` replaces the **whole** ruleset, and any rule you don't send is deleted.
That is how `actions` once lost a `required_linear_history` rule nobody meant to
remove. So check what's there before replacing it:

```sh
R=sigrix-mcp
gh api repos/sigrix-io/$R/rulesets --jq '.[] | "\(.id)  \(.name)  \(.target)"'
```
```sh
ID=23725824
gh api repos/sigrix-io/$R/rulesets/$ID --jq '{name, include: .conditions.ref_name.include, rules: [.rules[].type]}'
```

### Fetch, change, write back

Write the change as a `jq` filter, preview the result, then send it back.
Nothing has to be pasted, which matters because pasting a large JSON block into
an interactive shell can mangle it (see "When something is off"). Each filter
keeps only the fields the API accepts in a `PUT` and drops the `id`, `node_id`
and `_links` that a `GET` returns.

First set `F` to one of these:

**Swap the required checks** (this is what `postern` uses):

```sh
F='{name, target, enforcement, conditions, bypass_actors, rules: [.rules[] | if .type == "required_status_checks" then .parameters.required_status_checks = [{context: "validate", integration_id: 15368}, {context: "conformance-status", integration_id: 15368}] else . end]}'
```

**Turn a pull-request setting on or off.** Here, code-owner review goes back on
once there's a second maintainer:

```sh
F='{name, target, enforcement, conditions, bypass_actors, rules: [.rules[] | if .type == "pull_request" then .parameters.require_code_owner_review = true else . end]}'
```

**Add a rule.** Here, linear history. That means no merge commits, so the
allowed merge methods have to drop `merge` as well:

```sh
F='{name, target, enforcement, conditions, bypass_actors, rules: ([.rules[] | if .type == "pull_request" then .parameters.allowed_merge_methods = ["squash", "rebase"] else . end] + [{type: "required_linear_history"}])}'
```

Then preview the result, which changes nothing:

```sh
gh api repos/sigrix-io/$R/rulesets/$ID --jq "$F"
```

If the preview is right, apply it. It prints the updated ruleset back:

```sh
gh api repos/sigrix-io/$R/rulesets/$ID --jq "$F" | gh api -X PUT repos/sigrix-io/$R/rulesets/$ID --input -
```

## Verify

This reads the rules **actually in force** on each default branch, from every
ruleset at every level, including org-level ones that a repository's own
ruleset list doesn't show:

```sh
for r in sigrix-mcp mullion postern actions; do
  echo "== $r =="
  gh api "repos/sigrix-io/$r/rules/branches/main" --jq '{
    rulesets:     ([.[].ruleset_id] | unique),
    rules:        ([.[].type] | unique),
    approvals:    [.[] | select(.type=="pull_request") | .parameters.required_approving_review_count],
    code_owner:   [.[] | select(.type=="pull_request") | .parameters.require_code_owner_review],
    unattributed: [.[] | select(.type=="pull_request") | .parameters.require_extra_approval_for_unattributed_changes],
    last_push:    [.[] | select(.type=="pull_request") | .parameters.require_last_push_approval],
    checks:       [.[] | select(.type=="required_status_checks") | .parameters.required_status_checks[].context],
    up_to_date:   [.[] | select(.type=="required_status_checks") | .parameters.strict_required_status_checks_policy]
  }'
  for id in $(gh api "repos/sigrix-io/$r/rulesets" --jq '.[].id'); do
    gh api "repos/sigrix-io/$r/rulesets/$id" --jq '{name, enforcement, bypass: [.bypass_actors[]? | .bypass_mode]}'
  done
done
```

Add the new repository to the list. A healthy repository shows:

| Field | Expect |
|---|---|
| `rulesets` | **one** ID. Two means rulesets are stacked. |
| `rules` | `deletion`, `non_fast_forward`, `pull_request`, `required_status_checks` |
| `approvals` / `code_owner` / `unattributed` / `last_push` | `[0]` / `[false]` / `[false]` / `[false]` |
| `checks` | `["ci-passed"]` (for `postern`: `["validate", "conformance-status"]`) |
| `up_to_date` | `[true]` |
| name / enforcement / bypass | `default-branch` / `active` / `["pull_request"]` |

Each field is a list on purpose. One entry means one rule. Two entries means two
rulesets are stacked on the same branch.

## When something is off

| You see | It means |
|---|---|
| `Branch not protected (HTTP 404)` from `…/branches/main/protection` | That endpoint only covers the older *classic* branch protection, and this repository is protected by a ruleset instead. Use `…/rules/branches/main`. |
| A PR reads `blocked` with every check green | Either something wants an approval nobody can give (code owners, unattributed changes, last push), or a required check name is one nothing reports. The merge box on the PR says which. |
| A PR reads `behind` | The up-to-date rule is doing its job; the branch needs updating. For a Dependabot PR, comment `@dependabot rebase` rather than pressing **Update branch**. The button pushes a merge commit onto Dependabot's branch, and Dependabot stops maintaining a PR once someone else has changed it. |
| `mergeable_state: unknown` | GitHub works this out only when asked, and it can lag straight after a push. Read it again before concluding anything. |
| `Problems parsing JSON (HTTP 400)` after pasting JSON into the terminal | The shell mangled the paste; a long heredoc can arrive spliced and duplicated. Nothing changed. Use `--input <file>` or the fetch–change–write-back pattern instead. |
| `open …json: no such file or directory` | `--input` paths are relative to the current directory. Downloaded files usually land in `~/Downloads`. |
| A rule disappeared after a `PUT` | `PUT` replaces the whole ruleset. Check with "Look first" before replacing anything. |
| Two rulesets on one branch | A `POST` was run where a `PUT` was meant. Delete the stale one with `gh api -X DELETE repos/sigrix-io/<repo>/rulesets/<id>`, which prints nothing when it succeeds. |

## The rest of a new repository

These are files, so they go through pull requests like everything else. Copy
them from `sigrix-mcp`, the most recently aligned repository, and adjust:

- **Community files:** `CODE_OF_CONDUCT.md` (byte-identical across the
  repositories), `SECURITY.md`, `CONTRIBUTING.md`, and a team in `CODEOWNERS`.
- **Issue forms and PR template:** `.github/ISSUE_TEMPLATE/`, with blank issues
  turned off in its `config.yml`, and `.github/pull_request_template.md`.
- **Dependabot:** `.github/dependabot.yml` with the `github-actions` ecosystem at
  `/`, plus the package ecosystem the repository uses.
- **CI:** every job calls `setup-python-project` from this repository, pinned to
  a 40-character commit with a `# vX.Y.Z` comment. An aggregate job named
  `ci-passed` sits at the end.
- **Pin guard:** `tests/test_workflow_pins.py`, which fails if any `uses:` loses
  its commit pin or its version comment.
- **Release:** trusted publishing through a `pypi` environment. Create the
  environment in the repository's settings *and* the trusted publisher on PyPI;
  neither complains if it's missing until the first tag is pushed. After
  publishing, a `verify` job runs `verify-published-package`.
- **The ruleset:** everything above this section.
