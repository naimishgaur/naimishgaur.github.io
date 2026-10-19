---
layout: post
title: "The 404 Was Not the Clock"
date: 2026-10-19 12:00:00 +0530
categories: jekyll github-pages
---

The post URL 404'd after a push to GitHub Pages. Putting the frontmatter date in the past did not publish it.

## Problem

A Jekyll post about a USB-C hub, [Why a Powered USB-C Hub Made the Attached Keyboard Disappear](https://github.com/naimishgaur/naimishgaur.github.io/blob/master/_posts/2026-10-19-usb-c-hub-power-port-puts-mac-in-device-mode.md), was committed to `master` on `naimishgaur/naimishgaur.github.io`. GitHub Pages is configured as a legacy Jekyll build from `/` on `master`, theme `minima`, site timezone `Asia/Kolkata`.

The expected URL returned 404. So did the site root.

## The clock trap is real

Jekyll treats a dateless `date: YYYY-MM-DD` as midnight in the site timezone and drops posts dated in the future. With `timezone: Asia/Kolkata`, a post written in the evening whose frontmatter is only `2026-10-19` is midnight IST, which is in the past. A post written before midnight, or dated in UTC while the site clock is ahead, is silently omitted. The build still succeeds. The URL 404s. The homepage does not list it.

That is why the hub post's frontmatter was changed from a bare date to an explicit past timestamp:

```yaml
date: 2026-10-19 00:00:00 +0530
```

That commit (`34f9ece`) did not make the URL live. A future-dated post cannot explain a 404 on `/` for a repository whose first commit has no posts.

## What was actually failing

Checked at 22:06 IST, after the date fix had already been pushed:

| Check | Result |
|---|---|
| `https://naimishgaur.github.io/` | `404` |
| Hub post URL | `404` |
| `pages.status` | `errored` |
| `pages.build_type` | `legacy` |
| Builds of `6b0ed3e` and `b16bfbc` | `errored`, `"Page build failed."`, `duration: 0` |
| Build of `34f9ece` (the past-date fix) | `building`, `duration: 0`, `error: null` |
| `pages-build-deployment` Actions | queued ~13 minutes, then `in_progress` |

The repo has no `.github/workflows`. GitHub still injects `pages-build-deployment` and runs it on `ubuntu-latest`. The only check annotation on the failed commit was a notice, not a Jekyll error:

> The `ubuntu-latest` label will migrate to Ubuntu 26 beginning October 19, 2026.

That notice points at [actions/runner-images#14748](https://github.com/actions/runner-images/issues/14748). The issue says the label moves to Ubuntu 26.04 over several weeks starting 2026-10-19, finishing by 2026-11-19. It does not say the legacy Pages builder failed. The Pages API did not attach a build log either. `duration: 0` on `"Page build failed."` means the legacy builder rejected the site before a Jekyll log was published. The Actions queue is a second, separate delay.

## What to check before blaming the date

If the site root is 404, the post date is not the bug.

```bash
curl -sS -L -o /dev/null -w "%{http_code}\n" https://naimishgaur.github.io/

gh api repos/naimishgaur/naimishgaur.github.io/pages \
  --jq '{status, build_type}'

gh api repos/naimishgaur/naimishgaur.github.io/pages/builds \
  --jq '.[] | {status, error: .error.message, commit: .commit[0:7], duration}'

gh run list -R naimishgaur/naimishgaur.github.io --limit 5
```

A past frontmatter date is necessary. It is not sufficient. A green-looking push is not a deploy, and a queued `pages-build-deployment` job is not a Jekyll future-date drop.

## Source

- [Jekyll — Posts](https://jekyllrb.com/docs/posts/)
- [Jekyll — Configuration options](https://jekyllrb.com/docs/configuration/options/)
- [Troubleshooting Jekyll build errors for GitHub Pages sites](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/troubleshooting-jekyll-build-errors-for-github-pages-sites)
- [Creating a GitHub Pages site](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site)
- [REST API — Pages](https://docs.github.com/rest/pages/pages)
- [`ubuntu-latest` moving to Ubuntu 26.04](https://github.com/actions/runner-images/issues/14748)
- [Hub post source](https://github.com/naimishgaur/naimishgaur.github.io/blob/master/_posts/2026-10-19-usb-c-hub-power-port-puts-mac-in-device-mode.md)
