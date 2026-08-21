# Local `planemo test` findings (2026-07-20/21)

Notes from debugging a local `planemo test` run of `minfi_analysis.xml` under WSL2 Ubuntu on Windows. Kept here so the two issues found aren't re-discovered from scratch next time.

## 1. Manifest/annotation packages fail to load ("cannot load manifest package")

**Symptom:**

```
Loading required package: IlluminaHumanMethylation450kmanifest
Error in getManifest(object) :
  cannot load manifest package IlluminaHumanMethylation450kmanifest
Calls: preprocessRaw ... getManifestInfo -> getProbeInfo -> getManifest -> getManifest
Execution halted
```

**Root cause:** `macros.xml` pins old bioconda packages (`r-base=3.6.2`, `bioconductor-minfi=1.32.0`, and the 27k/450k/EPIC manifest + annotation packages). Those data packages ship as a raw R source tarball that only gets installed via a conda `post-link.sh` hook (`installBiocDataPackage.sh`, which downloads the tarball from bioconductor.org and runs `R CMD INSTALL`).

Planemo's local dependency resolution satisfies the tool's requirements by pulling a pre-built "mulled" combined environment (`mulled-v1-<hash>`) rather than running `conda create` package-by-package. When that mulled bundle is unpacked, the post-link scripts are never executed. Compiled packages (`minfi`, `rtracklayer`, etc.) work fine since they don't need post-link steps, but every `IlluminaHumanMethylation*manifest` / `*anno.*` package is left as an unextracted tarball under `share/`, never registered in R's `lib/R/library`.

**Fix applied (diagnostic environment only, not a tool-code change):** manually ran each of the 6 post-link scripts against the existing mulled env:

```bash
PREFIX=/path/to/miniconda3/envs/mulled-v1-<hash>
export PREFIX PATH="$PREFIX/bin:/usr/bin:/bin"
for pkg in bioconductor-illuminahumanmethylation27kmanifest \
           bioconductor-illuminahumanmethylation27kanno.ilmn12.hg19 \
           bioconductor-illuminahumanmethylation450kmanifest \
           bioconductor-illuminahumanmethylation450kanno.ilmn12.hg19 \
           bioconductor-illuminahumanmethylationepicmanifest \
           bioconductor-illuminahumanmethylationepicanno.ilm10b4.hg19; do
  bash "$PREFIX/bin/.$pkg-post-link.sh"
done
```

This downloads each package tarball from `bioconductor.org` and runs `R CMD INSTALL` — after this, `minfi_analysis`'s R script correctly loads all manifest/annotation packages and proceeds into `preprocessRaw`/`bumphunter`. Confirmed working twice, in two independent test runs.

**Implication for CI:** this may or may not reproduce on Travis — it depends on whether Travis's dependency resolution path also falls back to unpacking a pre-built mulled bundle, or actually runs `conda create` (which executes post-link scripts normally). Worth checking Travis logs for the same "cannot load manifest package" error if CI ever fails there.

## 2. `bumphunter`'s "Finding regions" step never completes locally (memory ceiling)

**Symptom:** after the manifest fix, the R script proceeds past `preprocessRaw`/QC into `bumphunter`, prints `[bumphunterEngine] Finding regions.`, and then runs indefinitely (confirmed 20+ minutes with zero further output, zero output files, no crash) despite `B=0` (no permutations) which should make this step take seconds.

**Root cause:** this specific dev machine has only 6.9GB total RAM, with the WSL2 VM capped at 5.3GB (`.wslconfig`: `memory=5632MB`). Running Galaxy's local job-runner stack (gunicorn + 2 celery processes) simultaneously with R loading the EPIC/450k annotation packages and scanning genome-wide probe positions pushes memory to the ceiling; the process survives (no OOM kill observed in dmesg/journalctl) but swaps so heavily that it doesn't practically finish.

**Reproduced twice** (two independent full test runs) at the exact same point, confirming this is a deterministic hardware/VM-memory limitation on this machine, not an intermittent fluke.

**Things that helped but weren't sufficient on their own:**
- Killing orphaned Galaxy/celery/gunicorn/node processes left running from earlier `planemo test` invocations (each full stack persists ~700MB-1GB after the parent `planemo test` process is killed, since `supervisord` daemonizes them — `pkill -f 'planemo test'` does **not** clean these up; kill `supervisord`, `celery`, `gunicorn` explicitly, or better, kill by explicit PID since `pkill -f` proved unreliable in this WSL setup)
- Patching planemo's local install to skip the `gx-it-proxy` (interactive tools) node.js process it launches for every test run — real but small savings
- Attempting to disable `enable_celery_tasks` in planemo's generated `galaxy.yml`: **does not work** — gravity/supervisord starts the celery worker/beat processes regardless of this flag; it only affects whether Galaxy internally *dispatches* tasks to celery, not whether the process runs at all

