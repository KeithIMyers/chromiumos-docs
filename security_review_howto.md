# Chrome OS security review HOWTO

## Things to look at

### Security boundaries

Does the feature poke holes in existing security boundaries? This is not a good
idea. Existing boundaries include:

*   Chrome renderer process to Chrome browser process: this boundary is
    traversed only with Chrome IPC or Mojo. It should not be trivial for a
    Chrome renderer process to directly manipulate or control the Chrome
    browser process.
*   Chrome browser process (running as user `chronos`) to system services: this
    boundary is traversed with D-Bus. It should not be possible for the Chrome
    browser process to directly manipulate a system service or directly access
    system resources (e.g. firewall rules or USB devices).
*   ARC++ container to Chrome browser or Chrome OS system: this boundary is
    traversed with Mojo IPC. It should not be possible for the container to
    directly manipulate resources outside of the container. **Trust decisions
    should always be made outside of the container.**
*   Userspace processes to kernel: this boundary is traversed by system calls.
    It should not be possible for userspace to gain untrusted kernel code
    execution (this includes loading untrusted kernel modules). Seccomp (see
    the [sandboxing] guide for details) should be used to secure this boundary.
*   Kernel to firmware: it should not be possible for a kernel compromise to
    result in persistent, undetectable firwmare manipulation. This is enforced
    by Verified boot.

Are security boundaries still robust after the feature has been implemented?
For example, a feature might be in theory respecting a boundary by implementing
IPC, but if the IPC interface is designed so that the higher-privilege side
implicitly trusts and blindly carries out what the lower-privileged side
requests, then the security boundary has been weakened.

Does the feature require adding new boundaries? For example, ARC++ was a
feature that required adding a new security boundary: the ARC++ container. A
new security boundary, while sometimes necessary to restrict what untrusted
code can do, also adds a layer that will need to be enforced and maintained
going forward.

### Privileges

*   Enforce the principle of least privilege: give code only the permissions it
    needs to do its job, and nothing else.
*   Use [sandboxing] to enforce the principle of least privilege.

### Untrusted input

*   Prefer robust parsers, e.g. protobuf.
*   Don't reimplement IPC using pipes or sockets, use Mojo or D-Bus.
*   Any code handling non-trivial untrusted data must be fuzzed to ensure its
    robustness.

### Sensitive data

*   Does the feature handle sensitive user data? Credentials?
*   Does it expose existing sensitive data to less privileged contexts? This is
    dangerous.

### Attack surface

*   What can an attacker gain by triggering the code added for the new feature?
*   Are new libraries being used? If so, what about robustness / handling of
    untrusted input etc. in the new library code?
*   Is the code accessible to remote attackers?

### Implementation robustness

*   Does the code have race conditions? Assumptions that are likely to break?
    State machine or protocol sequencing bugs (i.e. does an attacker sending
    packets in the wrong order break things?)
*   Are these fragile assumptions prone to breaking accidentally in the future?
    If so, add a test to enforce the assumption.

### Cryptography

*   Is the feature using cryptography for integrity, confidentiality, etc.?
*   Is the key management story sane?
*   Should key management tie in with signing infrastructure?
*   Do we know how to rotate keys in case there's a compromise? If your feature
    is using cryptographic keys extensively, you must write a key rotation
    doc as part of the feature.

[sandboxing]: https://chromium.googlesource.com/chromiumos/docs/+/master/sandboxing.md
