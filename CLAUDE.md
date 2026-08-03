# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is **not** the Qiita CLI source code. It is a *content* repository: Qiita articles stored as Markdown under `public/`, managed with the [`@qiita/qiita-cli`](https://github.com/increments/qiita-cli) npm package (installed as a devDependency). There is no application code, build step, linter, or test suite here — the "code" is the Markdown and its frontmatter.

## Commands

```bash
npx qiita login              # Qiita API 認証 (writes credentials.json, gitignored)
npx qiita new <basename>     # create public/<basename>.md with a fresh frontmatter stub
npx qiita preview            # local preview server (localhost:8888, per qiita.config.json)
npx qiita pull               # sync articles down from Qiita into public/
npx qiita publish <basename> # publish/update one article
npx qiita publish --all      # publish/update everything
```

`qiita.config.json` controls the preview host/port and `includePrivate` (set to `true`, so limited-share articles show up in preview — and, more importantly, get synced; see the republish loop below).

## Article frontmatter contract

Every file in `public/` is a Markdown document with YAML frontmatter. The fields are meaningful to the publish flow, not decorative:

- `id` — `null` for a not-yet-published article. On first publish Qiita assigns an ID and writes it back into the frontmatter. Never hand-edit `id`; that breaks the link to the remote article. Note that publishing **via the GitHub Action does not rename the file** to `<id>.md` — `ai-driven-development-glossary-1.md` kept its descriptive basename and only gained `id`/`updated_at`. The basename is just a local handle, so either naming style works; don't rename a published file to "fix" it.
- `updated_at` — server-managed; leave it alone. It is also what decides the staleness check described below, so never backdate it by hand.
- `ignorePublish` — set `true` to keep a draft out of `publish --all`.
- `private` — `true` publishes as a limited-share article rather than public.
- `tags` — Qiita requires 1–5 non-empty tags to publish; the `new` stub ships with a single empty string that must be replaced.

The unpublished stub (`public/newArticle001.md`) shows the shape produced by `qiita new`; the numeric-named files are already-published articles.

## Publishing pipeline

`.github/workflows/publish.yml` runs `increments/qiita-cli/actions/publish@v1` on every push to `main`/`master`, using the `QIITA_TOKEN` repository secret. **Merging to `main` publishes to Qiita.** Treat commits on `main` as production deploys of article content — content changes go out automatically, so preview locally before pushing.

After a successful run the Action pushes an `Updated by qiita-cli` commit back to `main`, writing the assigned `id` and the server's `updated_at` into the frontmatter. **Always `git pull` after publishing**, or your next push will conflict — or worse, resurrect the stale `id: null` frontmatter.

### Publish failure modes

`publish --all` collects every article that is not `ignorePublish: true` and is either modified or has `id: null`, validates the whole batch, and calls `process.exit(1)` if *any* item fails. **One bad file blocks every other article in the repository.** The failure is all-or-nothing, so a green pipeline can turn red because of a file you never touched. Three causes have actually bitten this repo:

1. **Missing `QIITA_TOKEN` secret.** The CLI uses the `QIITA_TOKEN` env var if set, and otherwise falls back to reading `~/.config/qiita-cli/credentials.json` — which does not exist on a runner. The symptom is a fast (~10s) failure ending in `ENOENT: no such file or directory, open '/home/runner/.config/qiita-cli/credentials.json'`, which reads like a missing-file bug but is really an unset secret. Check with `gh secret list`.
2. **A leftover `qiita new` stub.** The stub ships `tags: ['']`, and validation rejects empty tag strings. Since the stub also has `id: null` it is always a publish target. Either fill in the tags or set `ignorePublish: true` before committing it.
3. **A local copy older than Qiita.** If an article was edited in the Qiita web editor after it was last committed, the local file both differs from the remote and has an older `updated_at`. That trips the "内容がQiita上の記事より古い可能性があります" check and aborts the batch. Fix it by taking the remote version — `rm public/<id>.md && npx qiita pull` — since plain `pull` will not overwrite a locally-modified file. Do **not** reach for `publish --force`; that pushes the stale local copy over the newer published one.

A fourth failure mode is silent rather than red: **an endless republish loop on limited-share articles.** `syncArticlesFromQiita` filters out `private` items unless `includePrivate` is `true`, so with it off a limited-share article is never written to the remote cache. `modified` is computed as `!localFileContent.equals(remoteFileContent)`, and `equals(null)` is `false` — so the article looks modified forever, gets republished on every single push, and each run bumps `updated_at` and adds another `Updated by qiita-cli` commit. `includePrivate: true` is what stops it; keep it that way as long as any `private: true` article lives in `public/`.

Beware also that the body stored by Qiita is not byte-identical to what the CLI considers canonical — the raw API `body` field carries an extra trailing newline that `rawBody` does not. Diff against the CLI's own remote cache (`getItemData(filename, true)`), not against the API response, or you will "fix" a difference that does not exist and create a real one.

To debug without publishing anything, reproduce the CI conditions locally: clone the repo to a scratch directory, set `QIITA_TOKEN`, and run the CLI's own target-selection and validation against it. `loadItems()` exposes `modified`, `ignorePublish` and `isOlderThanRemote` per article, which is enough to see which file is blocking the batch.

## Images

Neither the CLI nor the Qiita API can upload images — the API has no such endpoint. There are two workable approaches, and this repo uses both:

- **Qiita's own storage** (older articles) — drag the image into the Qiita web editor and paste the resulting `qiita-image-store.s3.amazonaws.com` URL back into the Markdown.
- **GitHub raw URLs** (`ai-driven-development-glossary-1`) — commit the images under `images/<article-basename>/` and reference them as `https://raw.githubusercontent.com/shou-dev19/qiita-cli/main/images/<article-basename>/<file>.png`. Qiita serves those URLs as-is rather than proxying them, so **this only works while the repository is public**, and the images must be pushed before or with the article or the published page renders broken images.

Pinning the URL to `main` means re-committing a file under the same name updates the live article; pinning to a commit SHA instead freezes it. Article figures are flat-colour illustrations, so quantizing to a 256-colour palette (`convert in.png -strip -colors 256 -depth 8 PNG8:out.png`) cuts roughly 75% of the bytes with no visible loss — worth doing before committing, since these live in git forever.

## Dev container

`.devcontainer/setup.sh` installs the CLI agents and runs `npm install @qiita/qiita-cli --save-dev` on container create, which is why `package.json`/`package-lock.json` sometimes show as modified after a rebuild. Articles are authored in Japanese; keep new content and commit messages consistent with that.
