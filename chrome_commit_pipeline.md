# Life of a Chrome commit on Chrome OS

This document provides a brief overview of how Chrome changes get committed
and tested on Chrome OS.

For details on non-Chrome **Chrome OS** changes, see
[Life of a Chrome OS commit](cros_commit_pipeline.md).

## Create a Chrome change

### Make and upload changes

See the section on [contributing code] at [chromium.org]
for how to create a branch and make changes.

Once a change is completed and tested locally, upload it to [codereview]:

```git cl upload```

### Have your change reviewed

Use [codereview] to review your change and prepare it for the commit queue.
See [contributing code] for details.

## The Chrome commit pipeline

### The Commit Queue and Tryservers

The Chrome [commit queue] has a very large pool of builders that will apply
*individual* changes to the master, build them, and test them.

#### Before the patch is approved

A developer can click on 'Choose trybots' to select specific builders to run
(there are a lot of them).

Alternately they can click 'CQ dry run' to run all of the builders that the
CQ will run in advance, without scheduling a commit.

#### The commit queue

Once a change has been reviewed and approved, the developer can check
the 'Commit' checkbox. This will mark the change as ready for the CQ.

Depending on what was changed, the CQ selects a suite of tryserver builders for
[win], [mac], [linux], and [android].

Note: The [linux] builders include **linux-chromeos** builders (linux builders
with `chromeos=1`). These run `browser_tests` and `unit_tests` for Chrome on
Chrome OS.

If the CQ builders succeed then the change will be committed to the master.

Otherwise the 'Commit' checkbox will need to be re-checked once the failure
is fixed or determined to be unrelated to the change.

### The chromium waterfall

Once a change is committed on the master, it is picked up by the
[chromium waterfall]. This includes a very large number of builders that will
thoroughly test the commit, including a number of [linux-chromeos builders].

*Note: Due to lab limitations not every builder in the waterfall is included
in the Commit Queue. For Chrome OS there are a few Debug test builders that only
exist on the waterfall. Failures there are infrequent but possible, so keep an
eye out!*

## The Chrome OS commit pipeline for Chrome changes

Once a Chrome change lands in the master, it needs to get through the [PFQ]
before it will be picked up by Chrome OS. This is to protect the Chrome OS
builders, since Chrome OS depends heavily on Chrome itself.

### The PFQ Informational builders

There are two sets of continuous PFQ informational builders that apply the
most recent Chrome changes to the most recent Chrome OS build and run several
test suites:

* [chromiumos.chromium] is public and tests the Chromium builds.
    * It also includes asan and telemetry builders.
* [chromeos.chrome] is internal and tests the official Chrome builds.
    * It also tests 'cros chrome-sdk' (Simple Chrome) with official builds.

### The PFQ builders

The [PFQ] builds a daily Chrome version and the most recent Chrome OS
version (using the manifest from the latest CQ/paladin build). It then builds a
Chrome OS image and runs several test suites against the result.

* Once daily (currently at 8 PM PT) a Chrome release branch is created and
  tagged.
    * Sometimes additional tags are created for the release branch, e.g. after
      a revert or fix for a bad change is merged to the branch.
* Whenever such a branch tag is created, a new master [PFQ] build is triggered.
* The master triggers a series of Chrome and Chromium slave builders covering
  all cpu types (arm, amd64, and x86) and important variants. Coverage is
  ensured by [chromite/cbuildbot/binhost_test].
* PFQ Builders do the following:
    * Check out the Chrome OS source from the LKGM manifest.
    * Check out the Chrome source from the tagged Chrome release branch.
    * Build Chrome OS, including Chrome from the local source checkout.
    * Run VM tests on all non ARM builders.
    * Run HW tests on builders where HW is available, including ARC tests on
      boards that support it.
    * Generate a Simple Chrome environment for developers.
        * This generates a tarball that includes necessary dependencies from
          the Chrome OS version used to build Chrome.
        * These tarballs will be used by Simple Chrome once the [Chrome LKGM]
          builder identifies a stable Chrome OS version from recent canary
          builds.
    * Test the Simple Chrome environment by building Chrome (in parallel with
      HW tests).


[contributing code]: https://www.chromium.org/developers/contributing-code
[commit queue]: https://dev.chromium.org/developers/testing/commit-queue
[chromium.org]: http://www.chromium.org
[codereview]: https://codereview.chromium.org/
[win]: https://build.chromium.org/p/tryserver.chromium.win
[mac]: https://build.chromium.org/p/tryserver.chromium.mac
[linux]: https://build.chromium.org/p/tryserver.chromium.linux
[android]: https://build.chromium.org/p/tryserver.chromium.android/waterfall
[chromium waterfall]: https://build.chromium.org/p/chromium/waterfall
[linux-chromeos builders]: https://build.chromium.org/p/chromium.chromiumos/waterfall
[PFQ]: https://uberchromegw.corp.google.com/i/chromeos/builders/master-chromium-pfq
[chromiumos.chromium]: https://build.chromium.org/p/chromiumos.chromium/waterfall
[chromeos.chrome]: https://uberchromegw.corp.google.com/i/chromeos.chrome/waterfall
[chromite/cbuildbot/binhost_test]: https://cs.corp.google.com/chromeos_public/chromite/cbuildbot/binhost_test.py
[Chrome LKGM]: https://yaqs.googleplex.com/eng/q/5254238507106304
