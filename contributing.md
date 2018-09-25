# Chromium OS Contributing Guide

For new developers & contributors, the [Gerrit] system might be unfamiliar.
Even if you're already familiar with [Gerrit], how the Chromium project uses it
might be different due to the [CQ]/trybot integration and different labels.

For a general [Gerrit] overview, check out Android's [Life of a Patch].
This helpfully applies to everyone using [Gerrit].

If you haven't checked out the source code yet, please start with the
[Developer Guide].
Once you've got a local checkout, come back here.

[TOC]

## Account setup

Please follow the [Gerrit Guide] for getting access to the [Gerrit] instances.
Once that is all set up, you can come back here.

### Sign a Contributor License Agreement

Before uploading [CL]s, you'll need to submit a [Contributor License Agreement].
Review the "Legal stuff" section in [Contributing Code] document for more info.

For Googlers, please see the [internal CLA documentation] for more details,
especially when working with partners.

## Commit messages

For general documentation for how to write git commit messages, check out
[How to Write a Git Commit Message](https://chris.beams.io/posts/git-commit/).

As a quick overview, here's what a sample description should look like.
It will show up in its entirety in [Gerrit], and the first line will be used as
the subject line for the review (e.g. in e-mail notifications).

```
some-prefix: Here's a SHORT, one-line summary of my change

And here are more details
...this can be as long as I want.

BUG=b:99999, chromium:88888
TEST=Ran all the box tests

Change-Id: I8d7f86d716f1da76f4c85259f401c3ccc9a031ff
```

If you're unsure of the form to use in a particular repo, look at the recent
commit log via `git log` to get a sense for local customs.

### Link to issue trackers

Issue trackers are critical to the smooth running of the project, both for
tracking bugs/regressions as well as new features.
When making changes that are related to an issue (open or closed), the commit
message should link to the relevant issue via `BUG=` lines.

The general form is `BUG=bug-tracker:number`.
The Chromium OS project has supported various bug trackers over the years, but
currently there are 2 supported trackers: one at [crbug.com], for which you
should use the prefix `chromium:`, and one at [issuetracker.google.com] (see
[issue tracker]; internally known as Buganizer), for which you should use the
prefix `b:`.
If your changes are related to more than one issue, you can list all the
issues separated with commas, or include multiple `BUG=` lines.

The `BUG=` lines should be separated by the rest of the commit message by a
blank line, and should come before the `TEST=` lines (see below).

### Describe testing performed

When evaluating [CL]s, other developers want to know what kind of tests were
performed to make sure the code behaves as expected.
Reviewers who are familiar with these code paths can often suggest alternative
tests to run in case the ones run were not adequate.

These are described in the commit message with `TEST=` lines.
These come directly after the `BUG=` lines and are generally free-form.
There should be a blank line between them and the [Change-Id] tag at the end.

Some common examples:

*   `TEST=None`: Used if the [CL] in question doesn't need testing (e.g. fixing
    typos in documentation files).
*   `TEST=ran unittests`: Implies the unittests in the package are sufficient to
    prove functionality.
*   `TEST=precq passes`: Implies all unittest/vmtests/hwtests passed, and those
    tests are sufficient to validate the code.

### Change-Id

It is important to note that repo uses the [Change-Id] in your git commit
message to track code reviews.
So if you want to make a change to an existing CL, you'll want to use `git
commit --amend` rather than making an entirely new commit.

This allows you to follow the standard git flow by making multiple changes in
a single branch and uploading them together.

For more details, see [Gerrit]'s [Change-Id] documentation.

### CL dependencies {#CQ-DEPEND}

Sometimes work will span multiple [CL]s across different repos.
The `CQ-DEPEND=` lines are used to make sure [CL]s are merged in a specific
order, or altogether vs none at all.

*   Specify dependencies in the last paragraph of your change (after `TEST=`
    lines) using `CQ-DEPEND=CL:12345`.
*   Each dependency should start with a `CL:` prefix followed by a number
    (the [Gerrit] [CL] number on the server) or a [Change-Id].
*   You may specify multiple dependencies.
    Each dependency should be separated with a comma and a space (e.g.
    `CQ-DEPEND=CL:12345, CL:4321`).
    You can also split dependencies into multiple `CQ-DEPEND=` lines.
*   Use an asterisk prefix to denote internal dependencies (e.g. `CL:*4321`).
    Otherwise, the [CL] is interpreted as an external change (e.g. `CL:4321`).
*   You may specify `CQ-DEPEND` loops where "CL A" depends on "CL B", and "CL B"
    depends on "CL A" (there's no limit to the number of [CL]s).
    This makes sure the [CQ] will pick up and test them together atomically.
*   Atomic transactions within a single repository are supported.
    *   Merges across repos is not atomic due to [Gerrit] limitations.
        There is a small window where syncs may pull a partial set of changes.

Here's an example:

```
Add file to install to 9999 ebuild file

