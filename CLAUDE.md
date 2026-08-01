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

`qiita.config.json` controls the preview host/port and `includePrivate` (currently `false`, so private articles are excluded from preview).

## Article frontmatter contract

Every file in `public/` is a Markdown document with YAML frontmatter. The fields are meaningful to the publish flow, not decorative:

- `id` — `null` for a not-yet-published article. After the first publish, Qiita assigns an ID and **the CLI renames the file to `<id>.md`**. Never hand-edit `id` or rename an already-published file; that breaks the link to the remote article.
- `updated_at` — server-managed; leave it alone.
- `ignorePublish` — set `true` to keep a draft out of `publish --all`.
- `private` — `true` publishes as a limited-share article rather than public.
- `tags` — Qiita requires at least one non-empty tag to publish; the `new` stub ships with a single empty string that must be replaced.

The unpublished stub (`public/newArticle001.md`) shows the shape produced by `qiita new`; the numeric-named files are already-published articles.

## Publishing pipeline

`.github/workflows/publish.yml` runs `increments/qiita-cli/actions/publish@v1` on every push to `main`/`master`, using the `QIITA_TOKEN` repository secret. **Merging to `main` publishes to Qiita.** Treat commits on `main` as production deploys of article content — content changes go out automatically, so preview locally before pushing.

## Dev container

`.devcontainer/setup.sh` installs the CLI agents and runs `npm install @qiita/qiita-cli --save-dev` on container create, which is why `package.json`/`package-lock.json` sometimes show as modified after a rebuild. Articles are authored in Japanese; keep new content and commit messages consistent with that.
