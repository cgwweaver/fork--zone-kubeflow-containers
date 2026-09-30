colin to do
- read chat, vs this
- split this into style/work start and goals vs dep/etc info

check exact mamba ran!


# CLAUDE.md — context for work on zone-kubeflow-containers (fork)

> Working notes for Claude Code. Lives on the `claude-info` branch only.
> **Never commit this file into a PR branch.** On a PR branch:
> `git checkout claude-info -- CLAUDE.md && echo CLAUDE.md >> .git/info/exclude`

## work plan

require my explicit approval before pr upstream!!

can draft pr inside this fork if that helpful for your workflow


## Repo facts (verified on upstream `beta` @ d8c24bf, 2026-09-29)

- Upstream: `StatCan/zone-kubeflow-containers`. Fork: `cgwweaver/fork--zone-kubeflow-containers`. PRs target **`beta`** (auto-merge/promote workflows run weekly).
- Image chain: `base → mid → {jupyterlab, rstudio, sas_kernel, sas}`. R lives in `mid`.
- **PR #298 (merged to beta 2026-09-22)**: R 4.6.1 = conda `r-base` interpreter only; R packages = prebuilt **PPM Ubuntu 24.04 (noble) binaries**.
  - `images/mid/Dockerfile`: removes conda `r-*`, installs about 30 packages from PPM with a completeness check, adds apt sysreqs (PPM sysreqs API for that set, minus what `base` has), writes a version-bearing `HTTPUserAgent` to `/opt/conda/lib/R/etc/Rprofile.site`.
  - `images/mid/s6/cont-init.d/02-start-custom`: at startup, appends Artifactory repos `ppm` (`zone-r-ppm-bin-<codename>-<R major.minor>-remote`) and `cran-local` to Rprofile.site.
  - `images/mid/.Rprofile`: dropped `options(download.file.method="wget")`, because wget hides the user agent and then PPM serves **source** instead of binaries.
  - `images/base/s6/cont-init.d/01-copy-tmp-home`: copies `/tmp_home/$NB_USER` into `$HOME` with `cp --update=none`, so **existing homes keep their old `~/.Rprofile`** (wget) and silently keep compiling from source.
  - `tests/general/test_r.py`: checks R version, loads each package, checks the ir kernelspec.
  - Supersedes #290 (heavier route: CRAN apt R, scipy-notebook base).
- `images/base/Dockerfile` already has, among others: libxml2-dev, libglpk-dev, fontconfig, freetype, harfbuzz, fribidi, libtiff. `mid` adds: gdal, geos, proj, sqlite3, udunits2, abseil, libgit2, libuv, libwebp, hdf5, icu, zmq, x11.
- Not seen in either Dockerfile (verify): `cmake`, `pandoc`, `libnode`/V8.
- Fork CI: docker workflows use repo secrets, so they likely won't run in the fork. Test locally (Docker) or via an upstream PR.

## Background from earlier chats (condensed)

- **Why packages have sysdeps:** mostly they wrap mature C/C++ libraries (GDAL, libxml2, curl, openssl, libgit2). Other reasons: the fonts/graphics stack; cmake to build vendored code; external CLIs (pandoc, Chrome, Python). Performance usually needs only a compiler.
- **Hidden chains that matter for survey/tabulation users:**
  - `gt → juicyjuice → V8`
  - `tidyverse → ragg (fonts/png) + xml2 + curl`
  - `flextable → gdtools (cairo) + officer (ragg, openssl)`
  - `kableExtra → svglite`
  - `targets → igraph`
  - `qs2 → RcppParallel (cmake ≥ 3.5)`
  - `car / ggpubr → lme4 → nloptr (cmake or libnlopt)`
  - `cancensus / leaflet → sf → geo stack`
  - `reticulate → png (libpng) + Python`
- **Sysdep-free (fine):** data.table, dtplyr, collapse, duckdb/duckplyr (long compile only), haven (zlib), knitr, survey, srvyr, janitor, readxl, openxlsx2, nanoparquet, gridExtra, cowplot, patchwork.
- **Compile-time vs runtime:**
  - cmake, compilers and headers are needed only at install. Ephemeral is fine.
  - Shared libs (`.so`) and CLIs are needed at runtime. They must live in the image or in `$HOME`.
  - Vendored builds (RcppParallel's TBB, fs's libuv) are self-contained in the user lib.