BUG=chromium:99999
TEST=Tested with dependent CL's in trybot.
CQ-DEPEND=CL:12345, CL:*4321

Change-Id: I8d7f86d716f1da76f4c85259f401c3ccc9a031ff
```

## Upload changes

Once your changes are committed locally, you upload them using `repo upload`.
This command takes all of the changes that are unmerged, runs preupload checks
on them, asks them if you want to upload them, and then publishes them.

By default, `repo upload` looks across all branches & projects, so most of the
time you want to restrict this to the local repo instead:

```sh
# Command most people will use most of the time.
$ repo upload --cbr .

# General format.  See `repo upload --help` for more.
$ repo upload [.|${PROJECT-NAME}] [--current-branch] [--reviewers=REVIEWERS]
```

You'll often want to specify reviewers using the `--re` option, but don't worry
if you didn't specify it here as they can be added later.
See the [Adding Reviewers] section for more details.

Once you run `repo upload`, this uploads the changes and prints out a URL for
the code review for easy access.

For more in-depth details, check out [Gerrit]'s [Uploading Changes] document.

## Going through review

### Start the review

When [CL]s start in [Work-in-Progress (WIP)] mode, people are not notified.
You'll need to go to the web interface and click the **Start Review** button.
There you can add reviewers and comments before clicking the next **Start
Review** button.

If the [CL] is not in WIP mode, as soon as the [CL] is uploaded, notifications
are sent out to any reviewers (if `--re` is used).
You can still go to the web interface and click the **Reply** button to add
reviewers and comments before clicking the **Send** button.

### Add reviewers {#reviewers}

You should pick reviewers that know the code you're working on well and that
will do the best reviews.
Picking reviewers who will just rubber-stamp your changes is a bad idea.
The point of submitting changes is to submit good code, not to submit as much
code as you can.

If you don't know who should review your changes, start by looking for OWNERS
files in directories your commit touches.
These are great for finding the maintainers for the respective projects.

If there are no OWNERS files, you can use `git log` to find people.
You can use it on the specific files you're touching, or on the entire project.
Simply type the commands below in a directory related to your project:

```shell
$ git log <file>
$ git log <directory>
```

### Address feedback

Your reviewers will likely provide comments about changes that you should make
before submitting your code.
You should make such changes, commit them locally, and then re-upload your
changes for code review.

You can amend your changes to your git commit and re-run `repo upload`.

```shell
# <make some changes>
$ git add -u .
$ git commit --amend
```

If you have a chain of commits (which `repo upload .` converts to a chain of
[CL]s), and you need to modify any commits that are not at the top of the chain,
use interactive rebase:

```shell
$ git rebase -i
# This shows a list of cherry-picks into a temporary branch.
# Change some of the "pick" keywords to "edit".  Then exit the editor.

# Look at the first "edit"ed commit.  All earlier commits are cherry-picked.
$ git log

# Make some modifications.
$ git add -u .
$ git commit --amend
# Move on to the next "edit"ed commit.
$ git rebase --continue

