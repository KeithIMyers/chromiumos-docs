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
also mirrored in [chromefeatures.googleplex.com]. **As long as the feature has
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
implications of the feature, so that you are shipping something that doesn't
detract from the overall security posture of the product.

Even if you consider that the feature is trivial, or has no security
implications, **please refrain from flipping the security flag in the launch bug
yourself**. In most cases, features are not as trivial as they initially
appear. More importantly, the security team uses these flags to track features
and work on our side. We will flip the security flag when the feature is ready.

If the feature is big or complex, or if you find yourself implementing something
that needs to go against the recommendations in this document, please reach out
to the security team as soon as possible. Send email to
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

**The expectation is that the feature will be security-complete (e.g. the new
system service will be fully sandboxed) by the time the branch is cut**. Merging
CLs that implement security features to release branches is risky, so we avoid
it.

If the feature is enabled by default, the security flag in the launch bug
tracking the feature must be flipped before the milestone containing the feature
is promoted to the **beta channel**. This means that all the relevant
information (e.g. a design doc with a "Security considerations" section) must be
available before the milestone reaches the beta channel. Ideally, however, the
design doc will be finalized before the branch point.

If the feature is kept behind a flag, the security bit in the launch bug must
be flipped before the flag is enabled by default. This means that the feature
must be security-complete by the time the flag is enabled by default. Even in
this case we strongly recommend tackling security work earlier rather than
later. It's normally not possible to address security concerns in a feature
that's complete without requiring costly refactoring or rearchitecting.

In general, a feature that properly addresses the questions and recommendations
in this document can expect to have its security flag flipped by branch point.

## Life of a security review

Chrome OS security team members will use the following process when handling a
feature launch bug for the upcoming milestone:

1.  Look at the launch bug. Is there a design doc? If there's not, ask for one.
    If the design doc doesn't have a *Security Considerations* section, ask for
    one. Offer a link to the [review framework] as a guide for how to write that
    section.
2.  Use the [review framework] to evaluate if the feature is respecting security
    boundaries, handles sensitive data appropriately, etc. It's often also a
    good idea to consider existing features, in particular their security design
    and trade-off decisions made in previous reviews. Flip the launch flag to
    *Launch-Security-Started*.
3.  If any security-relevant aspects are unclear or if there are concerns,
    communicate this back to the feature owner via design doc comments and
    comment on the launch bug to clarify the security review status.
4.  Surface controversial design/implementation choices or aspects you're unsure
    about in the weekly Chrome OS security team meeting. Rely on the rest of the
    team to suggest useful alternative angles and to provide historic context
    and high-level guidance on Chrome OS security philosophy.
5.  If necessary, iterate with the feature owner to resolve any questions or
    concerns. Use whatever means of communication seems most appropriate to make
    progress. For simple questions, document comments or email threads will
    work. For in-depth discussion of product requirements, design choices, and
    implications on security assessment, it's usually better to ask for a
    meeting. Note that it is generally the responsibility of the feature owner
    to drive the review process to a conclusion. However, the security reviewer
    should strive for clear communication on what remains to be addressed at any
    point in the process.
6.  Once everything looks good, flip the review flag to *Launch-Security-Yes*.
    Document conclusions and aspects that were specifically evaluated in the
    security review in a bug comment. You can use the [review framework] to
    structure this. The information in the comment is useful for future
    reference when consulting previous security review decisions for guidance.
    Also, in case aspects of a feature are later found to cause security issues,
    it's useful to understand whether these aspects surfaced in the security
    review and the reasoning behind review conclusions. Note that the purpose is
    *not* to blame reviewers in case they have missed problems, but to help our
    future selves understand how we can improve the process as needed (for
    example by adding specific items to watch out for to the [review
    framework]).
    In case the review reaches an impasse, don't just mark *Launch-Security-No*
    as we're committed to engage productively as much as we can. Instead,
    surface the current state of things and bring in relevant leads to figure
    out a way forward.

## Review framework - things to look at

### Security boundaries

Does the feature poke holes in existing security boundaries? This is not a good
idea. Existing boundaries include:

*   Chrome renderer process to Chrome browser process: this boundary is
    traversed only with Chrome IPC or Mojo. It should not be trivial for a
    Chrome renderer process to directly manipulate or control the Chrome
    browser process. For browser-specific guidelines, check out
    [How To Do Chrome Security Reviews].
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
    the [sandboxing guide] for details) should be used to secure this boundary.
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

*   Implement the principle of least privilege: give code only the permissions
    it needs to do its job, and nothing else. E.g. if code doesn't need to be
    root, it shouldn't run as root. If it doesn't need to access the network, it
    shouldn't have access to the network.
*   See the [sandboxing guide] for more details.

### Untrusted input

*   Prefer robust parsers, e.g. protobuf. Avoid implementing your own
    serialization and deserialization routines: buggy (de)serialization code is
    routinely the source of security bugs. These bugs are dangerous because they
    can occur on the trusted side of an IPC boundary, allowing a less privileged
    process to possibly get code execution in a more privileged process. If you
    absolutely must write custom code, the requirement for new serialization
    code is for the code to be thoroughly fuzzed (see the
    [fuzzing documentation]) or written in a memory-safe language like Rust.
