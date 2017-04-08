# Chromium OS docs

This directory contains public [Chromium OS] project documentation that is
automatically [rendered by Gitiles]. The docs are written in [Gitiles-flavored
Markdown].

## General guidelines

See the [Chromium documentation guidelines] and [Chromium documentation best
practices].

## Style guide

Markdown documents must follow the [style guide].

## Previewing changes

You can preview your local changes using [md_browser]:

```bash
# at top of Chromium OS checkout
./chromium/src/tools/md_browser/md_browser.py -d docs
```

Then browse to e.g.
[http://localhost:8080/README.md](http://localhost:8080/README.md).

To review someone else's changes, apply them locally first.

[Chromium OS]: https://www.chromium.org/chromium-os
[rendered by Gitiles]: https://chromium.googlesource.com/chromiumos/docs/+/master/
[Gitiles-flavored Markdown]: https://gerrit.googlesource.com/gitiles/+/master/Documentation/markdown.md
[Chromium documentation guidelines]: https://chromium.googlesource.com/chromium/src/+/master/docs/documentation_guidelines.md
[Chromium documentation best practices]: https://chromium.googlesource.com/chromium/src/+/master/docs/documentation_best_practices.md
[style guide]: https://github.com/google/styleguide/tree/gh-pages/docguide
[md_browser]: https://chromium.googlesource.com/chromium/src/tools/md_browser/+/master/