# Finally upload when all modifications are ready.
$ repo upload . --current-branch
```

### Getting Code-Review

Ultimately, the point of review is to obtain both look good to me (LGTM) and
approved statuses in your [CL]s.
Those are tracked by the Code-Review+1 (LGTM) and Code-Review+2 (LGTM &
approved) [Gerrit] labels.

Reviewers use the Code-Review+2 label to say the [CL] looks good (LGTM), and
the reviewer is also approving it as they're fully OK with it.

Some reviewers might be OK with the [CL], but they aren't comfortable approving
it (e.g. they aren't that familiar with the particular piece of code).
They'll add a Code-Review+1 label and wait for someone else to approve it.

Only once you've obtained a Code-Review+2 label can you move on.
Note that two Code-Review+1 labels does not equal a Code-Review+2.
It simply means more than one person said the code looks good, but they all want
someone else to approve the [CL].

### Setting verified

Some reviewers, depending on how they reviewed things, might add the Verified+1
label to indicate that they also tested/verified the [CL].
This is not requirement for them and is entirely their own preference.

Ultimately it's your responsibility to mark the [CL] as Verified+1 to indicate
that all your testing has passed.
It's recommended you do this even if other reviewers set Verified+1 themselves.

## Send your changes to the Commit Queue

Once you've got Code-Review+2 (LGTM & approved) and Verified+1 labels, it's time
to try to merge it into the tree.
You'll add the Commit-Queue+1 label so the [CQ] will pick it up.
If the [CL] doesn't have Code-Review+2, Verified+1, and Commit-Queue+1 labels,
then the [CL] will never be picked up by the [CQ].
Further, if someone adds Code-Review-2 or Verified-1, the [CQ] will ignore it.

Before the [CQ] picks up your [CL] it must pass the pre-CQ.
The pre-CQ is triggered automatically when your [CL] is marked Code-Review+2.
You can trigger this earlier by adding the Trybot-Ready+1 label yourself.
If the pre-CQ passes, it will not be required again before the [CQ] runs.

More details on the Commit Queue can be found in the [Commit Queue
Overview][CQ].

### Merge conflicts

It is possible that your change will be rejected because of a merge conflict.
If it is, rebase against any new changes and re-upload your [CL].

### Updating CLs after Code-Review+2

Whenever non-trivial changes are made to a [CL], all labels are cleared.
This means a Code-Review+2 label must be attained again.
For developers with access, they'll often apply Code-Review+2 to their own
[CL] with a comment like "inheriting CR+2 from previous patch".
The expectation here is that the developer hasn't made significant changes that
the reviewers would have objected to.

If the developer doesn't have access, they'll have to get approvals from the
reviewers again.

If only trivial changes are made, then the Code-Review+2 labels will be sticky.
The kind of trivial changes are:

* Rebases onto newer commits without conflicts.
* Changes to the commit message.

### Make sure your changes didn't break things

If all testing passes, the [CQ] will merge your [CL] directly.
If something did go wrong, the [CQ] will post details of the run.
This will often include a lot of logs that you're expected to go through and
make sure the failure wasn't due to your [CL].

If you're confident your [CL] was not at fault, simply add Commit-Queue+1 again.

Often times the [CQ] and sheriffs will triage a failed [CQ] run and mark all the
unrelated [CL]s are Commit-Queue+1 again for you.

If you're still unsure, feel free to reach out to the sheriffs or reviewers.

## Bypass the commit queue (chumping)

Rarely should you bypass the [CQ] (aka "chumping a [CL]").
Doing so puts the system at risk by including a [CL] that hasn't been properly
tested on devices.
This should be reserved for people trying to fix existing breakage, and should
be coordinated with the sheriffs.

You can do this by hitting the **Submit** button when a [CL] has Code-Review+2
and Verified+1.

## Clean up

After you're done with your changes, you're ready to clean up.
You'll want to delete the branch that repo created.
There are a number of ways to do so; here is one way:

```shell
# Command most people will use most of the time; run it in the project.
$ repo abandon ${BRANCH_NAME} .

