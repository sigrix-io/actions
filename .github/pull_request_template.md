## What this changes

<!-- One or two sentences. The diff shows what changed; say why. -->

## Does a caller see it?

<!--
Callers pin a commit, so nothing reaches them until they bump; CHANGELOG.md is
how whoever reviews that bump decides whether it matters. Delete the rows that
do not apply.

- [ ] No: docs, CI or tests only. Neither action changes.
- [ ] An action changes, and every caller keeps working with the inputs it
      passes today.
- [ ] An input is renamed or removed, or its default changes. Name the callers
      that need an edit when they bump.
-->

## The CI step that fails without it

<!--
.github/workflows/ci.yml runs the actions rather than asserting about them.
Name the step that covers this change, or the one you added.
-->

## Checks

- [ ] Every `uses:` inside an action names a 40-character commit with its `# vX.Y.Z`
- [ ] A caller's input reaches a shell through `env:`, never written into `run:`
- [ ] `CHANGELOG.md` has a line under *Unreleased*
