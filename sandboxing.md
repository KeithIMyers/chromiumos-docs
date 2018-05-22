# Sandboxing Chrome OS system services

[TOC]

In Chrome OS, OS-level functionality (such as configuring network interfaces)
is implemented by a collection of system services, and provided to Chrome over
D-Bus. These system services have greater system and hardware access than
the Chrome browser.

Separating functionality like this prevents an attacker exploiting the Chrome
browser through a malicious website to be able to access OS-level functionality
directly. If Chrome were able to directly control network interfaces,
a compromise in Chrome would give the attacker almost full control
over the system. For example, by having a separate network manager, we can
reduce the functionality exposed to an attacker to just querying interfaces and
performing pre-determined actions on them.

Chrome OS uses a few different mechanisms to isolate system services from Chrome
and from each other. We use a helper program called Minijail (executable
`minijail0`). In most cases, Minijail is used in the service's init script.
In other cases, [Minijail wrappers] are used if a service wants to apply
restrictions to the programs that it launches, or to itself.

## Best practices for writing secure system services

Just remember that code has bugs, and these bugs can be used to take control
of the code. An attacker can then do anything the original code was allowed to
do. Therefore, code should only be given the absolute minimum level of
privilege needed to perform its function.

Aim to keep your code lean, and your privileges low. Don't run your service as
`root`. If you need to use third-party code that you didn't write, you should
definitely not run it as `root`.

