# CLAUDE.md

This repo is a Japanese translation of the Carpentries Incubator lesson
[targets-workshop](https://github.com/carpentries-incubator/targets-workshop)
(an introduction to the `targets` R package), built with the Carpentries
Workbench and translated with the dovetail `locale/ja/` overlay.

The general method (branches, overlay, translation rules, checker, local
build, CI) is in the user-level skill `carpentries-translate`. Load it before
touching translations, CI or branches. This file records only what is
specific to this repo.

## Repo facts

- Fork: `swcarpentry-ja/targets-workshop` (always pass `-R swcarpentry-ja/targets-workshop` to `gh`)
- Upstream: `carpentries-incubator/targets-workshop` (git remote `upstream`)
- Language code: `ja`, from the repo **variable** `LANG_CODE`
  (`gh variable list -R swcarpentry-ja/targets-workshop`). If the site ever
  deploys in English, check this first. The old `LANG_CODE` secret is unused.
- Deploy workflow: Docker `docker_build_deploy.yaml` ("01 Maintain: Build and
  Deploy Site"), unmodified from upstream. The legacy `sandpaper-main.yaml`
  with hand-made dovetail changes was dropped in the Nov 2026 upstream merge.
- Published site: https://swcarpentry-ja.github.io/targets-workshop/ (from `gh-pages`)
- Maintainer: Joel Nitta (joelnitta@gmail.com)
- Translation base (upstream commit the translation matches): `ac9f3a6`
  (2026-11-27). Content last changed upstream at `71a2a6b` (2026-10-23); later
  commits up to `ac9f3a6` were workflow and lockfile updates only. The first
  translation (Oct 2024) was made from `b981d30`.

## Deliberate differences from upstream

- `CLAUDE.md` (this file). Nothing else outside `locale/`:
  `git diff --stat upstream/main l10n_main -- . ':!locale'` should list only it.

History: from Oct 2026 the fork carried its own regenerated
`renv/profiles/lesson-requirements/renv.lock`, because old pinned versions
failed to compile on R 4.6 (upstream issue #82). Upstream has since
regenerated its lockfile for R 4.6 (`944ddda`), so the fork uses upstream's
lockfile again.

## Lesson-specific notes

- Episodes generate `episodes/files/plans/plan_*.R` at build time and show
  slices of them with `code = readLines("files/plans/plan_6.R")[2:21]`. When
  upstream changes a plan, these line ranges change too; a stale range gives
  errors such as `object 'example_branch_name' not found` or shows the wrong
  lines. The skill's checker catches range mismatches.
- `quarto.Rmd` needs Quarto installed for a local build.
- Local build with `sandpaper.use_renv = FALSE` needs at least `targets`,
  `tarchetypes`, `tidyverse`, `broom`, `quarto` and `visNetwork` in the local
  library (the penguins data comes from base R since upstream #101).
- Japanese headings give Japanese anchors (e.g. `#背景-targets-を使用しないバージョン`);
  `packages.Rmd` keeps upstream's explicit `{#method-2}` id.
- Upstream `branch.Rmd` says "load `model_summaries`" where the plan no longer
  has that target (upstream issue #81); the Japanese file mirrors it until
  upstream fixes it.
- Code comments in chunks were left in English.
- Latin words next to Japanese are spaced inconsistently (`RStudioの` / `RStudio の`);
  follow the surrounding text.
- A setup-r change made to chase a CI failure seen elsewhere was unnecessary
  and was reverted (`4412bc3`): this fork builds fine on its own runner.
