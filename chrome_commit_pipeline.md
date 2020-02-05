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

Once a Chrome change lands in the master, it needs to be landed via [PUpr]
uprev before it will be picked up by Chrome OS. PUpr watches chrome tags
as they are created each day and cuts a CL to the current gardener to land.
Changes are included in ChromeOS builds once the uprev CL makes it through
the ChromeOS CQ. There is a [PUpr Dashboard] that shows the current status
of the uprev CLs as well as the age of the current chrome version on ChromeOS.

[contributing code]: contributing.md
[commit queue]: https://chromium.googlesource.com/chromium/src/+/master/docs/infra/cq.md
[chromium.org]: http://www.chromium.org
[codereview]: https://codereview.chromium.org/
[win]: https://build.chromium.org/p/tryserver.chromium.win
[mac]: https://build.chromium.org/p/tryserver.chromium.mac
[linux]: https://build.chromium.org/p/tryserver.chromium.linux
[android]: https://build.chromium.org/p/tryserver.chromium.android/waterfall
[chromium waterfall]: https://build.chromium.org/p/chromium/waterfall
[linux-chromeos builders]: https://build.chromium.org/p/chromium.chromiumos/waterfall
[chrome_informational]: http://go/legoland/builderSummary?buildBranch=master&builderGroups=chrome_informational
[chromite/cbuildbot/binhost_test]: https://cs.corp.google.com/chromeos_public/chromite/cbuildbot/binhost_test.py
[Chrome LKGM]: https://yaqs.googleplex.com/eng/q/5254238507106304
[PUpr]: https://g3doc.corp.google.com/company/teams/chrome/ops/chromeos/continuous_integration/pupr.md?cl=head
[PUpr Dashboard]: http://go/chromeos-dash-pupr-chrome
