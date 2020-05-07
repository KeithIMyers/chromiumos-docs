# Working with Coreboot upstream and Chromium

[TOC]

## Introduction

Chrome OS uses and actively contributes to [coreboot.org]. Development happens
in the chromium copy of the coreboot repo, and the resulting patch must be
pushed to upstream coreboot. While this can be done in one local git checkout,
many developers find it easier to use two coreboot git checkouts:

1. regular chromium checkout
2. coreboot.org checkout

This document describes how to set up a second coreboot.org upstream checkout
and how to synchronize patches between the two checkouts.

## Setting up a second coreboot checkout

The most familiar local coreboot checkout is the one from chromium. It lives
under `src/third_party/coreboot` in the chromium workspace. If you followed the
[ChromeOS Developer guide], it lives at the full path of
`~/chromiumos/src/third_party/coreboot`. The following steps will add an
additional coreboot.org upstream checkout at the secondary location on your
machine: `~/devel/coreboot`.

First, check that you have a username and public ssh key that will be
associated with your user on coreboot.org once you've registered. This can be
done on coreboot.org's [Gerrit account page]. If you need to generate a public
ssh key, follow the instructions on [Gerrit's documentation].

Add the following entry to your `.ssh/config` for ssh access to Gerrit:

```
Host review.coreboot.org
    Port 29418
    IdentityFile ~/.ssh/coreboot_upstream
```

Next, set up a remote coreboot.org repository in local coreboot.org git. In the
bash excerpt below, `______` will be the username on your operating system, and
`{USERNAME}` should be your username on Gerrit.

```bash
$ mkdir -p ~/devel/coreboot
$ cd ~/devel/coreboot
$ git init
Initialized empty Git repository in /home/______/devel/coreboot/.git/
$ git remote add upstream ssh://{USERNAME}@review.coreboot.org:29418/coreboot.git
$ git fetch upstream
```

After the above all the objects have been pulled down, but there isn't a
checkout. One can create a local branch tracking coreboot.org's master branch:

```bash
$ git checkout -b upstream_master upstream/master
```

Now the local checkout is tracking `upstream/master` on the local
`upstream_master` branch.

Assuming there is a chromium coreboot checkout at
`~/chromiumos/src/third_party/coreboot`, you can link the two repositories using
git remotes that are local to the system.

```bash
$ cd ~/devel/coreboot
$ git remote add cros-coreboot ~/chromiumos/src/third_party/coreboot
$ git fetch cros-coreboot

$ cd ~/chromiumos/src/third_party/coreboot
$ git remote add upstream_local ~/devel/coreboot
$ git fetch upstream_local
```

The two repositories now have remotes that track one another. This serves as
the basis for cherry-picking patches back and forth or rebasing commits from
one repository to the other.

## Developing with two coreboot checkouts

Developing should be done in the chromium tree since Chrome OS can build images
to test by flashing and booting on a machine.  When a commit is ready in the
chromium tree, it's time to push to coreboot.org.  Assuming that the patch was
done in a branch called feature1 using the normal repo workflow `repo start
feature1 .` within the chromium coreboot repository, there will be a branch
named feature1 that is visible to the coreboot.org local repo after a fetch.

```bash
$ cd ~/devel/coreboot
$ git fetch cros-coreboot
remote: Counting objects: 4204, done.
remote: Compressing objects: 100% (1229/1229), done.
remote: Total 2495 (delta 2069), reused 1576 (delta 1245)
Receiving objects: 100% (2495/2495), 456.72 KiB | 0 bytes/s, done.
Resolving deltas: 100% (2069/2069), completed with 519 local objects.
From /home/______/chromiumos/src/third_party/coreboot
 * [new branch]      feature1   -> cros-coreboot/feature1
```

Now it's possible to cherry-pick or rebase the chromium patches into a branch
that is tracking coreboot upstream.

```bash
$ git cherry-pick cros-coreboot/feature1
```

And the patch can be pushed to coreboot.org

```bash
$ git push upstream HEAD:refs/for/master
```
[coreboot.org]: https://coreboot.org
[ChromeOS Developer guide]: ./../developer_guide.md
[Gerrit account page]: https://review.coreboot.org/#/settings/ssh-keys
[Gerrit's documentation]: https://gerrit-documentation.storage.googleapis.com/Documentation/2.14.2/user-upload.html#ssh