*   Don't reimplement IPC using pipes or sockets, use Mojo or D-Bus. The concern
    is the same as in the previous case: hand-written IPC is brittle, and bugs
    in IPC mechanisms could allow a less privileged process to subvert the
    execution of a more privileged process, which is the definition of a
    privilege escalation bug. Refer to the [Mojo IPC security guidelines] for
    more details.
*   Any code handling non-trivial untrusted data must be fuzzed to ensure its
    robustness. See our [fuzzing documentation] for more information.

### Sensitive data

*   Does the feature handle sensitive user data? Maintain Chrome OS's
    guarantee: user data is never exposed at rest (i.e. when the user is not
    logged in or when the device is off), and different users on the same device
    cannot access each other's data.
*   User data stored by the Chrome browser in the user's home directory is
    protected by Chrome OS's user data encryption. If the feature accesses this
    data from outside a Chrome session (e.g. not running as the `chronos`) user,
    how is it making sure that the data will not be saved to disk unencrypted?
*   Does the feature expose existing sensitive data to less privileged
    contexts? This is dangerous, and should be avoided. If necessary, consider
    filtering the data that is made available, allowing only reading the
    minimum amount of data required, disallowing modifications, and take care
    to not allow cross-user modification of user data.

### Attack surface

*   What can an attacker gain by triggering the code added for the feature? For
    example, new code might be calling a previously unused system call with
    parameters received directly from another process over IPC. If that other
    process is compromised, and the new code doesn't validate these parameters,
    the feature just added that system call as attack surface exposed to all
    processes that can access the IPC endpoint.
*   Are new libraries being used? If so, how confident are we about the quality
    of the code in the library? Do the upstream maintainers patch security bugs?
    Will the team adding the feature be responsible for updating the library
    when security bugs get discovered? Library code is subject to the same (or
    arguably more) scrutiny than code written by Chrome OS engineers: is it
    robust against malformed or malicious input? Can it be fuzzed?
*   Is the code accessible to remote attackers? Is it exposed directly over the
    network? If so, consider whether this is really necessary, and whether it
    can be mitigated with firewall rules or other restrictions.

### Implementation robustness

*   Does the code use threads? Are you confident there are no race conditions?
    Consider using the most straightforward threading or multi-process
    constructs you can (like `libbrillo`'s [BaseMessageLoop]).
*   Does the code implement a state machine or a protocol? Will an attacker
    sending packets in the wrong order break things? Write the code in a
    defensive way, to be robust in the face of malicious input and to *fail
    closed*, this is, to fail by denying access or by refusing to carry out a
    request, instead of the opposite.
*   Are there other assumptions prone to accidentally breaking in the future?
    This can include assuming an IPC is only ever called with certain
    parameters, or assuming a certain process is always present on the system,
    or that the state on disk for a file or a directory is always the same.
    If so, add a test to enforce the assumption, and revisit your code to be
    robust in the event of a broken expectation.

### Cryptography

*   If the feature is using cryptography for integrity, confidentiality, or
    authentication, reach out to the security team. You can always find us at
    [chromeos-security@google.com]. We can put you in contact with crypto
    experts to make sure that your use of cryptography is correct.
*   There should be no need for features to roll their own crypto. Use
    well-established cryptographic libraries like BoringSSL or OpenSSL.
*   Prefer to use high-level primitives. E.g. don't take an AES-CBC
    implementation and add your own padding, use a high-level primitive that
    simply encrypts and decrypts data provided a key.
*   If you're using keys, what is the key management story? Where are keys
    kept? Do they need to be hardware-protected?
*   Should key management tie in with our signing infrastructure? This is
    important if the keys are being used to sign code or other artifacts.
*   Do we know how to rotate keys in case there's a compromise? If the feature
    is using cryptographic keys extensively, you must write a key rotation
    doc as part of the feature.
*   For a quick primer on cryptography best practices, check out this
    [newer list of cryptographic right answers], as well as this
    [older reference of cryptographic right answers]; but remember to reach out
    to the security team to validate your design.

[sandboxing guide]: https://chromium.googlesource.com/chromiumos/docs/+/master/sandboxing.md
[crbug.com]: https://crbug.com
[chromefeatures.googleplex.com]: https://chromefeatures.googleplex.com
[chromeos-security@google.com]: mailto:chromeos-security@google.com
[fuzzing documentation]: https://chromium.googlesource.com/chromiumos/docs/+/master/fuzzing.md
[BaseMessageLoop]: https://chromium.googlesource.com/aosp/platform/external/libbrillo/+/master/brillo/message_loops/base_message_loop.h
[newer list of cryptographic right answers]: http://latacora.singles/2018/04/03/cryptographic-right-answers.html
[older reference of cryptographic right answers]: https://www.daemonology.net/blog/2009-06-11-cryptographic-right-answers.html
[review framework]: #review-framework-things-to-look-at
[How To Do Chrome Security Reviews]: https://docs.google.com/document/d/1JDC411NquvDGTQjQbtQSzHtBaQyCFMDTAAQZGifdmuE/edit
[Mojo IPC security guidelines]: https://chromium.googlesource.com/chromium/src/+/master/docs/security/mojo.md