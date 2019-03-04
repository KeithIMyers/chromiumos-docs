# SELinux in Chrome OS

SELinux is a kernel security module that provides ability to write accessing
policies to archive mandatory access control.

In this documentation, it will briefly introduce
  - How SELinux play a role in Chrome OS;
  - How to write SELinux policy for Chrome OS;
  - How to troubleshooting SELinux in Chrome OS.

[TOC]

## Terms in This Documentation

This documentation contains many SELinux terms. It will be explained in this
section.

- `SELinux`: short for `Security-Enhanced Linux`. It provides MAC (Mandatory
  Access Control) to Linux. It defines (SELinux) a context for each object no
  matter if it's a process, normal file, directory, link, a socket, etc. When
  the actor (subject) requests to use a permission on the object, it checks the
  predefined policy to see if it's allowed or not. If the policy does not allow
  the action, then the calls that require this permission are denied. Automatic
  transition upon executing, or creation of a file is also possible.
- `security context`: Security context, also known as security label, is a
  string containing multiple parts, used to identify the process to look up
  security rules before granting or denying access to certain permissions.
  - `domain`: the `security context` for a process can be called as `domain`.
  - `parts`: The security context consists of 4 parts,
    `<user>:<role>:<type>:<range>`. For the syslog file `/var/log/messages`, it
    looks like `u:object_r:cros_syslog:s0`, and for a process like upstart, it
    looks like `u:r:cros_init:s0`.
    - `user`: identifies an SELinux user (not related to POSIX user). Chrome OS
      doesn't use multi-user. The only user is `u`.
    - `role`: identifies an SELinux role. Chrome OS doesn't use multi-role. The
      role for files is `object_r`, and for process (including `/proc/PID/*`)
      is `r`.
    - `type`: this is the most basic and important part in the security context.
      It's a string that independent from each other. For Chrome OS, it's the
      key part to identify the process or file for rules look-up.
    - `range`: range contains combinations of security classes and security
      levels. Security classes are independent from each other, while one
      security levels is dominated by another one. Chrome OS doesn't use
      multi-class security or multi-level security, but ARC container runnning
      Android program is using MCS and MLS.
- `attributes`: attribute is a named group of types.
- `rules`: rules defines whether an access request should be allowed, or logged,
  and how the security context will transit after the access.  Common rules to
  be used in Chrome OS (excluding ARC) SELinux policy are as follows. More
  details will be described in sections about how to write policies.
    - `allow`: `allow contextA contextB:class { permissions }`. Grants
      `contextA` `permissions` access to `class` (e.g. file, sock_file, port)
      under `contextB`. Either contextA or contextB can be changed to a group of
      contexts or `attributes` that can be assigned to multiple contexts.
    - `auditallow`: the same syntax as allow. It will log the access after it's
      being granted. auditallow-only will not grant the access.
    - `dontaudit`: the same syntax as allow. It will stop logging the given
      access after it's being denied.
    - `neverallow`: the same syntax as allow. It is not denying anything at
      runtime. It only performs compile-time check to see whether there's any
      conflict between `allow` and `neverallow`. Android has CTS to test the
      policy running active in the system break their neverallows or not.
    - `allowxperm`: `allowxperm contextA contextB:class permission args`. It
      check the args too at permission request. The arg is ususally not file
      paths but flags, since file paths should already be reflected in the
      context.  `allowxperm`-only will not grant the access. It must be used in
      combination of `allow`. Like `allow`, `allowxperm` also has similiar
      command in `auditallowxperm, `dontauditxperm`, and `neverallowxperm`.
    - `type_transition`: defines how type will auto change to a different one.
      Common type_transitions are:
      - `type_transition olddomain file_context:process newdomain`: when process
        in olddomain executes file_context, the process auto-transits to
        newdomain.
      - `type_transition domain contextA:{dir file ...} contextB [optional
        name]`: when process in domain, create a {dir file ...} optionally named
        as `name`, in `contextA`, the created file are auto-labelled as
        `contextB`. This is only true if the process didn't explicitly set the
        creation context. Without the type_transition, the created dir, file,
        and etc auto-inherits label from its parent.


For more details:

*   [Security Context](https://selinuxproject.org/page/NB_SC)
*   [Object Classes and Their Permissions](https://selinuxproject.org/page/ObjectClassesPerms)

## SELinux in Chrome OS boot process

SELinux has no presence until init (upstart) loads the policy, for example, in
the bootloader. This section will not discuss the stage before initially loading
the policy.

Please note: *SELinux is only enabled on ARC-enabled boards, or amd64-generic
based boards.*

1. init loads the selinux policy based on configs in /etc/selinuc/config, and
   mounts the selinuxfs to /sys/fs/selinux. init will be assigned with initial
   contexts (kernel).

1. init re-execs itself, to trigger type_transition `type_transition kernel
   chromeos_init_exec:process cros_init`, which will auto-transits init into
   cros_init domain.

1. init starting up service. init will startup services, and kernel will
   auto-transits based on defined type transition rules. See next section for
   details.

### init starting up services

init will start services based on the config files in /etc/init/ and their
dependencies. The "service" here not only include the daemon process service,
but also some pre-startup, or short-lived script.

#### Simple startup script embedded in init config file

Simple service startups are simply written in `<service-name>.conf` like

```
exec /sbin/minijail0 -l --uts -i -v -e -t -P /var/empty -T static \
    -b / -b /dev,,1 -b /proc \
    -k tmpfs,/run,tmpfs,0xe -b /run/systemd/journal,,1 \
    -k tmpfs,/var,tmpfs,0xe -b /var/log,,1 -b /var/lib/timezone \
    /usr/sbin/rsyslogd -n -f /etc/rsyslog.chromeos -i /tmp/rsyslogd.pid
