# Chromium OS docs

This directory contains public [Chromium OS] project documentation that is
automatically [rendered by Gitiles]. The docs are written in [Gitiles-flavored
Markdown].

## General guidelines

See the [Chromium documentation guidelines] and [Chromium documentation best
practices].

## Style guide

Markdown documents must follow the [style guide].

## Making changes

This repository is managed by the [repo] tool, so you can make changes to it
using the same techniques that you'd use for any other repositories in the
project. Feel free to bypass the commit queue and commit changes immediately
after they are reviewed.

## Previewing changes

You can preview your local changes using [md_browser]:

```bash
# at top of Chromium OS checkout
./chromium/src/tools/md_browser/md_browser.py -d docs
```

Then browse to e.g.
[http://localhost:8080/README.md](http://localhost:8080/README.md).

To review someone else's changes, apply them locally first, or just click the
`gitiles` link near the top of a Gerrit file diff page.

[Chromium OS]: https://www.chromium.org/chromium-os
[rendered by Gitiles]: https://chromium.googlesource.com/chromiumos/docs/+/master/
[Gitiles-flavored Markdown]: https://gerrit.googlesource.com/gitiles/+/master/Documentation/markdown.md
[Chromium documentation guidelines]: https://chromium.googlesource.com/chromium/src/+/master/docs/documentation_guidelines.md
[Chromium documentation best practices]: https://chromium.googlesource.com/chromium/src/+/master/docs/documentation_best_practices.md
[style guide]: https://github.com/google/styleguide/tree/gh-pages/docguide
[repo]: https://source.android.com/source/using-repo
[md_browser]: https://chromium.googlesource.com/chromium/src/tools/md_browser/+/master/
