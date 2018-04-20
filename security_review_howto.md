# Chrome OS security review HOWTO

## The security review process

Chrome OS development is structured around six-week cycles called *milestones*.
Every six weeks a new release branch is created, based off our main development
branch (also known as *trunk* or *tip-of-tree*). Accordingly, a new milestone is
pushed to Chrome OS devices every six weeks. It takes seven to eight weeks from
the time a branch is cut to the time a new software image built from that
branch is pushed to devices on the stable channel.

A feature targeting a given milestone will be reviewed during that milestone's
development cycle, or shortly after the branch is cut. The Chrome OS security
team tracks features by looking at *Launch bugs* filed in [crbug.com], which are
also mirrored in [chromefeatures.googleplex.com]. **As long as your feature has
an associated launch bug, the security team will track it**. Make sure that the
launch bug links to a design doc that includes a section covering the security
implications of the feature. **The rest of this document describes what
questions such a "Security implications" section should answer and what
concerns it should address**.

Launch bugs include a set of cross-functional review flags, one of which is the
security flag. The security team will flip this flag after the feature owner
has successfully engaged the security team to understand the security
implications of the feature. **Don't think of the security review process as an
arbitrary bar set by the security team that you have to pass no matter what**.
Instead, think of it as the process by which you take ownership of the security
implications of your feature, so that you are shipping something that doesn't
detract from the overall security posture of the product.

Even if you consider that your feature is trivial, or has no security
implications, **please refrain from flipping the security flag in the launch bug
yourself**. In most cases, features are not as trivial as they initially
appear. More importantly, the security team uses these flags to track features
and work on our side. We will flip the security flag when the feature is ready.

If your feature is big or complex, or if you find yourself implementing
something that needs to go against the recommendations in this document, please
reach out to the security team as soon as possible. Send email to
[chromeos-security@google.com], and try to include a design doc, even if it's
just an early draft. **When in doubt, just reach out**. We are always happy to
discuss feature design.

The Chrome OS security team will normally not look at the implementation details
of a feature -- there is just not enough time to read through thousands of lines
of code each milestone. Instead, we prefer to focus on ensuring that the design
of the feature is contributing to, rather than detracting from, the overall
security posture of the system. The reason for this is two-fold: first, the time
constraints mentioned before. Second is the fact that even with careful review
bugs will likely slip through, and a sound, defensive design will ensure that
these bugs don't end up being catastrophic. For particularly risky code we can
always contract out a security audit.

**The expectation is that your feature will be security-complete (e.g. your new
system service will be fully sandboxed) by the time the branch is cut**. Merging
CLs that implement security features to release branches is risky, so we avoid
it.

If your feature is enabled by default, the security flag in the launch bug
tracking the feature must be flipped before the milestone containing the feature
is promoted to the **beta channel**. This means that all the relevant
information (e.g. a design doc with a "Security considerations" section) must be
available before the milestone reaches the beta channel. Ideally, however, the
design doc will be finalized before the branch point.

If your feature is kept behind a flag, the security bit in the launch bug must
be flipped before the flag is enabled by default. This means that the feature
must be security-complete by the time the flag is enabled by default. Even in
this case we strongly recommend tackling security work earlier rather than
later. It's normally not possible to address security concerns in a feature
that's complete without requiring costly refactoring or rearchitecting.

In general, a feature that properly addresses the questions and recommendations
in this document can expect to have its security flag flipped by branch point.

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
[crbug.com]: https://crbug.com
[chromefeatures.googleplex.com]: https://chromefeatures.googleplex.com
[chromeos-security@google.com]: mailto:chromeos-security@google.com
