# supergreen

Test "always green" master.

The aim of this repository is to demonstrate the minimal way to get an "always green" master, which means that master every merge to master will have always been tested against the exactly current version master, making it impossible to "break" master through concurrent pull requests. Namely, to solve the same thing [bors-ng](https://github.com/bors-ng/bors-ng) solved but in a much simpler way.

## Test setup

In this repository, we have a CI build configured via Github Actions that will first wait for two minutes, and then ensure that all lines present in `calls.txt` are also present in `defines.txt`. This is meant to demonstrate the case where a function is renamed via refactoring, changing all call sites, while at the same time somebody adds a new call site in a different pull. If pulls are not tested against current master, this could result in a merge to master which will not pass tests.

## Required settings

Set the following options in Settings:

- Options -> Merge button -> Allow squash merging: We only want to use squash merging in this sample. YMMV.
- Options -> Merge button -> Allow auto-merge: Pull author needs to be able to set up auto-merging when all requirements are fulfilled. Auto-merging is sticky for the pull, once it is set up.
- Options -> Merge button -> Automatically delete head branches: Automatically clean up litter. YMMV.
- Branches -> Branch protection rules (master) -> Require a pull request before merging: All merges via pulls.
- Branches -> Branch protection rules (master) -> Require approvals (1): No unreviewed changes in master ever.
- Branches -> Branch protection rules (master) -> Dismiss stale pull request approvals when new commits are pushed: Require review of new changes before merge, no unreviewed changes in master ever.
- Branches -> Branch protection rules (master) -> Require review from Code Owners: `CODEOWNERS` file is used to set up who can approve changes to which files in the repository. **This is the real access policy of this setup.**
- Branches -> Branch protection rules (master) -> Require status checks to pass before merging: All changes must pass tests to be merged.
- Branches -> Branch protection rules (master) -> Require branches to be up to date before merging (CI): When master changes (something else has been merged), all pulls need to be brough up to date before they can be merged. **This is the part that keeps the master supergreen.**
- Branches -> Branch protection rules (master) -> Require conversation resolution before merging: YMMV.
- Branches -> Branch protection rules (master) -> Require signed commits: YMMV.
- Branches -> Branch protection rules (master) -> Require linear history: Irrelevant with squash merges, but can be good as a safeguard if merges seen as a bad thing. YMMV.
- Branches -> Branch protection rules (master) -> Include administrators: If administrators are not included, then they are unable to use auto-merging etc. since they bypass requirements. Suggestion is to keep this on to have the same workflow for everyone, unless administrator is a special github user which isn't used for daily coding.

## Solving the missing piece

This setup will handle everything, except for the fact that when a pull is merged to master, all other pull request authors will need to manually go click "Update branch". We want all pull requests to automatically be updated when master changes. This can be achieved with [tibdex/auto-update@v2](https://github.com/tibdex/auto-update), but unfortunately changes done via the `GITHUB_TOKEN` [do not trigger new workflows](https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows#triggering-new-workflows-using-a-personal-access-token).

So instead we use [kodiak](https://github.com/chdsbd/kodiak) as a [Github App](https://github.com/marketplace/kodiakhq). To do this, we simply install the free app, and then add `.kodiak.toml`:

```
# .kodiak.toml
version = 1

[update]
always = true # default: false
require_automerge_label = false # default: true
```

We do not use the automerging functionality of kodiak, as Github handles everything else automatically for us, except the automatic update of the branches. In the future, [merge queues](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/using-a-merge-queue) will solve this use case, but currently they are not publicly available and do not support squash merges.

## Example pull

[#4](https://github.com/nakedible/supergreen/pull/4) is a pull request which was made concurrently with #3, and then automatically updated many times from master, until finally the kodiak merge commit triggered the build correctly and allowed the pull to auto-merge to master.
