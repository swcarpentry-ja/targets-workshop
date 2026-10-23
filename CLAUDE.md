# CLAUDE.md

Notes for Claude on how this repo's translation setup works. Read this before touching translations, CI, or branches.

## What this repo is

A Japanese translation of the Carpentries Incubator lesson
[targets-workshop](https://github.com/carpentries-incubator/targets-workshop)
(an introduction to the `targets` R package). The lesson is built with the
Carpentries Workbench (sandpaper + varnish + pegboard).

- This repo: `swcarpentry-ja/targets-workshop` (a fork of the upstream lesson)
- Upstream: `carpentries-incubator/targets-workshop` (no `upstream` remote is configured; add one or fetch by URL when merging)
- Published site: https://swcarpentry-ja.github.io/targets-workshop/ (gh-pages)
- Maintainer: Joel Nitta (joelnitta@gmail.com)

Translation tooling is the R package [dovetail](https://github.com/joelnitta/dovetail)
(`joelnitta/dovetail`). There is no Crowdin config, no `po/` files, and no
automated "update translations" tool: stale translations are fixed by hand
(or by Claude) by diffing against upstream.

## Branch model

| Branch | Contents |
|---|---|
| `main` | The English lesson, kept in sync with upstream `main`. Do not put translations here. |
| `l10n_main` | `main` plus the `locale/ja/` overlay. This is the working/translation branch and what CI builds. |
| `gh-pages` | Built site, written by CI. Never edit by hand. |
| `md-outputs` | Intermediate markdown outputs from the Workbench; leave alone. |

Upstream changes flow: upstream `main` -> this repo's `main` / merged into `l10n_main`.
After merging, `locale/ja/` is usually stale and needs re-translating (see below).

## How the overlay works

`locale/ja/` mirrors the lesson's file layout (`config.yaml`, `index.md`,
`episodes/*.Rmd`, `learners/`, `instructors/`, `profiles/`, ...). To build,
dovetail's `make_translated_dir()`:

1. rsyncs the full English lesson (repo root) into a temp directory, then
2. overlays everything from `locale/ja/` on top (same relative paths win).

So:
- Any file with no counterpart in `locale/ja/` falls back to the English one (figures in `episodes/fig/`, `renv/`, `episodes/data/`, etc.).
- `locale/ja/config.yaml` **replaces** the root `config.yaml`, including the episode list. A new upstream episode must be added to **both** (root `config.yaml` and `locale/ja/config.yaml`) or it won't appear in the Japanese site. `lang: ja` lives at the end of the locale config.
- R code chunks in translated `.Rmd` files must stay functionally identical to upstream; only prose, comments meant for readers, chunk captions, and headings are translated. Chunk labels, `file=`/`code=` references (e.g. `plan_6.R[2:28]`), and `source()` paths must match upstream exactly, since they point at files that the lesson generates at build time. After an upstream change to line ranges like `[2:28]`, update the translation too.
- Translated episode front matter (`title`, `teaching`, `exercises`, `questions`, `objectives`, `keypoints`) is translated; numeric `teaching`/`exercises` values should match upstream.
- Heading text produces anchor ids (Japanese headings give Japanese anchors like `#背景-targets-を使用しないバージョン`). Internal links to headings must be checked against the built site.

## Updating translations after an upstream change

1. Merge upstream `main` into `l10n_main` (resolve conflicts; keep `locale/` untouched).
2. For each upstream-changed file, diff the English file against its base and against `locale/ja/<same path>`. The translation base commit is recorded in git history (initially `b981d30`); use `git diff <old>..<new> -- episodes/ learners/ ...` to see what changed.
3. Update `locale/ja/` to match: new episodes get a new translated file plus an entry in `locale/ja/config.yaml`; removed episodes get removed from both.
4. Match the **existing wording and style** (register, terminology, how code terms like `targets` are set in backticks and spaced from Japanese text). Look at neighbouring translated episodes before writing new text.
5. Render locally (below), then have Joel review the translation diff.

## Building locally

Use dovetail, from an R session at the repo root (on `l10n_main`):

```r
options(sandpaper.use_renv = FALSE)   # use the local library instead of restoring renv.lock
dovetail:::make_translated_dir(
  translated_dir = "/tmp/ja-lesson", overwrite = TRUE, lang = "ja",
  l10n_branch = NULL, clean = FALSE)
sandpaper::build_lesson("/tmp/ja-lesson", preview = FALSE)
```

(`render_trans_from_dir()` / `render_trans_from_branch()` in dovetail are the higher-level wrappers.) Use the session scratchpad instead of `/tmp` when working as Claude.

- Local `renv::restore()` of the lesson lockfile can fail to compile old packages on a new R (it did on R 4.6). Building with `sandpaper.use_renv = FALSE` avoids this; make sure `targets`, `tarchetypes`, `tidyverse`, `broom`, `quarto`, `palmerpenguins`, `visNetwork` etc. are installed locally.
- `build_lesson()` has no `skip_manage_deps` argument; use the option above.
- Builds are the real test of a translation: a stale translation can fail with errors such as `object 'example_branch_name' not found` because the R code no longer matches the generated files.
- Quarto-dependent episode (`quarto.Rmd`) needs Quarto installed.

## CI / deployment

`.github/workflows/sandpaper-main.yaml` ("01 Build and Deploy Site"):

- Checks out `l10n_main` (not `main`).
- Writes the repo secret `LANG_CODE` (set to `ja`) into `~/.Renviron`, installs `carpentries/sandpaper` and `joelnitta/dovetail`.
- Runs `dovetail:::make_translated_dir()` into a temp dir, then `sandpaper:::ci_deploy()` to push the result to `gh-pages`.
- Pushing to `l10n_main` already starts the build and deploy; no manual trigger is needed. To run it manually: `gh workflow run "01 Build and Deploy Site" -R swcarpentry-ja/targets-workshop --ref l10n_main`.
- **Always pass `-R swcarpentry-ja/targets-workshop` to `gh` commands** (the default repo is also set with `gh repo set-default`, but don't rely on it). Without it, `gh` can target upstream `carpentries-incubator/targets-workshop`, and workflow dispatches/reruns would run there by mistake. Check with `gh repo view --json nameWithOwner` if unsure.
- If a run fails, look at the fork's own runs (`gh run list -R swcarpentry-ja/targets-workshop`) before changing any workflow files; workflow files are upstream boilerplate and should not be edited to chase a failure seen in a different repo.
- GitHub disables scheduled workflows after 60 days of repo inactivity (`disabled_inactivity`); re-enable with `gh workflow enable`.
- Other workflows (`pr-*.yaml`, `update-workflows.yaml`, `update-cache.yaml`) are upstream Workbench boilerplate; `update-workflows.yaml` bumps them (sandpaper version is in `sandpaper-version.txt`).

### Lockfile

The lesson's R dependencies are pinned in `renv/profiles/lesson-requirements/renv.lock`.
The CI runner uses the newest R (4.6.x as of Oct 2026), and old pinned versions (rlang, Rcpp, data.table, Matrix, isoband, base64enc, ...) fail to compile against R 4.6's removed C API. The fork's lockfile was therefore regenerated with current package versions (119 packages) via `renv::snapshot()` and CI passes with it. The same problem exists in the upstream lockfile (upstream issue #82). If CI fails in the "Setup Package Cache" step with C compile errors, regenerate the lockfile the same way rather than patching individual packages. Take care when merging upstream: do not let an old upstream lockfile overwrite the regenerated one.

## Things to check before pushing

- `git status` is on `l10n_main`; no stray changes to English files unless they came from an upstream merge.
- Every episode in the root `config.yaml` is also in `locale/ja/config.yaml` and has a file in `locale/ja/episodes/`.
- A local dovetail build finishes without errors.
- Pushing to `l10n_main` triggers a public deploy; confirm with Joel first.
