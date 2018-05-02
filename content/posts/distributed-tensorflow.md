---
title: "Distributed Tensorflow and more"
tag: ["AI", "Deep Learning", "tenforflow", "Distributed System"]
date: 2018-04-28T23:00:36+08:00
draft: true
---

# Tensorflow 分布式训练的各种玩儿法 - 蹭 1.8 的热度

Tensorflow 1.8 发布了！ 保持着差不多一个月一个版本，够可以的！完整 Release Note 请移步 [Github](https://github.com/tensorflow/tensorflow/releases/tag/v1.8.0)。

抛开 Tensorflow Lite 不说，我特别关心的是这一条：

> Can now pass `tf.contrib.distribute.MirroredStrategy()` to `tf.estimator.RunConfig()` to run an `Estimator` model on multiple GPUs on one machine

解读下：TF 进一步抽象的高层（high level） API `Estimator` 通过 `MirroredStrategy()` 支持单机多卡分布式了（In-graph replication, all-reduce synchronous）。我们有理由相信，后面应该会有更多的分布式策略被支持，多节点，异步，甚至模型并行。

而我们的目标呢，基于目前的机房建设在某些场景下肯定是多机多卡才够劲儿。 所以，今天就简单总结下，我接触 Tensorflow 以来了解到的分布式训练的各种玩儿法，以及接下来会继续跟进的几个方向。

## 经典 ps & worker 模式

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

## All-reduce

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

## horovod

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


## More


## Refs:
* http://nvie.com/posts/a-successful-git-branching-model/
* https://github.com/nicoespeon/gitgraph.js