```
in syslog.conf, or

```
script
  ARGS=""
  case ${WPA_DEBUG} in
    excessive) ARGS='-ddd';;
    msgdump)   ARGS='-dd';;
    debug)     ARGS='-d';;
    info)      ARGS='';;
    warning)   ARGS='-q';;
    error)     ARGS='-qq';;
  esac
  exec minijail0 -u wpa -g wpa -c 3000 -n -i -- \
    /usr/sbin/wpa_supplicant -u -s ${ARGS} -O/run/wpa_supplicant
end script
```
in wpa_supplicant.conf.

The earlier one, without a small script, kernel SELinux subsystem will
auto-transits the domain from `u:r:cros_init:s0`, `u:r:minijail:s0` upon
executing `/sbin/minijail0`, and then auto-transits to `u:r:cros_rsyslogd:s0`
upon executing `/usr/sbin/rsyslogd` so the AVC for domain `u:r:cros_rsyslogd:s0`
can be applied to the process. From now in the process, anything file access,
port usages, network usages, capability request, module load, and etc, will be
checked against AVC rules with scontext (source context) being
`u:r:cros_rsyslogd:s0`.

While the latter one, with a small script between `script` and `end script`, of
course, init forks a subprocess (still under cros_init domain) to execute shell,
upon executing shell, this will transits the subprocess to
`u:r:cros_init_scripts:s0` domain, some simple AVCs could be added to cover all
the simple embedded scripts together. Complex script or script needing
permissions more than file, or directory read, write, or creation, or exec,
should be avoided in the simple script, and should use a separate script. Within
the script, it will auto-transits to other domains upon executing the service
program, directly (for example, `exec /usr/sbin/rsyslogd` or indirectly (via
minijail0 the same as above).

#### Separate script to start the service

But there're some more complex service startup scripts, which are written in a
separate (shell) script. The init config file will look like `exec
/path/to/script.sh` or `exec /bin/sh /path/to/script.sh`

The earlier one is always preferred since it tells the kernel exact script being
executed, so automatic domain transition can be feasible upon executing the
script, to not to mix up permission requirements of the single script to the
whole init scripts.

The latter one should be **avoided** _if the script has complex permission
requirements_, like special capabilities, create device, modify sysfs, mount
filesystem, or load kernel modules. Using the latter approach will make the
kernel not able to distinguish which script to be executed, since the exec
syscall are always the same to exec `/bin/sh`, even the same with `script ...
end script` simple embedded script.

If using the earlier one in the init config file, it will auto transits to a
custom defined domain, let's say `u:r:cros_service_a_startup:s0`,  AVC rules
will be defined for this special domain, and finally a type_transition rule will
transit it from `u:r:cros_service_a_startup:s0`  to the domain owning the
service itself, like `u:r:cros_service_a:s0`. In the new domain, it will confine
the rules for the service.

#### Pre-start, pre-shutdown, and post-stop

pre-start, pre-shutdown, or post-stop scripts are usually simple embedded
scripts like

```
pre-start script
  mkdir -p -m 0750
  /run/wpa_supplicant
  chown wpa:wpa /run/wpa_supplicant
