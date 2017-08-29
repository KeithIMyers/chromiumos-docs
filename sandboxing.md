# Sandboxing Chrome OS system services

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
In other cases, Minijail wrappers are used if a service wants to apply
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
consider further restricting their privileges (see section Minijail wrappers).

# Just tell me what I need to do

* Create a new user for your service: https://chromium-review.googlesource.com/#/c/225257/
* Optionally, create a new group to control access to a resource and add the new user to that group: https://chromium-review.googlesource.com/#/c/242551/
* Use Minijail to run your service as the user (and group) created in the previous step. In your init script:
  * `exec minijail0 -u <user> /full/path/to/binary`
  * See section User ids.
* If your service fails, you might need to grant it capabilities. See section Capabilities.
* Consider sandboxing your service using Seccomp-BPF, see section Seccomp-BPF.

# User ids

The first sandboxing mechanism is user ids. We try to run each service as its
own user id, different from the `root` user, which allows us to restrict what
files and directories the service can access, and also removes a big chunk of
system functionality that's only available to the root user. Using the
permission_broker service as an example, here's its Upstart config file (lives
in /etc/init):

`permission_broker.conf`
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
https://chromium-review.googlesource.com/#/c/242551/

There's a test in the CQ that keeps track of the users present on the system,
so the test has to be updated at the same time the user is added, with another
CL (e.g. https://chromium-review.googlesource.com/#/c/242552/) that needs to
land at the same time. You can use CQ-DEPEND for this
(http://www.chromium.org/developers/tree-sheriffs/sheriff-details-chromium-os/commit-queue-overview,
"How do I specify the dependencies of a change?").

If you're only adding a new user or group, though, and not adding the new user
to any existing groups, the CQ will still pass.

# Capabilities

Some programs, however, require some of the system access usually granted only
to the root user. We use capabilities for this. Capabilities allow us to grant
a specific subset of root's privileges to an otherwise unprivileged process.
The link above has the full list of capabilities that can be granted to a
process. Some of them are equivalent to root, so we avoid granting those. In
general, most processes need capabilities to configure network interfaces,
access raw sockets, or performing specific file operations. Capabilities are
passed to Minijail using the `-c` switch. permission_broker, for example, needs
capabilities to be able to chown() device nodes.

`permission_broker.conf`
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
index of the capability in capability.h, and changed to a mask as in
CAP_TO_MASK:

```c
#define CAP_TO_MASK(x)      (1 << ((x) & 31)) /* mask for indexed __u32 */
```

# Seccomp-BPF

Removing access to the filesystem and to root-only functionality is not enough
to completely isolate a system service. A service running as its own user id
and with no capabilities has access to a big chunk of the kernel API. The
kernel therefore exposes a huge attack surface to non-root processes, and we
would like to restrict what kernel functionality is available for sandboxed
processes.

The mechanism we use is called Seccomp-BPF. Minijail can take a policy file
that describes what syscalls will be allowed, what syscalls will be denied,
and what syscalls will only be allowed with specific arguments. The full
description of the policy file language can be found in the source.

Abridged policy for mtpd on amd64 platforms:

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

`mtpd-9999.ebuild`
```bash
# Install seccomp policy file.
insinto /opt/google/mtpd
newins "mtpd-seccomp-${ARCH}.policy" mtpd-seccomp.policy
```

And finally, the policy file has to be passed to Minijail, using the `-S`
option:

`mtpd.conf`
```bash
# use minijail (drop root, set no_new_privs, set seccomp filter)
exec minijail0 -u mtp -g mtp -G -n -S /opt/google/mtpd/mtpd-seccomp.policy -- \
        /opt/google/mtpd/mtpd -minloglevel="${MTPD_MINLOGLEVEL}"
```

# Minijail wrappers

TODO(jorgelo)
