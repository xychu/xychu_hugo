---
title: "Git Branching Model"
tag: ["git"]
date: 2018-04-28T17:00:56+08:00
draft: true
---

# Git Project branching & releasing model

In short, use `tag` to track every releases and use `branch` to keep developing parallelized.


## Two main branches

During the project development lifecycle, there will be two branches that are almost always there  and be used often. Also, they carry  all the release tags.

![master_release_branches](../../images/master_release.png)

### `master( or develop)` branch

We consider `origin/master` to be the main branch where the source code of `HEAD` always reflects a state with the latest delivered development changes for the next release. Some would call this the “integration branch”. This is where any automatic nightly builds are built from.

**NOTE:** It's better not to commit directly onto your local `master` branch, try to use feature branch instead. Keep a clean local `master` will give you strong confident to get changes from remote by `rebase` or even just `pull` `origin/master`, also save your time in case your changes conflict with others. But if you're a git expert and you know exactly where you're and what you're doing, that's another story :-)

### `release` branch

We consider `origin/release` to be the main branch where the source code of `HEAD` always reflects a `deploy-ready` state, including `release-candidate`.

When the source code in the `master` branch reaches a stable point and is ready to be released, all of the changes should be merged  into `release`  and then tagged with a release number.

Normally, it's shoud be  one release candidate with a  `rc` tag and deploy onto the  `stage` environment. During the test or verify stage, all release related commits, including bug-fixes, should be landed on the `release` branch **only**.

Therefore, when we decide it is production ready, we can bump version to a normal release, add tag and merge back into `master`.

**NOTE:** We use `release` branch to do the release procedure, including tests or verifications which may take days long, so the development that did not related or targeted to this release will keep moving forward on `master` branch. That's the point of using branches to keep developing parallelized.

## Other development branches

It's always okay and recommended to create personal or feature branches, just remember to delete them when they've got merged or retired.

And when bug or something bad happend on production environment, we need to do a hot-fix, this is the time we should use a `hotfix-<target_release_tag>` branch.

### personal or feature branch

Normally branch off from:
	 `master`
Must merge back into:
	`master`
Common steps:
```bash
# keep up-to-date with remote master
git fetch origin && git rebase --onto=master origin/master
# create feature branch,
git checkout -b myfeature master
# hack, hack, hack
...
# push to remote and create a pull request or merge request or a code review
git push origin myfeature
```

### hotfix branch

Must branch off from:
	`<target_release_tag>`
Must merge back into:
	`release` and `master`
Branch naming convention:
	`hotfix-<target_release_tag>`
Common steps:
```bash
# keep up-to-date with remote master
git fetch origin
# create hotfix branch,
git checkout -b hotfix-<buggy_release_tag> <buggy_release_tag>
# hot fix
...
# git tag and merge back to release and master after a quick verification on the fixes
git tag -a -m "[hotfix]: xxxx" <buggy_release_tag>-hotfix
git push origin hotfix-<buggy_release_tag>
# push hotfix git tag
git push origin <buggy_release_tag>-hotfix
```

![hotfix_branch](../../images/master_release_hotfix.png)

## Release procedure

* make sure all targetd feature commits have landed into the `master` branch

```bash
git fetch origin && git rebase --goto=master origin/master
```

* merge them into the `release` branch

```bash
git checkout release
git rebase master
# or
git rebase --goto=release master
```

* tag with release candidate tag(optional)

```bash
git tag -a -m "[rc]: xxxx" <release_tag>-rc
```

* deploy to test/stage environment
* merge release related commits if any
* when it's ready to become a normal release, add release tag

```bash
git tag -a -m "[release]: xxxx" <release_tag>
```

* merge back into `master` along with tag

```bash
# push release to remote, merge it back to master via PR or merge request or code review tool
git push origin release
git push origin <release_tag>
```

* deploy production environment with release tag


## Refs:

* http://nvie.com/posts/a-successful-git-branching-model/
* graph generated by: https://github.com/nicoespeon/gitgraph.js