**Conclusion:** getting a real green `planemo test` run for `minfi_analysis` needs either a machine/VM with more RAM (8GB+ free recommended) or reliance on CI, which won't share this constraint. Further local retries on this machine are not expected to succeed without a memory increase.

## Environment gotchas hit while doing this (WSL2 + Git Bash on Windows)

- `wsl.exe -d Ubuntu -- bash -lc "..."` calls with `/mnt/c/...` path arguments get mangled by Git Bash's MSYS path auto-conversion unless prefixed with `MSYS_NO_PATHCONV=1`.
- Inline shell variable assignment + read within a single `bash -lc "VAR=x; echo $VAR"` string is unreliable through this particular Bash-tool → `wsl.exe` invocation path (assignment silently doesn't persist) — write a real script file and execute that instead, or use fully literal values.
- The WSL2 VM's `/tmp` is not guaranteed to survive a VM restart (observed during a low-disk-space incident) — use `--no_cleanup` and point `--job_output_files`/`--test_output_json` at a path on the Windows-mounted filesystem if you need results to survive an interruption.

# Local `planemo test` findings (2026-08-05/08), Apple Silicon Mac

Notes from getting `minfi_analysis.xml` and the `ewas_harmonisation` stage tools running locally on a 16GB M2 MacBook Air (macOS 15.6, arm64). Neither of the two WSL2 issues above reproduced here (this machine already has enough RAM, and — as it turns out — a pre-existing host R install masked the manifest-loading issue entirely; see below). But getting genuinely *isolated* test execution working surfaced a new chain of issues, each hiding the next. Kept here so it isn't re-discovered from scratch.

## 1. Docker/mulled container resolution silently fails, falls back to unmanaged execution

**Symptom:** `planemo test --biocontainers` reports tests as passing (or failing on ordinary script bugs), but the job isn't actually running inside the container it built. Confirmed by adding a diagnostic `cat(.libPaths(), ...)` to a tool's R script: it printed `/Library/Frameworks/R.framework/...` — a macOS-only path that can never exist inside a Linux container.

**Root cause:** every container resolver Galaxy tries (`ExplicitContainerResolver`, both `CachedMulledDockerContainerResolver` variants, `MulledDockerContainerResolver`, `BuildDockerContainerResolver`) logs `found description [None]` for tools with large/bespoke multi-package requirement sets, even after `BuildDockerContainerResolver` has actually built a working image (confirmed: running that exact image by hand with the tool's exact generated R script works fine). The resolver's own post-build verification is failing/timing out on this Docker/OrbStack arm64 setup. Since `--biocontainers` also deliberately empties `resolvers_conf.xml` (to force container-only resolution — see planemo's `galaxy/config.py`), Galaxy is left with *no* dependency management path at all when container resolution also fails, and silently runs the raw command against whatever's on `PATH` instead of hard-failing.

**Implication:** don't trust a "biocontainers" test pass on this machine without independently confirming the job actually ran in a container (e.g. a `.libPaths()`/`sessionInfo()` diagnostic, or checking `docker ps -a` timestamps against the job's runtime). This machine happened to have a fairly complete pre-existing Bioconductor/minfi R install (unrelated prior work), which made the silent fallback look like a pass for some tools and only surfaced as a real failure where the host install had a gap (missing `nleqslv`, breaking `wateRmelon::betaqn`).

**Not fixed** — worked around by switching to `--conda_dependency_resolution` instead of chasing this further.

## 2. Conda dependency resolution on osx-arm64: Bioconductor packages aren't built for it

**Symptom:** `conda create` for `bioconductor-minfi` (or anything depending on it) fails immediately:
```
LibMambaUnsatisfiableError: Encountered problems while solving:
  - nothing provides bioconductor-biocparallel >=1.36.0,<1.37.0 needed by bioconductor-minfi-1.48.0-r43hdfd78af_0
```

**Root cause:** bioconda mostly publishes `linux-64` and `osx-64` (Intel) builds for compiled Bioconductor packages; native `osx-arm64` builds are missing for much of this dependency graph.

**Fix:** force the Intel subdir and rely on Rosetta 2 emulation — set globally in `~/.condarc`:
```yaml
subdir: osx-64
```
(equivalent to `CONDA_SUBDIR=osx-64`, but see #3 below for why the env-var form matters more than expected). Confirmed: the exact same `conda create` command that failed on `osx-arm64` resolves and installs cleanly under `osx-64`.

## 3. Conda 26.7.0 bug: `validate_subdir_config()` crashes when no env is active

**Symptom:** with `subdir: osx-64` set via `.condarc` (a global/file source, not env var or CLI flag), `conda create` invoked with no environment currently active (exactly how Galaxy invokes it as a subprocess) crashes instead of installing:
```
TypeError: expected str, bytes or os.PathLike object, not NoneType
  File ".../conda/cli/common.py", line 226, in validate_subdir_config
    elif not paths_equal(context.active_prefix, context.root_prefix):
  File ".../conda/common/path/__init__.py", line 119, in paths_equal
    return abspath(path1) == abspath(path2)
```

**Root cause:** `context.active_prefix` is `None` when no env is active; `validate_subdir_config`'s file-sourced-config branch calls `paths_equal(None, context.root_prefix)` without a None-guard, even though the function's own docstring says a subdir override from "a global file" should be fine as long as it's "a base env" (which "no active env" trivially is).

**Fix applied (patches the local conda install, not a repo file):** added a None-guard in `<conda_prefix>/lib/python3.13/site-packages/conda/cli/common.py`, in `validate_subdir_config`, right before the `paths_equal(context.active_prefix, context.root_prefix)` branch:
```python
elif context.active_prefix is None:
    pass  # no active env (e.g. invoked from a subprocess with no env
    # activated) is equivalent to using base -- this is ok
elif not paths_equal(context.active_prefix, context.root_prefix):
```
Setting `CONDA_PREFIX`/`CONDA_DEFAULT_ENV` env vars before invoking `conda create` also avoids the crash (env-var-sourced subdir config takes a different, safe branch) but did **not** reliably propagate through Galaxy's subprocess invocation in practice — the source patch is the durable fix. Re-apply after any Miniforge/conda upgrade, since it edits an installed package file directly.

## 4. Failed/interrupted conda env-creation attempts leave a corrupted package cache

**Symptom:** even after fixing #2/#3, a freshly-"succeeding" `conda create` (using package caches built up across many earlier failed automatic retry attempts) produced an env where required annotation packages were silently absent, and a compiled package (`preprocessCore`, a `minfi` dependency) failed to `dyn.load` due to a missing shared library — see #5.

**Fix:** `conda clean -a -y` to clear the tarball/package cache, then rebuild the env from scratch. Galaxy/planemo's automatic dependency resolution retries a failing env creation many times (with slightly different arguments each time — pinned versions, then `=*`, then per-package) before giving up; each attempt can leave partially-extracted packages in the shared cache that get silently reused (skipping proper install steps) by later attempts, including manual ones.

## 5. Old pinned dependency set (`r-base=3.6.2` era) needs an explicit OpenBLAS pin

**Symptom:** after a clean rebuild, loading `minfi` still fails:
```
Error: package or namespace load failed for 'minfi' in dyn.load(...):
 unable to load shared object '.../lib/R/library/preprocessCore/libs/preprocessCore.dylib':
  dlopen(...): Library not loaded: @rpath/libopenblasp-r0.3.7.dylib
  Reason: tried: '.../lib/libopenblasp-r0.3.7.dylib' (no such file) ...
```

**Root cause:** `minfi_analysis.xml`'s pinned `bioconductor-preprocesscore` build (implied by the old `r-base=3.6.2`/`bioconductor-minfi=1.32.0` pins in `macros.xml`, not pinned itself) was compiled against a specific OpenBLAS ABI whose `.dylib` embeds the exact version in its install name (`libopenblasp-r0.3.7.dylib`). The solver picked a newer, unpinned `libopenblas` (0.3.12) instead, which ships under a different filename — the dynamic loader can't find the one `preprocessCore` actually links against.

**Fix (diagnostic environment only, not a tool-code change):** explicitly add `libopenblas=0.3.7` to the `conda create` command for this tool's env (confirmed available via `conda search -c conda-forge "libopenblas=0.3.7"`). After this, `library(minfi)` plus both annotation packages load cleanly and both of `minfi_analysis.xml`'s tests pass end-to-end.

**Implication for CI:** this is very likely osx-64/Rosetta-specific — Travis CI runs on Linux, where this old dependency set has presumably been resolving fine for years. Not expected to reproduce there.

## Net result

With all five of the above addressed, both `minfi_analysis.xml` tests and `ewas_harmonisation`'s Stage 1 and Stage 2 tests pass under genuinely isolated `--conda_dependency_resolution` execution (confirmed via `.libPaths()` pointing at the conda env, not host R). The `dmp`/`qcpng`/`qctab` float-precision and PNG-encoding tolerances added to the tool XMLs (see git history) proved robust across this — and the earlier, differently-drifted — environment, needing no further adjustment.
