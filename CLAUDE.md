# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This is a **Galaxy tool wrapper** (not a standalone application) that packages the Bioconductor `minfi` R package into a Galaxy-installable tool for analyzing Infinium Human Methylation BeadChip data. It is distributed via the Galaxy Tool Shed (`kpbioteam/ewastools`). There is no application source code to build — the repository *is* the tool definition consumed by the Galaxy/Planemo toolchain.

## Architecture

- `minfi_analysis.xml` — the Galaxy tool definition. This is the entire tool: it declares inputs/outputs, and embeds an R script (inside `<configfile>`) that Galaxy renders with parameter substitution and executes via `Rscript`. There is no separate `.R` file — editing the analysis logic means editing the R code embedded inside the XML's `<configfile>` block.
  - The `<command>` block symlinks each uploaded `.idat` file (red/green channel pairs) into the working directory using the Galaxy element identifier as filename, then runs the generated R script.
  - The embedded R pipeline: `read.metharray` → `preprocessRaw` → QC (`getQC`/`plotQC`) → `ratioConvert` → optional preprocessing branch selected by the `optpp` param (none / `preprocessFunnorm` / `preprocessQuantile` / SNP removal via `dropLociWithSnps`) → `bumphunter` for DMRs → `dmpFinder` for DMPs, merged against a UCSC genome table.
  - Galaxy parameter substitution uses `$paramname` inside the CDATA block; R's `$` accessor on data frames must be escaped as `\$` to avoid being interpreted as a Galaxy/Cheetah variable.
- `macros.xml` — shared Galaxy XML macros/tokens reused by `minfi_analysis.xml`: the `@VERSION@`/`@MINFI_VERSION@` tokens, the `<requirements>` block (r-base, bioconductor-minfi, and the 27k/450k/EPIC manifest+annotation packages, rtracklayer), and shared `<citations>`.
- `test-data/` — fixture files referenced by the `<tests>` block in `minfi_analysis.xml` (paired `.idat` red/green files from GEO accession GSM1588704-707, a `phenotypeTable.txt`, `ucsc.gtf` genome table, and expected outputs). Test data is compared by planemo against tool outputs (some outputs, like the QC plot PNG, use `compare='sim_size'` since exact byte comparison isn't meaningful for images).
- `.shed.yml` — Tool Shed metadata (category, owner, homepage, repo URL) used when publishing to the Galaxy Tool Shed.
- `.tt_skip` — list of directories `planemo ci_find_repos` should exclude from CI (currently empty, so nothing is skipped).

## Development commands

This repo is tested and linted via [Planemo](https://planemo.readthedocs.io/), the standard Galaxy tool development CLI, as run in `.travis.yml`.

```bash
pip install planemo

# Lint the tool (shed-level lint used in CI)
planemo shed_lint --tools --ensure_metadata --urls --report_level warn --fail_level error --recursive .

# Run the tool's declared <tests> against test-data/, installing conda dependencies as needed
planemo test --update_test_data --conda_dependency_resolution --conda_auto_install \
  --conda_channels iuc,conda-forge,bioconda,defaults .

# Serve the tool locally in a Galaxy instance for manual testing
planemo serve .
```

Python lint (flake8) is also run in CI (`flake8 --exclude=.git,./deprecated/ .`), though this repo has no Python source of its own — it applies to any helper scripts that may be added.

CI (`.travis.yml`) only lints/tests repositories or tools that changed in the commit range (via `planemo ci_find_repos`/`ci_find_tools`), chunked across 4 parallel jobs. On push to `master` it publishes to both the Test Tool Shed and the main Tool Shed via `planemo shed_update`.

## Editing the tool

- Bump `version` in `minfi_analysis.xml`'s `<tool>` tag whenever the embedded R script or params change, per Galaxy Tool Shed conventions.
- Keep `macros.xml`'s `@MINFI_VERSION@` token in sync with the `bioconductor-minfi` requirement version if the minfi dependency is upgraded.
- When adding/renaming Galaxy params, update the matching `<test>` block's `<param>`/`<output>` entries so `planemo test` stays valid.