- **Persistence in a notebook pod:**
  - Wiped on restart: `/opt/conda`, `/usr`, `/tmp` (includes anything you `mamba install`).
  - Persists: `$HOME` (user R lib `~/R/r-packages-X.Y`, `~/.local` from `pip --user`, `~/.Renviron`, `~/.R/Makevars`).
- **Conda R + system headers trap:** configure finds `/usr/include`, but the conda linker doesn't search `/usr/lib`. Seen as igraph `cannot find -lxml2` when building from source. PPM binaries avoid the build step entirely.
- **Air-gap traps:** these download during install or at runtime: V8 (static libv8), arrow (libarrow), torch, and DuckDB extensions other than parquet (httpfs, spatial, excel…).
- **Correction to an earlier chat:** it claimed "PPM binaries don't fit conda R". In practice the maintainers verified that they load fine (#298). Posit only formally guarantees them for distro R, so keep a load test.
- **Colin's state (2026-09):** on R 4.6.1. Built qs2/targets from source by working around cmake/libxml2 (`mamba install cmake libxml2 libxml2-devel`). Working now.

## Goals

1. **Me first:** get PPM binaries in my own home (see Todo 1). Likely the root cause of all my source compiles.
2. **Upstream, small and high-value:** make sure existing homes get binaries too, not just new homes.
3. **Upstream, medium:** make commonly user-installed packages (beyond the shipped 30) load as PPM binaries, by adding their runtime sysreqs to the image, using the same PPM-sysreqs approach as #298.
4. **Upstream, test:** a canary test that installs and loads a list of common user packages from PPM and fails on missing shared libs (`ldd` "not found").
5. **Nice-to-have:** user docs on persistence (what survives restart) and how to install R packages; confirm pandoc availability in Jupyter.

## Todo

1. [ ] **Diagnose in the Zone (no code).** Run `getOption("download.file.method")`, `getOption("HTTPUserAgent")` and `getOption("repos")`.
   - If `"wget"`: back up `~/.Rprofile`, remove that line (or `cp /tmp_home/$USER/.Rprofile ~/.Rprofile`), restart R.
   - Reinstall a heavy package, e.g. `install.packages("igraph")`. Expect a fast binary install, no compile output.
   - Note: the old `.Rprofile` also puts older `r-packages-*` dirs on `.libPaths()`, so 4.5-built packages can shadow 4.6 ones. Consider trimming that.
2. [ ] **Issue plus PR: stale `~/.Rprofile` in existing homes.**
   - Option A: in `02-start-custom`, idempotently comment out `download.file.method="wget"` in `$HOME/.Rprofile`, with a log message.
   - Option B: set `download.file.method` and `HTTPUserAgent` in Rprofile.site *after* the user profile… not possible, since the user profile runs last. So A, or a one-time migration notice.
   - Check whether the aaw#569 wget workaround is still needed anywhere (RStudio "internet routines cannot be loaded").
3. [ ] **Broader sysreqs PR.**
   - Candidate package list: gt, V8, igraph, targets, qs2, RcppParallel, flextable, officer, ragg, kableExtra, terra, lme4, car, nloptr, duckdb, gert, usethis, haven, data.table, collapse, survey, srvyr, janitor, openxlsx2, nanoparquet, reticulate, cancensus.
   - Query the PPM sysreqs API for noble and diff against the base+mid apt lists.
   - Add missing runtime libs, noting the size impact.
4. [ ] **Extend `tests/general/test_r.py`** with a canary list: install from PPM, `library()`, and an `ldd` scan for "not found".
5. [ ] Decide whether `cmake` and `pandoc` belong in `mid`, for source-only packages and for rmarkdown outside RStudio.
6. [ ] Optional: explore `cran-local` (the Artifactory local CRAN repo) for team-built binaries of internal packages.

## Open questions

- Do conda's libs in `/opt/conda/lib` ever shadow system libs for PPM binaries (ABI mismatch)? The canary test should reveal this.
- Which image and branch is the Zone actually running for users: `beta` or `master`?