end script
```
in wpa_supplicant.conf.

This can be confined together with `u:r:cros_init_script:s0` since

 - It's simple enough, and doing almost similar things as all other simple
   pre-start, pre-shutdown, or post-stop domains;
 - Chrome OS has system image verification to make sure everything under
   /etc/init is the same as original state.

Like startup script, there're still very small number of services, using an
external script file for the startup. For example `pre-start exec
/usr/share/cros/init/shill-pre-start.sh` in shill.conf. This can be either
separate domains if they involves complex permissions like mounting/unmounting
filesystems, loading/unloading kernel modules, or special capabilities.

## How to write SELinux policy for Chrome OS

SELinux policy contains definitions of classes, access vectors (list of
permissions), security contexts (types, users, roles, and ranges), access vector
rules, and transition rules.

For most developers the necessary classes, access vectors, users, roles, and
ranges have already been defined. The most common workflow will be defining new
rules only.

### Defining Types

Types can be defined in the following syntax

```
type <type_name>[, <attribute1>, <attribute2>, ...];
```

This defines a type named `<type_name>`, and optionally add the atributes:
`<attribute1>`, `<attribute2>`, ...

Also, for an already defined type `<type_name>`, it can add an additional
attribute `<attributeN>` by using the following syntax:

```
typeattribute <type_name>, <attributeN>;
```

While attributes can be defined in

```
attribute <attribute>;
```

### File Contexts

#### System Image

File contexts for files in system image is defined in
`platform2/sepolicy/file_contexts/chromeos_file_contexts`.

Each line defines a path and its security context. For example,

```
/sbin/init u:object_r:chromeos_init_exec:s0
```

It defines the security context (label) for file `/sbin/init` to be
`u:object_r:chromeos_init_exec:s0`. Security context here must be complete
security context containing user, role, type, and range. Type-only will not
work. Chrome OS files always use `u` as user, `object_r` as role, and `s0` as
range for files in system image.

The path can also be an regular expression. For example

```
/usr/share/zoneinfo(/.*)? u:object_r:cros_tz_data_file:s0
```

Defined file labels are labelled during `build_image` phase. A simple `emerge`
invocation won't label files in the build directory. `cros deploy` doesn't label
it either.

#### Runtime Files

Runtime files consistents of persistent runtime files in stateful partition (for
example, /var/lib, /var/log), and volatile runtime files in tmpfs (for example,
/run).

##### At creation

Both runtime files needs to be created at the correct security context. Relabel
are prohibited in general, except for policy upgrades.

Creation label can be handled by either
[`setfscreatecon`(3)](https://manpages.ubuntu.com/manpages/bionic/en/man3/setfscreatecon.3.html)
from `libselinux`, or type transition rules.

Type transition is recommended since it doesn't require to modify the program to
be SELinux-aware. Developers should try their best to make sure files that need
type transition on creation, are created at a unique path to reduce the usage of
file name in the type transition rules. For example, only the daemon process
create the corresponding directory like `/var/lib/<service>`,
`/var/log/<service>`, etc, not startup script, nor some random tests.

```
filetrans_pattern(<domain>, <contextA>, <contextB>, file|dir|...);
filetrans_pattern(<domain>, <contextA>, <contextB>, file|dir|..., <file name>);
```

Above macros, provides ability when `<domain>` create a file|dir|... under
`<contextA>`, the created file|dir|... will be lablled as `<contextB>`. If the
`<file name>` is provided, only created file|dir|... with exact name will be
labelled as `<contextB>`.

For example,

```
filetrans_pattern(cros_rsyslogd, cros_var_log, cros_syslog, file, "messages");
```

will label "messages" created by process in `cros_rsyslogd` domain under
directory with `cros_var_log` type be labelled as `cros_syslog`. Thus, the
created structure will look like

```
/var/log => u:object_r:cros_var_log:s0
/var/log/messages => u:object_r:cros_syslog:s0
```

##### Persistent label for upgrades or recoveries

For files under persistent path, e.g. /var/lib, or /var/log, the fullpath based
file label **MUST** also be defined in `chromeos_file_contexts` together with
system image.

This is to make sure when policy upgrades, the new label can be restored upon
startup script without the need to recreate the files.

### Access Vector Rules

### Type Transition

Transition rules controls auto type transition upon creating of an object (file,
dir, sock_file, etc.), or executing an executables.

`type_transition` share the same syntax for both file type and process type,

```
type_transition `source_type` `target_type` : `class` new_type [object name];
```

#### File Type Transition

For file type transition, when processes running in `source_type` create a
`class` (file, dir, etc) under `target_type`, the created object is labeled as
`new_type` by default.

The same example as bove would be

```
type_transition cros_rsyslogd cros_var_log:file cros_syslog "messages";
```

`filetrans_pattern` macro wraps the type_transition rule and necessary AV rule
to one single macro to let creating object as a different type more easily.

#### Process Type Transition

For process transition, when a process running in `source_type` execute an
executable labelled as `target_type`, the process automatically transits to
`new_type`.

The example is

```
type_transition minijail cros_anomaly_collector_exec:process cros_anomaly_collector;
```

When a process under minijail execute a file labelled as
cros_anomaly_collector_exec, the after-exec process will be running under
cros_anomaly_collector domain.

There's also a useful macro like `filetrans_pattern` for process type transition
that wraps not only type_transition rule but also corresponding AV rules, called
`domain_auto_trans`. The detail will be explained the later sections.

Besides type_transition rule, there're also some other type rules that's less
frequently used, it can be referred from [SELinux Project
Wiki](https://selinuxproject.org/page/TypeRules)

### Useful Macros in Chrome OS

- filetrans_pattern

  - Syntax:

    ```
    filetrans_pattern(source_type, target_type, new_type, class);
    filetrans_pattern(source_type, target_type, new_type, class, object_name);
    ```

  - Explanation:

    When a process running in `source_type` creates an object in `class` (file,
    dir, etc), in directory labelled as `target_type`. If the object name
    matches `object_name`, the created object is automatically labelled as
    `new_type`.

    `source_type`, or `target_type` can be an attribute.

    `object_name` is optional. If `object_name` is not provided, all objects
    with the `class` created under `target_type` by `source_type` will be
    `new_type`, no matter what name the new object is.

    This macro will also grants necessary access for the creation, it will allow
    - `source_type` to create a `class` as `new_type`.
    - `source_type` to `add_name` in a directory labelled as `target_type`

  - Example:

    ```
    filetrans_pattern(cros_rsyslogd, cros_var_log, cros_syslog, file, "messages");
    filetrans_pattern(chromeos_startup, cros_var, cros_var_log, dir, "log");
    filetrans_pattern(cros_rsyslogd, tmpfs, cros_rsyslogd_tmp_file, file);
    filetrans_pattern({cros_session_manager cros_browser}, cros_run, arc_dir, dir, "chrome");
    filetrans_pattern(cros_browser, arc_dir, wayland_socket, sock_file, "wayland-0");
    ```

- domain_auto_trans

  - Syntax:

    ```
    domain_auto_trans(source_domain, exec_type, new_domain);
    ```

  - Explanation:

    When a process running in `source_domain`, executes a `file` labelled as
    `exec_type`, it becomes a process running in `new_domain`.

    The macro will also grants necessary for the execution, it will allow
    - `source_domain` to `execute` `exec_type`.
    - use `exec_type` as the entrypoint of `new_domain`.

  - Example:

    ```
    domain_auto_trans(minijail, cros_rsyslogd_exec, cros_rsyslogd);
    # source_domain chromeos_domain is an attribute.
    domain_auto_trans(chromeos_domain, minijail_exec, minijail);
    domain_auto_trans(cros_init, cros_unconfined_exec, chromeos);
    ```

### Naming Conventions

TODO

### Practice in Examples

TODO

## Troubleshooting

### How to rule out SELinux a possible cause of a problem

Let's assume you write your own program, or modified a program. Suppose it
doesn't work along with the system, you're sure you didn't implement anything
wrong, and suspect it could be SELinux denying some operations.

There're three approaches to identify an potential SELinux problem.

1. Currently, Chrome OS doesn't register any audit daemon, the `[kauditd]` in
   kernel will handle audits, and should output to syslog, as well as serial
   port.  You could dig into the audit log by grepping `avc:` or `audit:`. If
   there's a denial in non-permissive domain, there should be audit messages
   being logged, with `permissive=0`.  You could read the audit log to see if
   it's related to your program.

  - If you're on `betty` or other cros_vm instance, serial port output can be
    found at `/tmp/cros_vm_*/kvm.monitor.serial` in the cros_sdk chroot
    environment.
  - syslog will be logged to `/var/log/messages` and systemd-journald. You can
    read the log by reading `/var/log/messages` or by executing `journalctl`.
    Please note the messages file could be rotated to
    `/var/log/messages.{1,2,3,4,...}` if the system is running long term.

  You should be able to find `permissive=0` in above log locations. If you saw
  some denials with `permissive=1`, it doesn't mean it's denied.  `permissive=1`
  only mean this access it not allowed by policy, but SELinux is still allowing
  it because the domain is not enforced.

1. A quick command could test whether your program works by putting the whole
system permissive. By executing `setenforce 0` as root in developer mode, you
can put the whole system permissive. You'll be able to test if your program
comes to work.

1. If you program is a daemon process which fails so early before you can have a
console access, you could change the SELinux config file located at
`/etc/selinux/config` to
```
SELINUX=permissive
SELINUXTYPE=arc
```
Please keep
`SELINUXTYPE=arc` unchanged, and only changing `SELINUX=` line to `permissive`.
Please don't change it to `disabled` otherwise your system may fail to boot
since init will halt when it fails to load an SELinux policy.

Approach 2 and 3 to put the whole system permissive won't give you any useful
information on what's wrong. Audit log won't print even the policy says to deny
this because the whole system is permissive. It only gives you a true or false
answer, you'll need the audit logs to find out exact what the problem is. We'll
talk more at the how to debug section.

### How to read the denials in audit logs

An AVC audit message looks like

```
audit: type=1400 audit(1550558262.594:5434): avc:  denied  { read } for
pid=26768 comm="cat" name="messages" dev="dm-0" ino=40
scontext=u:r:cros_ssh_session:s0 tcontext=u:object_r:cros_syslog:s0 tclass=file
permissive=1
```

We'll walk through this audit log as an example.

- `type=1400` means it's an AVC audit log. So it's not what we care about. In
  most cases, you're looking at this kind of audit logs.
- `denied` means this permission usage is denied. The other result here could be
  `granted`, `granted` is only printed if the AVC rule has **auditallow** rules
  like `auditallow domainA domainB:file read`.
- `{ read }` means the permission requested is `read`, there could be many
  different kind of permissions, like `open`, `execute`, `append`, `name_bind`,
  `unlink`, etc. The permission inside `{}` could be more than one in one audit
  message, like `{ read open }`.
- `pid=26768` means the pid of the process.
- `comm="cat"` means the program command is cat. It's identical (but truncated)
  to /proc/PID/comm
- `name="messages"`, `dev="dm-0"`, `ino=40` are related information. Please read
  the link below in SELinux Project Wiki to know more.
- `scontext=u:r:cros_ssh_session:s0` means the source context for this
  permission request. If it's a process, it means the domain for the acting
  process.
- `tcontext=u:object_r:cros_syslog:s0` means the target context for this
  permission request. The tcontext could be file labels if it's a file (open,
  read, write, create, etc), process domain (use fd, netlink socket read/write,
  /proc/PID/ to read process attributes, unix socket sendto, etc), filesystem
  type (associate cros_var_lib on labeledfs, associate logger_device on
  devtmpfs, etc), and etc.
- `tclass=file` means the class type of the acting target. For example, in this
  case, it's `file`, it could be `udp_socket`, `fd`, `capability`,
  `netlink_kobject_uevent_socket`, etc.
- `permissive=1` means current audit is permissive or not. If it's permissive,
  it's not really getting denied, otherwise it really denies this permission
  request.

For more details, it can be referred from

[SELinux Project Wiki: NB AL](https://selinuxproject.org/page/NB_AL)

[Red Hat Documentation: Raw Audit
Messages](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/security-enhanced_linux/sect-security-enhanced_linux-fixing_problems-raw_audit_messages)

[Gentoo Wiki: Where to find SELinux permission denial
details](https://wiki.gentoo.org/wiki/SELinux/Tutorials/Where_to_find_SELinux_permission_denial_details)

[CentOS Wiki: Troubleshooting
SELinux](https://wiki.centos.org/HowTos/SELinux#head-02c04b0b030dd3c3d58bb7acbbcff033505dd3af)

### How to debug your SELinux policy

#### Analyzing audit logs

The most important and fundamental way to debug your policy is to read the audit
log.

In the previous audit log example, we know it's "`cat`" process in
`cros_ssh_session` domain was denied to read file "`messages`" in device `dm-0`
labelled as `cros_syslog`.

There're the main things to look at in the audit logs.

- Actor
  - Which process, pid / name;
  - The context of the actor (scontext);

- Actee / Target
  - Which target?
  - Which class. process, file, sock_file, port, etc?

By looking at the log, the main thinkabout would be"

- Is the actor and target at the correct context? No? => Fix the context.
  - File label:
    - In system image: add the context to
      `platform2/sepolicy/file_contexts/chromeos_file_contexts`, and
      `platform2/sepolicy/sepolicy/chromeos/file.te`.
    - In stateful partition:
      - (Recommended and Easiest) Introduce a new file type in
        `.../sepolicy/chromeos/file.te`, and use `filetrans_pattern` macro to
        allow auto assigning labels upon file creation.
      - Introduce a new file type in `.../sepolicy/chromeos/file.te`, and modify
        the program to set correct creation label.
        [setfscreatecon(3)](https://manpages.ubuntu.com/manpages/bionic/en/man3/setfscreatecon.3.html)
  - Process domain: fix the executable file label, and use `domain_auto_trans`
    macros if possible.
  - Port context, etc: you're probably already an advanced SELinux policy writer
    if you met this point. You can refer to [SELinux Project
    Wiki](https://selinuxproject.org/), for example
    [portcon](https://selinuxproject.org/page/NetworkStatements#portcon)
- Is the action really needed? No? => Fix the program to eliminate the action.

  Examples of mostly unneeded actions:
  - relabelfrom/relabelto: only some startup script should need this.
  - capability dac_override: in most cases, it could be avoided by reordering
    chown / actual read-write.
  - mount/mounton: except for some startup script, or programs using libminijail.
    This should be avoided. For programs using minijail0 wrapper, `-T static`
    mode is strongly recommended to leave all the high privileged permissions to
    minijail.

#### Inspecting the runtime state

- File labels: `ls -Z file` or `ls -Zd directory`
- Process domains: `ps -Z`, `ps auxZ`, `ps -Zp <PID>`
- Current domain: `id -Z`
- Run in a different domain: `runcon <context> <program> <args...>` for example,
  `runcon u:r:cros_init:s0 id -Z`. The transition from current domain to new
  domain must be allowed to let this work. Currently, either `cros_ssh_session`
  or `frecon` is running permissive, it should always work if you're executing
  in the console, or via ssh.

#### Update the policy

After understanding why the denials occurred by reading the log, policy may need
updating to fix the problem.

##### Locate the policy

In general, Chrome OS policy lives in `sepolicy` directory in
`chromiumos/platform2` project, which is `src/platform2/sepolicy` in the repo
tree checkout.

A quick grep on the scontext will locate the where it is defined, and most of
its policies.

For example, if we want to change `cros_ssh_session`:

```
$ grep cros_ssh_session . -R
./policy/chromeos/dev/cros_ssh_session.te:type cros_ssh_session, domain, chromeos_domain;
./policy/chromeos/dev/cros_ssh_session.te:permissive cros_ssh_session;
./policy/chromeos/dev/cros_ssh_session.te:typeattribute cros_ssh_session netdomain;
./policy/chromeos/dev/cros_sshd.te:domain_auto_trans(cros_sshd, sh_exec, cros_ssh_session);
./policy/chromeos/file.te:filetrans_pattern(cros_ssh_session, cros_etc, cros_ld_conf_cache, file, "ld.so.cache~");
./policy/chromeos/log-and-errors/cros_crash.te:-cros_ssh_session
```

We can see it's defined in `cros_ssh_session.te`, which means most of our
changes should live in that file.

##### Searching the compiled policy file

`sesearch` is an excellent tool to search inside a compiled policy. You can use
this tool to search what is allowed, what denials are not logged, what grants
are logged, and type transitions, etc.

on Debian-based systems (or gLinux), `sudo apt install setools` will install
this tool.

You can search a policy file, say, `policy.30`, in following examples:

```
# Search all allow rule with scontext to be cros_ssh_session or attributes
cros_ssh_session attributes to, tcontext to be cros_sshd or attributes cros_sshd
attributes to with class to be process
$ sesearch policy.30 -A -s cros_ssh_session -t cros_sshd -c process
# Search all type transitions with scontext to be exactly minijail
$ sesearch policy.30 -T -s minijail -ds
```

`man sesearch` will provide all the options to search `allow`, `auditallow`,
`dontaudit`, `allowxperm`, etc, with filters on scontext, tcontext, class,
permissions.

##### Put domain to permissive

Sometimes, during debugging, you may not to want to put the system permissive.
You can put only one domain permissive.

1. Locate the policy file as above.

1. Simply add `permissive <domain type>`, for example, `permissive
cros_ssh_session` will put `cros_ssh_session` to permissive.

This will only put the given domain to permissive, and everything with the
permissive actor domain (scontext) will not actually being denied.

But please note, some operation may indicate other permission at runtime. For
example, file creation will check
`{ associate } scontext=file_type tcontext=fs_type class=filesystem `, these kind of
denials may occur. If you saw similiar denials please reach kroot@ or fqj@,
we'll fix it.

##### Writing policy fix

1. Identify whether labeling files is needed. If yes, label the files either in
   file_contexts or via type_transition.

1. Fix the program or add `dontaudit` rule to prevent from spamming logs if it
   shouldn't be allowed.

1. Write allow rule or allowxperm rule based on denials seen, and the behavior
   analysis of the program.

   1. for one-time program-specific permission requests, simply `allow[xperm]
      scontext tcontext:class perms [args for allowxperm];`
      `scontext`, `tcontext`, and `perms` can be plural in format like `{ a b c
      }`

   1. for permission requests that may apply to other programs, create an
      attribute and attribute current domain to it. And write corresponding
      rules for that attribute.

   1. use m4 macros wisely, we have many useful macros like `r_file_perms`,
      `rw_file_perms`, `create_file_perms`, `filetrans_pattern`, and
      `domain_auto_trans`.

For more details in writing policy, please refer to previous sections about
writing policies.

#### Useful build flags for debug

1. `USE="selinux_develop"`: log permissive denials and make sure log is almost
   not suppressed by printk limit.

1. `USE="selinux_experimental"`: build with SELinux mode in permissive by
   default. This is equivalant to manually changing `SELINUX=permissive` in
   `/etc/selinux/config`

1. `USE="selinux_audit_all"`: remove all the `dontaudit` rule before compiling
   to the final monolithic policy. There're some should-be-denied access with
   `dontaudit` rules, so denials don't spam the log. But you may want to see
   them sometimes during development or debugging process.

For Googlers, there's a nice introduction presentation slides how debugging
SELinux policies to refer to though it's for Android, at
[go/sepolicy-debug](https://goto.google.com/sepolicy-debug)
