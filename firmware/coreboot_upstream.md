# Working with coreboot Upstream and Chromium

Aaron Durbin

The following are notes on the workflow used by myself. There are likely better
ways so feel free to comment and/or add new recipes.

[TOC]

# Intro

Now that we are periodically syncing coreboot.org patches into the chromium repo
it has become even easier to work with [coreboot.org](coreboot.org) first.
However, it does take some git fu to flip back and forth. While it's certainly
possible to do all this in one local git checkout I find it easier to use two
coreboot git checkouts:

1. regular chromium checkout
2. coreboot.org checkout

The rest of the document will describe how to set those up and work between
them.

# coreboot checkouts

The most familiar local coreboot checkout is the one from chromium. It lives
under `src/third_party/coreboot` in the chromium workspace. On my machine it
lives at the full path of
`/home/adurbin/local/chromiumos-full/src/third_party/coreboot`. Additionally,
I keep a coreboot.org upstream checkout at the following location on my machine:
`/home/adurbin/devel/coreboot`.

In order to get an upstream checkout one can follow the instructions posted on
the coreboot.org wiki page [here](https://www.coreboot.org/Git). I usually just
crib the URL from the page and add it as a remote repository using the ssh
method. However, it's instructive to read over the page previously linked to get
an understanding of the things involved using coreboot's gerrit instance. For
the ssh method It first requires adding a public ssh key to associate with your
user on coreboot.org once you've registered. This can be achieved on
coreboot.org's gerrit account page:
<https://review.coreboot.org/#/settings/ssh-keys>

`.ssh/config` entry for ssh access to gerrit

```
Host review.coreboot.org
    Port 29418
    IdentityFile ~/.ssh/coreboot_upstream
```

Setting up remote coreboot.org repository in local coreboot.org git.

```bash
$ mkdir -p ~/devel/coreboot
$ cd ~/devel/coreboot
$ git init
Initialized empty Git repository in /home/adurbin/devel/coreboot/.git/
$ git remote add upstream ssh://review.coreboot.org:29418/coreboot.git
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
`/home/adurbin/local/chromiumos-full/src/third_party/coreboot` we can link the
two repositories using git remotes that are local to the system.

```bash
$ cd ~/devel/coreboot
$ git remote add cros-coreboot ~/local/chromium-full/src/third_party/coreboot
$ git fetch cros-coreboot

$ cd ~/local/chromium-full/src/third_party/coreboot
$ git remote add upstream_local ~/devel/coreboot
$ git fetch upstream_local
```

The two repositories now have remotes that track one another. This serves as the
basis for cherry-picking patches back and forth or rebasing commits from one
repository to the other.

When I typically develop a change I do my testing in the chromium tree since
Chrome OS will build images I can flash and boot on a machine I'm working on.
When I'm sufficiently happy with a commit it's time to push to coreboot.org.
Assuming that the patch was done in a branch called feature1 using the normal
repo workflow `repo start feature1 .` within the chromium coreboot repository
there will be a branch named feature1 that is visible to the coreboot.org local
repo after a fetch.

```bash
$ cd ~/devel/coreboot
$ git fetch cros-coreboot
remote: Counting objects: 4204, done.
remote: Compressing objects: 100% (1229/1229), done.
remote: Total 2495 (delta 2069), reused 1576 (delta 1245)
Receiving objects: 100% (2495/2495), 456.72 KiB | 0 bytes/s, done.
Resolving deltas: 100% (2069/2069), completed with 519 local objects.
From /home/adurbin/local/chromiumos-full/src/third_party/coreboot
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