Use the libraries provided by the system/SDK. In Chrome OS,
[libchrome](http://www.chromium.org/chromium-os/packages/libchrome) and
[libbrillo](http://www.chromium.org/chromium-os/packages/libchromeos)
(née libchromeos) offer a lot of functionality to avoid reinventing the wheel,
poorly. Don't reinvent IPC, use D-Bus. Don't open listening sockets, connect to
the required service.

Don't (ab)use shell scripts, shell script logic is harder to reason about and
[shell command-injection bugs](http://en.wikipedia.org/wiki/Code_injection#Shell_injection)
are easy to miss. If you need functionality separated from your main service,
use normal C++ binaries, not shell scripts. Moreover, when you execute them,
consider further restricting their privileges (see section [Minijail wrappers]).

# Just tell me what I need to do

* Create a new user for your service: https://crrev.com/c/225257
* Optionally, create a new group to control access to a resource and add the
  new user to that group: https://crrev.com/c/242551
* Use Minijail to run your service as the user (and group) created in the
  previous step. In your init script:
  * `exec minijail0 -u <user> /full/path/to/binary`
  * See section [User ids].
* If your service fails, you might need to grant it capabilities. See section
  [Capabilities].
* Use as many namespaces as possible. See section [Namespaces].
* Consider sandboxing your service using Seccomp-BPF, see section [Seccomp-BPF].

# User ids

The first sandboxing mechanism is user ids. We try to run each service as its
own user id, different from the `root` user, which allows us to restrict what
files and directories the service can access, and also removes a big chunk of
system functionality that's only available to the root user. Using the
permission_broker service as an example, here's its Upstart config file (lives
in `/etc/init`):

[`permission_broker.conf`](https://chromium.googlesource.com/chromiumos/platform2/+/master/permission_broker/permission_broker.conf)
```bash
env PERMISSION_BROKER_GRANT_GROUP=devbroker-access

start on starting system-services
stop on stopping system-services
respawn

# Run as 'devbroker' user.
exec minijail0 -u devbroker -c 0009 /usr/bin/permission_broker \
    --access_group=${PERMISSION_BROKER_GRANT_GROUP}
```

Minijail's `-u` argument forces the target program (in this case
permission_broker) to be executed as the devbroker user, instead of the root
user. This is equivalent of doing `sudo -u devbroker`.

The user (devbroker in this example) needs to be added to the system using the
user eclass, as in this CL (for a different user):
https://crrev.com/c/242551

There's a test in the CQ that keeps track of the users present on the system
that request additional access (e.g. listing more than one user in a group).
If your user does that, the test baseline has to be updated at the same time
the accounts are added with another CL (e.g. https://crrev.com/c/894192).
If you're unsure whether you need this, the PreCQ/CQ will reject your CL when
the test fails, so if the tests pass, you should be good to go!

You can use CQ-DEPEND to land the CLs together
(see [How do I specify the dependencies of a change?](http://www.chromium.org/developers/tree-sheriffs/sheriff-details-chromium-os/commit-queue-overview)).

# Capabilities

Some programs, however, require some of the system access usually granted only
to the root user. We use
[capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html)
for this. Capabilities allow us to grant
a specific subset of root's privileges to an otherwise unprivileged process.
The link above has the full list of capabilities that can be granted to a
process. Some of them are equivalent to root, so we avoid granting those. In
general, most processes need capabilities to configure network interfaces,
access raw sockets, or performing specific file operations. Capabilities are
passed to Minijail using the `-c` switch. permission_broker, for example, needs
capabilities to be able to chown() device nodes.

[`permission_broker.conf`](https://chromium.googlesource.com/chromiumos/platform2/+/master/permission_broker/permission_broker.conf)
```bash
env PERMISSION_BROKER_GRANT_GROUP=devbroker-access

start on starting system-services
stop on stopping system-services
respawn

# Run as <devbroker> user.
# Grant CAP_CHOWN and CAP_FOWNER.
exec minijail0 -u devbroker -c 0009 /usr/bin/permission_broker \
    --access_group=${PERMISSION_BROKER_GRANT_GROUP}
```

Capabilities are expressed using a capabilities mask, calculated from the
index of the capability in
[capability.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/capability.h),
and changed to a mask as in `CAP_TO_MASK`:

```c
#define CAP_TO_MASK(x)      (1 << ((x) & 31)) /* mask for indexed __u32 */
```

# Namespaces

Many resources in the Linux world can be isolated now such that a process has
its own view of things. For example, it has its own list of mount points, and
any changes it makes (unmounting, mounting more devices, etc...) are only
visible to it. This helps keep a broken process from messing up the settings
of other processes.

For more in-depth details, see the
[namespaces overview](http://man7.org/linux/man-pages/man7/namespaces.7.html).

In Chromium OS, we like to see every process/daemon run under as many unique
namespaces as possible. Many are easy to enable/rationalize about: if you don't
use a particular resource, then isolating it is straightforward. If you do
rely on it though, it can take more effort.

Here's a quick overview. Use the command line option if the description below
matches your service (or if you don't know what functionality it's talking
about -- most likely you aren't using it!).

*   `--profile=minimalistic-mountns`: This is a good first default that enables
    mount and process namespaces. This only mounts `/proc` and creates a few
    basic device nodes in `/dev`. If you need more things mounted, you can use
    the `-b` (bind-mount) and `-k` (regular mount) flags.
*   `--uts`: Just always turn this on. It makes changes to the host / domain
    name not affect the rest of the system.
*   `-e`: If your process doesn't need network access (including UNIX or netlink
    sockets)
*   `-l`: If your process doesn't use SysV shared memory or IPC

This option does not work on Linux 3.8 systems.  So only enable it if you know
your service will run on a newer kernel version.

* `-N`: If your process doesn't need to modify common
  [control groups settings](http://man7.org/linux/man-pages/man7/cgroups.7.html)

# Seccomp-BPF

Removing access to the filesystem and to root-only functionality is not enough
to completely isolate a system service. A service running as its own user id
and with no capabilities has access to a big chunk of the kernel API. The
kernel therefore exposes a huge attack surface to non-root processes, and we
would like to restrict what kernel functionality is available for sandboxed
processes.

The mechanism we use is called
[Seccomp-BPF](https://www.kernel.org/doc/Documentation/prctl/seccomp_filter.txt).
Minijail can take a policy file
that describes what syscalls will be allowed, what syscalls will be denied,
and what syscalls will only be allowed with specific arguments. The full
description of the policy file language can be found in the
[source](https://chromium.googlesource.com/aosp/platform/external/minijail/+/master/syscall_filter.c).

Abridged policy for
[mtpd on amd64 platforms](https://chromium.googlesource.com/chromiumos/platform2/+/master/mtpd/mtpd-seccomp-amd64.policy):

```
# Copyright (c) 2012 The Chromium OS Authors. All rights reserved.
# Use of this source code is governed by a BSD-style license that can be
# found in the LICENSE file.
read: 1
ioctl: 1
write: 1
timerfd_settime: 1
open: 1
poll: 1
close: 1
mmap: 1
mremap: 1
munmap: 1
mprotect: 1
lseek: 1
# Allow socket(domain==PF_LOCAL) or socket(domain==PF_NETLINK)
socket: arg0 == 0x1 || arg0 == 0x10
# Allow PR_SET_NAME from libchrome's base::PlatformThread::SetName()
prctl: arg0 == 0xf
```

Any syscall not explicitly mentioned, when called, results in the process
being killed. The policy file can also tell the kernel to fail the system call
(returning -1 and setting errno) without killing the process:

```
# execve: return EPERM
execve: return 1
```

To write a policy file, run the target program under strace and use that to
come up with the list of syscalls that need to be allowed during normal
execution. This script will take strace output files and generate a policy
file suitable for use with Minijail. On top of that, the `-L` option will print
the name of the first syscall to be blocked to syslog. The best way to proceed
is to combine both approaches: use strace and the script to generate a rough
policy, and then use `-L` if you notice your program is still crashing.

The policy file needs to be installed in the system, so we need to add it to
the ebuild file:

[`mtpd-9999.ebuild`](https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/master/chromeos-base/mtpd/mtpd-9999.ebuild)
```bash
# Install seccomp policy file.
insinto /opt/google/mtpd
newins "mtpd-seccomp-${ARCH}.policy" mtpd-seccomp.policy
```

And finally, the policy file has to be passed to Minijail, using the `-S`
option:

[`mtpd.conf`](https://chromium.googlesource.com/chromiumos/platform2/+/master/mtpd/mtpd.conf)
```bash
# use minijail (drop root, set no_new_privs, set seccomp filter)
exec minijail0 -u mtp -g mtp -G -n -S /opt/google/mtpd/mtpd-seccomp.policy -- \
        /opt/google/mtpd/mtpd -minloglevel="${MTPD_MINLOGLEVEL}"
```

## Detailed instructions for generating a seccomp policy

* Generate the syscall log:
  `strace -f -o strace.log <cmd>`
* Cut off everything before the following for a smaller filter that can be used with LD_PRELOAD:
```
futex(<uaddr>, FUTEX_WAKE_PRIVATE, 1) = 0
futex(<uaddr>, FUTEX_WAIT_BITSET_PRIVATE|FUTEX_CLOCK_REALTIME, 1, NULL, <uaddr2>) = -1
rt_sigaction(SIGRTMIN, {<sa_handler>, [], SA_RESTORER|SA_SIGINFO, <sa_restorer>}, NULL, 8) = 0
rt_sigaction(SIGRT_1, {<sa_handler>, [], SA_RESTORER|SA_RESTART|SA_SIGINFO, <sa_restorer>}, NULL, 8) = 0
rt_sigprocmask(SIG_UNBLOCK, [RTMIN RT_1], NULL, 8) = 0
getrlimit(RLIMIT_STACK, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
brk(0)                                  = 0x7f8a0656e000
brk(<addr>)                             = <addr>
```
* Run the policy generation script:
  `~/chromiumos/src/aosp/external/minijail/tools/generate_seccomp_policy.py strace.log > seccomp.policy`
* Test the policy:
  `minijail0 -S seccomp.policy -L <cmd>`
* To find a failing syscall without having seccomp logs available:
  `dmesg | grep "syscall="`
  for something similar to:
```
NOTICE kernel: [  586.706239] audit: type=1326 audit(1484586246.124:6): ... comm="<executable>" exe="/path/to/executable" sig=31 syscall=130 compat=0 ip=0x7f4f214881d6 code=0x0
```
* Then do:
  `minijail0 -H | grep <nr>`
  where `<nr>` is the `syscall=` number above.

# Minijail wrappers

TODO(jorgelo)

[Minijail wrappers]: #Minijail-wrappers
[User ids]: #User-ids
[Capabilities]: #Capabilities
[Namespaces]: #Namespaces
[Seccomp-BPF]: #Seccomp_BPF