# General format.  See `repo abandon --help` for more.
$ repo abandon ${BRANCH_NAME} ${PROJECT-NAME}
```

*** note
**Warning**: If you don't specify a project name, the `repo abandon` command
will throw out any local changes across **all projects**.
You might also want to look at `git branch -D` or `repo prune`.
***

## Advanced topics

### Share your changes using the Gerrit sandbox

It is possible to upload changes to a personal sandbox on [Gerrit].
This way, changes can be shared between developers before it's ready for review.

*** note
The sandbox spaces are **not** private.
Anyone can find & access commits posted here.
Do not use this to hold secret work.

You're free to create as many branches as you want under your own namespace
`refs/sandbox/${USER}/`.
We leave it up to your discretion to properly manage these branches.
Please don't abuse it by uploading large binary files that don't belong in git.

Further, the sandbox spaces are a bit loose with access.
You can push to any path under `refs/sandbox/`, but we've all agreed to restrict
ourselves to the `${USER}` subdir.
***

```shell
$ project_url="https://chromium.googlesource.com/$(git config remote.cros.projectname)"
$ git push ${project_url} HEAD:refs/sandbox/${USER}/${BRANCH_NAME}
```

Other developers can then fetch your changes using the following commands:

```shell
$ project_url="https://chromium.googlesource.com/$(git config remote.cros.projectname)"
$ git fetch ${project_url} refs/sandbox/${USER}/${BRANCH_NAME}
$ git checkout FETCH_HEAD
```

In a given repository, you can explore sandboxes using the `ls-remote` command:

```shell
$ git ls-remote cros "refs/sandbox/${USER}/*"
$ git ls-remote cros "refs/sandbox/*"
```

Once you're finished with a sandbox, you can delete it:

```shell
$ project_url="https://chromium.googlesource.com/$(git config remote.cros.projectname)"
$ git push $project_url :refs/sandbox/${USER}/${BRANCH_NAME}
```

### Switch back to master/ToT

While you're working on your changes, you might want to go back to the mainline
for a little while (maybe you want to see if some bug you are seeing is related
to your changes, or if the bug was always there).
If you want to go back to the mainline without abandoning your changes, you can
run the following commands from within a directory associated with your project.

*** note
The `m/master` is a ref managed by `repo` to point to the right remote and
branch for the particular repo.
The remote name (e.g. `cros` or `cros-internal`) depends on where the repo is
hosted, and the remote branch name (e.g. `master`) depends on what the specific
project is using for its current development branch.
Thus `m/master` should point to the right branch regardless.
***

```shell
$ git checkout m/master
```

When you're done, you can get back to your changes by running:

```shell
$ git checkout ${BRANCH_NAME}
```

Take care when running `repo sync` in case it switches branches on you again.

### Work on something else while waiting for reviews

If you want to start on another (unrelated) change while waiting for your code
review, you can `repo start` to create another branch.
When you want to get back to your first branch, run the following command from
within a directory associated with your project:

```shell
$ git checkout ${BRANCH_NAME}
```

[Adding Reviewers]: #reviewers
[Change-Id]: https://gerrit-review.googlesource.com/Documentation/user-changeid.html
[CL]: https://dev.chromium.org/glossary
[Contributing Code]: https://dev.chromium.org/developers/contributing-code#TOC-Legal-stuff
[Contributor License Agreement]: https://cla.developers.google.com/
[CQ]: https://dev.chromium.org/developers/tree-sheriffs/sheriff-details-chromium-os/commit-queue-overview
[crbug.com]: https://crbug.com/
[Developer Guide]: developer_guide.md
[Gerrit]: https://gerrit-review.googlesource.com/Documentation/
[Gerrit Guide]: https://dev.chromium.org/chromium-os/developer-guide/gerrit-guide
[internal CLA documentation]: https://chrome-internal.googlesource.com/chromeos/docs/+/master/signcla.md
[issue tracker]: https://developers.google.com/issue-tracker/
[issuetracker.google.com]: https://issuetracker.google.com/
[Life of a Patch]: https://source.android.com/setup/contribute/life-of-a-patch
[Uploading Changes]: https://gerrit-review.googlesource.com/Documentation/user-upload.html
[Work-in-Progress (WIP)]: https://gerrit-review.googlesource.com/Documentation/intro-user.html#wip
