# Tips And Tricks for Chromium OS Developers

This page contains tips and tricks for developing on Chromium OS.

The free-form content on this page is contributed by developers. Before adding
content to this page, consider whether the content might fit better in a
different document, such as the [Chromium OS Developer Guide] or
the [Chromium OS Developer FAQ]:

*   **Content that belongs in this document:**
    *   tips that developers can use to optimize their workflow, but that aren't
        strictly required
    *   instructions to help developers explore/understand the build environment
        in greater depth
*   **Content that does not belong in this document:**
    *   things that every developer needs to know right away when they start
        developing
        *   Put such information in the [Chromium OS Developer Guide].
    *   things that only a very small subset of developers need to know
        *   Put such information on a page dedicated to that small subset of
            developers.

*** note
**Note:** The tips on this page generally assume that you've followed the
instructions in the [Chromium OS Developer Guide] to build your image.
***

[TOC]

## Device Tips

### How to get more commands available on the release image

If you're using the shell in a Chromium OS device (requires developer mode, or a
system where you've set a custom `chronos` password), you may find that you're
missing commands you wish you had. Try putting `busybox` on a USB key or an SD
Card. I formatted by USB key with `ext3` and named it `utils` (and that is
assumed in these instructions) using the Disk Utility that comes with Ubuntu (go
to the System menu at the top of the screen, then Administration). Since busybox
is built as part of a normal build (it's used in the factory image), you can
copy it to your USB key like this (run from outside the chroot).

```bash
sudo chown root:root /media/utils
sudo chmod o+rx /media/utils
sudo cp ~/chromiumos/chroot/build/${BOARD}/bin/busybox /media/utils/
sudo chmod o+rx /media/utils/busybox
```

Then, you can go add symlinks for whatever your favorite busybox commands are:

```bash
for cmd in less zip; do
  sudo ln -s busybox /media/utils/$cmd
done
```

Unmount your USB key:

```bash
sudo umount /media/utils
```

Plug your USB key into your Chromium OS device. Make sure that the browser has
started (i.e., you're not still on the login screen) so that Chromium OS will
mount your USB key to `/media/utils`. You can get access to your commands with:

```bash
sudo mount -o remount,exec /media/utils
export PATH="$PATH:/media/utils"
```

If you want a list of all of the commands that `busybox` supports, you can just
run `busybox` to get a list. If you want to run a busybox command without
making a symlink, just type `busybox` first. Like: `busybox netstat`.

*** note
**SIDE NOTE**: If you didn't build Chromium OS, it's possible that you can run a
busybox binary obtained from some other source (e.g.
[official binaries](https://busybox.net/downloads/binaries/]). You'll have the best luck if
the busybox that you get was statically linked.
***

### How to wipe the stateful partition

You can use:

```bash
crossystem clear_tpm_owner_request=1 && reboot
```

You can also reflash the DUT with with the `--clobber-stateful` flag enabled:

```bash
cros flash myDUTHostName --clobber-stateful ${BOARD}/latest
```

### How to enable a local user account

This is not to be confused with how to [enable the chronos account]. This set of
instructions allows you to login to the browser as something other than guest
without having any network connectivity. **Most people don't need to do this**.

The local user account allows login with no password even if you can not connect
to the Internet. If you are customizing Chromium OS and having trouble logging
in due to your customizations, it may be handy to be able to bypass
authentication and log yourself in as a test user. This is disabled by default
for security reasons, but if you want to enable it for a backdoor user
`USERNAME`, enter the following from inside the chroot:

***note
**Note:** You will be prompted to enter a password. The `-d` flag preserves
existing user accounts on the device.

```bash
ssh myDUTHostName "/usr/local/autotest/bin/autologin.py -d -u '${USERNAME}'"
```
***

### How to avoid typing 'test0000' or any password on SSH-ing to your device

Modify your `/etc/hosts` and `~/.ssh/config` to make pinging and `ssh`-ing to
devices easier.

1.  Copy `testing_rsa` and `testing_rsa.pub` from chromite into your home folder
    for both inside and outside your chroot.
```bash
# Required for both inside and outside chroot
cp ~/chromiumos/chromite/ssh_keys/testing_rsa* ~/.ssh/
# ssh will ignore the private key if the permissions are too wide.
chmod 0600 ~/.ssh/testing_rsa
```
2.  Add the below to your `~/.ssh/config`:

```bash
Host dut
  HostName dut
  User root
  CheckHostIP no
  StrictHostKeyChecking no
  IdentityFile ~/.ssh/testing_rsa
  ControlMaster auto
  ControlPersist 3600
```

Include this for any lab/test machine:

```bash
Host 100.*
  User root
  IdentityFile ~/.ssh/testing_rsa
```

***note
**Note:** This assumes you have the following entry in your `/etc/hosts`:
***

```bash
100.107.2.189 dut # Replace with your DUT IP and chosen name for your DUT
```

All of the above enables you to do the below from inside and outside of your
chroot:

```bash
$ ping dut
$ ssh dut # No need to specify root@ or provide password
```

### How to get a fully functional vim on a test device

By default there is a very minimalist build of `vim` included on test images. To
replace it with a fully functional copy of `vim` do the following:

```bash
export BOARD=eve # Replace with the board name of your test device
export TESTHOST=127.0.0.1 # Replace with the hostname or ip address of the test device
export JOBS=$(nproc)

emerge-${BOARD} --noreplace --usepkg -j${JOBS} app-editors/vim-core app-vim/gentoo-syntax
USE=-minimal emerge-${BOARD} -j${JOBS} app-editors/vim

cros deploy "${TESTHOST}" app-vim/gentoo-syntax app-editors/vim-core app-editors/vim
```

## CrOS Chroot Tips

### How to modify my prompt to tell me which git branch I'm in

Add the following to your `~/.bash_profile`:

```branch
export PS1='\h:\W$(__git_ps1 "(%s)") \u\$ '
```

### How to search the code quickly

When you [installed depot\_tools], you got a tool called `git-gs` that's useful
for searching your code. According to the [depo_tools info page], `git-gs` is a
"Wrapper for git grep with relevant source types". That means it should be the
fastest way to search through your code.

You can use it to search the `git` project associated with the current directory
by doing:

```bash
git gs ${SEARCH_STRING}
```

If you want to search all git projects in the `repo` project associated with the
current directory:

```bash
repo forall -c git gs ${SEARCH_STRING}
```

On Linux, you can install [Silversearcher],
which is a faster and smarter version of `ack`/`grep`. `Ag` is very fast and
works independent of git repositories, so you can do queries from your top-level
folder.

```bash
sudo apt-get install silversearcher-ag
# For keyword based search
ag "keyword" .
# To ignore certain paths
ag --ignore chroot/ "keyword" .
# ag has the same syntax as grep. For example, excluding files that match a pattern
ag -L "keyword" .
```

### How to free up space after lots of builds

Use `cros clean` outside the chroot. This should take care of purging your
`.cache`, builds downloaded from devserver, etc.

### How to share files for inside and outside the chroot

The `cros_sdk` command supports mounting additional directories into your chroot
environment. This can be used to share editor configurations, a directory full
of recovery images, etc.

You can create a `src/scripts/.local_mounts` file listing all the directories
(outside of the chroot) that you'd like to access inside the chroot. For
example:

```bash
# source(path outside chroot) destination(path inside chroot)
/usr/share/vim/google
/home/${USER}/Downloads /Downloads
```

Each line of `.local_mounts` refers to a directory you'd like to mount, and
where you'd like it mounted. If there is only one path name, it will be used as
both the source and the destination directory. If there are two paths listed on
one line, the first is considered to be the path `OUTSIDE` the chroot and the
second will be the path `INSIDE` the chroot. The source directory must exist;
otherwise, `cros_sdk` will give off an ugly python error and fail to enter the
chroot.

*Note:* For security and safety reasons, all directories mounted via
`.local_mounts` will be read-only.

### How to symbolize a crash dump from a Chromium OS device

There is a nifty little script in your chroot, under `trunk/src/scripts` called
`cros_show_stacks`. The script takes in the parameters: board name, IP of the
remote device you want to fetch and symbolize crashes from, and a path to the
debug symbols. If the path is not specified, it picks it up from your local
build correctly (`/build/$BOARD/usr/lib/debug/breakpad`)

Typical usage. Note that `/Downloads` below refers to the Downloads folder
outside the chroot, the shared folder mapping is specified in
`~/chromiumos/src/scripts/.local_mount`.

```bash
./cros_show_stacks --board=eve --remote $DUT_IP --breakpad_root /Downloads/eve_debug_syms/debug/breakpad/
```

### How to make sudo a little more permissive

***note
**If you are at Google,** you will instead need to follow the instructions on the
[internal workstation notes] page for finishing the sudoers setup.
***

To set up the Chrome OS build environment, you should turn off the `tty_tickets`
option for sudo, because it is not compatible with  `cros_sdk`. The following
instructions show how:

```bash
cd /tmp
cat > ./sudo_editor <<EOF
#!/bin/sh
echo Defaults \!tty_tickets > \$1          # Entering your password in one shell affects all shells
echo Defaults timestamp_timeout=180 >> \$1 # Time between re-requesting your password, in minutes
EOF
chmod +x ./sudo_editor
sudo EDITOR=./sudo_editor visudo -f /etc/sudoers.d/relax_requirements
```

***note
**Note:** See the [sudoers man page] for full detail on the options available in
sudoers.
***

***note
**Note:** depending on your Linux distribution's configuration, `sudoers.d` may not
be the correct directory, you may check `/etc/sudoers` for an `#includedir`
directive to see what the actual directory is.
***

## Repo Tips

### How to set up repo bash completion on Ubuntu

Get a copy of [repo\_bash\_completion] and copy it to your home directory (e.g.,
under `~/etc`). Then, add the following line to your `~/.bashrc`:

```bash
[ -f "$HOME/etc/repo_bash_completion" ] && . "$HOME/etc/repo_bash_completion"
```

### How to configure repo to sync private repositories

Create a file called `.repo/local_manifests/my_projects.xml` and add your private project
into it.

```xml
<?xml version="1.0" encoding="UTF-8"?>`
<manifest>
  <remote  name   = "private"
           fetch  = "https://chromium.googlesource.com"
           review = "https://chromium-review.googlesource.com/" />
  <project path   = "src/thirdparty/location"
           name   = "nameofgitrepo"
           remote = "private" />
</manifest>
```

If you want to pull in from a different git server, you will need to add a
different remote for the project. Please type `repo manifest -o tmp.xml` to dump
your current manifest to see an example. More documentation on the manifest file
format is available on the [repo Manifest format docs].

### How to make repo sync less disruptive by running the stages separately

Just running `repo sync` does both the network part (fetching updates for each
repository) and the local part (attempting to merge those updates in to your
tree) in one go. This has a couple of drawbacks. Firstly, because both stages
are mixed up, you need to avoid making changes to your tree for the whole time,
including during the slow network part. Secondly, errors telling you which
repositories weren't updated because of your local changes get lost among the
many messages about the update fetching.

To alleviate these problems, you can run the two stages separately using the
`-n` (network) and `-l` (local) switches. First, run `repo sync -n`. This
performs the network stage, fetching updates for each of the repositories in
your tree, but not applying them. While it's running, you can continue working
as usual. Once it's finished and you've come to a convenient point in your work,
run `repo sync -l`. This will perform the local stage, actually checking out the
updated code in each repository where it doesn't conflict with your changes. It
generally only prints output when such a conflict occurs, which makes it much
easier to see which repositories you need to manually update. You can run it
multiple times until all the conflicts are fixed.

### How to find a list of branches that I've created with repo

See the `repo branches` command (i.e. `repo help branches`).

## Gerrit and Crbug Tips

Be sure to also take a look at the [git and gerrit intro].

### How to refer to bugs concisely

If you have a bug number and want to send it to someone else in an email, you
can use the *crbug* shortcut. So, to refer to bug 1234, you'd refer to:

<https://crbug.com/1234>

### How to refer to change-lists (CLs) concisely

When you want to send a short link for a chromium CL you can make short-links
such as:

<https://crrev.com/c/1234>

See <https://crrev.com> for instructions on linking to Chrome (internal) CLs
or making links based on the git hash.

### How to fetch patches/changes from a Gerrit CL with the Command Line

Very often, you may want to fetch changes from someone else's or your own CL
uploaded for code-review on [chromium-review.googlesource.com]. If you only want
to download and test a single change use the following repo command:

```bash
repo download -c 344778
```

If you want to download a specific patch set you can add a `/` followed by the
patch set number (e.g. `344778/2`). You can also list multiple CLs separated by
spaces.

***note
**Note:** this will clear other changes you may have already cherry-picked. If
you need to test multiple commits you either need to list them all or use the
git command below.
***

If you are more familiar with working with git, you can use the following. Make
sure to be cd-ed into the correct directory for that git repository. Replace
`cros` with whatever remote name is listed in your `.git/config` (e.g. `origin`,
`aosp`, etc.).

```bash
COMMIT_HASH=2a22b9dc94c9c13dbff0a1c397f49cb6456f4f2c
git fetch cros "${COMMIT_HASH}" && git cherry-pick FETCH_HEAD
```

This works because the CLs are stored on the same remote git server as where all
the branches and refs reside. You can switch to a branch that has its HEAD
pointing to a CL, based on just the CL id as follows.

Continuing the first example, to fetch <https://crrev.com/c/344778/> in a local
branch called *test-branch*:

```bash
repo download -b test-branch 344778
```


[Chromium OS Developer Guide]: developer_guide.md
[Chromium OS Developer FAQ]: https://dev.chromium.org/chromium-os/how-tos-and-troubleshooting/developer-faq
[repo\_bash\_completion]: https://chromium.googlesource.com/chromiumos/platform/dev-util/+/HEAD/host/repo_bash_completion
[repo Manifest format docs]: https://gerrit.googlesource.com/git-repo/+/HEAD/docs/manifest-format.md
[installed depot\_tools]: https://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools_tutorial.html#_setting_up
[depo_tools info page]: https://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools.html
[enable the chronos account]: developer_guide.md#Set-the-chronos-user-password
[internal workstation notes]: https://g3doc.corp.google.com/company/teams/chromeos/sites/resources/ubuntu-workstation-notes.md#configuring-etcsudoers
[sudoers man page]: https://www.sudo.ws/man/1.9.1/sudo.man.html
[chromium-review.googlesource.com]: https://chromium-review.googlesource.com/
[Silversearcher]: https://github.com/ggreer/the_silver_searcher
[git and gerrit intro]: developer_guide.md
