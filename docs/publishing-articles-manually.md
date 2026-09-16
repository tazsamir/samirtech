# Drafting and publishing articles

The canonical site is https://samirtech.org/. Cloudflare Pages builds Hugo source through its Git integration. `.github/workflows/cloudflare-pages.yml` validates the source; it does not deploy the tracked `public/` directory.

## Check the repository first

```bash
cd ~/Projects/samirtech
git status --short --branch
git branch --show-current
git remote -v
```

Resolve unrelated changes before editing. Do not pull `main` into an arbitrary topic branch. Fetch and review branch ancestry before merging or pushing; do not assume the checked-out branch is the production branch.

## Create an unpublished draft

Create `content/posts/my-new-article.md` with YAML front matter:

```yaml
---
title: "My New Article"
date: 2026-09-16T09:00:00+01:00
draft: true
description: "A short, accurate description."
tags: [homelab]
---
```

Use the actual current date (`date --iso-8601=seconds`), not the example date. Preserve an existing article's URL when updating it. Add series metadata only when the article belongs in that series, checking existing order values first.

## Verify both builds

Use fresh temporary directories. Never delete or regenerate the repository's `public/` merely to check an article.

```bash
preview=$(mktemp -d /tmp/samirtech-drafts.XXXXXX)
production=$(mktemp -d /tmp/samirtech-production.XXXXXX)
hugo --buildDrafts --minify --destination "$preview"
hugo --minify --destination "$production"
test -f "$preview/posts/my-new-article/index.html"
test ! -e "$production/posts/my-new-article/index.html"
```

Inspect the rendered draft, links and front matter. Check that the draft's title and URL do not leak into production indexes, feeds or sitemaps. Review source and rendered content for secrets, private addresses, paths and identifying infrastructure details.

Preview locally when wanted:

```bash
hugo server --buildDrafts --bind 127.0.0.1
```

Do not expose a draft-enabled preview publicly. Files in `static/` are copied even when the related article is a draft. Keep unpublished supporting assets outside `static/`; `docs/archive/` stores unused diagram alternatives and is not published by Hugo.

## Save the intended source only

```bash
git add content/posts/my-new-article.md
git diff --cached --check
git diff --cached --stat
git commit -m "Draft new article"
```

Stage documentation or assets by exact path when intentional. Do not use broad staging or stage generated `public/` output. A local commit is not a push or a publication.

## Publish only after approval

1. Change `draft: true` to `draft: false` after editorial approval.
2. Ensure the publication date is not in the future. A future date also requires a subsequent build when that date arrives; time passing alone does not redeploy the site.
3. Build into another fresh temporary destination and inspect the exact generated article, navigation, feed and sitemap.
4. Review and commit only intended source changes.
5. Confirm the production branch and ancestry. Merge through a reviewed pull request, or use an explicitly approved fast-forward push to production. `git push origin main` does not push a topic branch's current HEAD.
6. Check the validation run and Cloudflare deployment for the exact commit.
7. Fetch the canonical article URL and verify distinctive new text, not just HTTP 200. If necessary allow a short propagation delay and retry with `curl -fsSL`.

A successful build or push is not proof that production changed. Tracking old files under `public/` does not make them the deployment source. Hugo deprecation warnings are maintenance items unless they cause the build to fail.
