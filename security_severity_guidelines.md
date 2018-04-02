# Chrome OS Security Severity Guidelines

These are the severity guidelines for Chrome OS Security
Issues. They are related to to the [Severity Guidelines for Chrome Security
Issues]. One key difference between the Chrome and  Chrome OS security model is
that Chrome OS needs to protect against physically local attackers in
certain cases, such as at the lockscreen.

[TOC]

## Critical Severity

Critical severity issues breach vital security boundaries. The following
boundaries are considered critical:

*   Verified boot (any attack that persists across reboot)
*   User isolation. One user can exploit another user or access
    the encrypted data of another user (i.e. [crbug.com/764540])
*   Native code execution via a remote vector
*   Kernel code execution via a remote vector
*   A lock screen bypass from the lockscreen

They are normally assigned priority **Pri-0** and assigned to the current
stable milestone (or earliest milestone affected). For critical severity
bugs, [SheriffBot] will automatically assign the milestone.

> For critical vulnerabilities, we aim to deploy the patch to all
> Chrome OS users in under 30 days.

Critical vulnerability details may be made public in 60
days, in accordance with Google's general
[vulnerability disclosure recommendations], or [faster (7 days)],
if there is evidence of active exploitation.

A typical type of critical vulnerability on Chrome OS is
an exploit chain comprising individual bugs that allows
persistent root access on a machine, even on guest mode ([766253]).

Note that the individual bugs that make up the chain will have lower
severity ratings.

## High Severity

A high severity bug allows code execution in a sandboxed process. Bugs
which would normally be critical severity with mitigating factors may be
rated as high severity. Mitigating factors include:

*   A specific user interaction is required to trigger the exploit.
    For example, a user must be tricked into typing something into an address
    bar, or the vulnerability requires a user to install an extension or
    Android APK.
*   A small fraction of Chrome OS devices are affected due to hardware
    or kernel specific issues.

The above might be generalized to: bugs that allow bypassing of protection
domains are rated as High severity. A Chrome sandbox escapes allows
bypassing the Chrome sandbox. A bug that allows code execution as the
chronos user would also be rated High severity. The individual bug that
allows kernel code execution from root (or a regular user) would be rated
High severity. The bug that allows for exploit persistence given root
access would be rated High severity as well.

In general, these are the layers of protection whose bypasses we consider
High-severity bugs:

*   Native code execution in a renderer process
*   Browser process/`chronos` user code execution
*   Root (or other more privileged user, such as system service users)
    code execution
*   Code execution in the kernel that requires local priveleges to exploit
*   Persistent code execution

Full chain exploits don't always need to break all these layers. For example,
most persistent exploitation doesn't need a kernel exploit.

They are normally assigned priority **Pri-1** and assigned to the current
stable milestone (or earliest milestone affected). For high severity bugs,
[SheriffBot] will automatically assign the milestone.

For high severity vulnerabilities, we aim to deploy the patch to all Chrome
users in under 60 days.

## Medium Severity

Medium severity bugs allow attackers to read or modify limited amounts of
information, or are not harmful on their own but potentially harmful when
combined with other bugs. This includes information leaks that could be
useful in potential memory corruption exploits, or exposure of sensitive
user information that an attacker can exfiltrate. Bugs that would normally
rated at a higher severity level with unusual mitigating factors may be
rated as medium severity.

Examples of medium severity bugs include:

*   Memory corruption in a system service that's not directly
    triggerable from an exposed interface
*   An out of bounds read in a system service

They are normally assigned priority **Pri-1** and assigned to the current
stable milestone (or earliest milestone affected). If the fix seems too
complicated to merge to the current stable milestone, they may be assigned
to the next stable milestone.

## Low Severity

Low severity vulnerabilities are usually bugs that would normally be a higher
severity, but which have extreme mitigating factors or highly limited scope.

Example bugs:

*   Someone with local access to the machine could disable security
    settings without authenticating (i.e. disable the lockscreen).

They are normally assigned priority **Pri-2**. Milestones can be assigned
to low severity bugs on a case-by-case basis, but they are not normally
merged to stable or beta branches.

[Severity Guidelines for Chrome Security Issues]: https://chromium.googlesource.com/chromium/src/+/master/docs/security/severity-guidelines.md
[crbug.com/764540]: https://bugs.chromium.org/p/chromium/issues/detail?id=764540
[SheriffBot]: https://www.chromium.org/issue-tracking/autotriage
[vulnerability disclosure recommendations]: https://security.googleblog.com/2010/07/rebooting-responsible-disclosure-focus.html
[faster (7 days)]: https://security.googleblog.com/2013/05/disclosure-timeline-for-vulnerabilities.html
[766253]: https://bugs.chromium.org/p/chromium/issues/detail?id=766253
