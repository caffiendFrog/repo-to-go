# Portability Audit — CCBR/CHAMPAGNE

**Audit target:** commit `420f1bb` (branch `main`, 2026-07-18), working tree clean
**Method:** static inspection of the repository. No pipeline was executed — the audit
sandbox had no Nextflow or container runtime available and `get.nextflow.io` was outside
the network allowlist. Checks that require a live run are marked *skipped* and listed under
[Unknowns](#unknowns).
**Requested targets:** local Linux with Docker, macOS, and PC. **Constraint:** offsite execution.

> **Legend for findings:** P0 = prevents execution or risks invalid results · P1 = major
> undocumented setup or reproducibility problem · P2 = maintenance/usability · P3 = optional hardening.
> Line references are to the audited commit and may shift as the code changes.

---

## Executive summary

**Overall rating: partially portable.**

The pipeline is well-structured for portability in principle — every real process declares a
container, nf-core/CCBR modules are git-SHA pinned, reference genomes fall back to
user-supplied FASTA/GTF, scripts use `#!/usr/bin/env` shebangs with no CRLF or GNU-only
commands — but the **documented offsite entry path does not enable a container engine**, so a
naive offsite run fails, and the **required Nextflow version is documented only in prose**, not
enforced by the manifest.

**Environments evidenced as working:** none verified in this audit. CI runs the **stub** graph on
`ubuntu-latest` (Nextflow 25.10.0) on every push; the **containerized** test runs only on manual
dispatch (`test_run=true`), so even Linux + Docker is exercised intermittently, not continuously
proven. Biowulf is the only environment the project itself claims to support operationally.

**Three highest-value next actions**

1. Make the offsite `--mode local` path enable a container engine (add `docker`/`singularity` to
   the profile, or document it in the README quickstart). Local runs currently execute with no
   engine and no software on `PATH`. **(P0-1)**
2. Set `manifest.nextflowVersion = '>=25.10.0'` so the new output-DSL syntax fails with a clear
   message instead of a cryptic parse error. **(P0-2)**
3. Fix the syntax error in `conf/frce.config` (unbalanced brace) and pin the mutable `:latest`
   cutadapt tag. **(P0-3 / P1-1)**

---

## Support matrix

Container runtime and executor are independent: any executor can pair with docker or singularity,
but **neither is enabled by default** in the portable base config.

| Target | Overall | Container runtime | Executor | Evidence | Verification still needed |
|---|---|---|---|---|---|
| **Local Linux + Docker** | likely | Docker: likely (only with explicit `-profile docker`) | local | `nextflow.config:44-51` defines a valid `docker` profile; CI test step uses `-profile docker` but only on manual dispatch (`build.yml:56-61`) | `champagne run -profile docker -c tests/nxf/ci_test.config --mode local` on a clean Linux+Docker host; confirm images pull and commands succeed |
| **macOS** | unknown / blocked | Docker: likely; CLI wrapper is blocked | local | `bin/champagne:3` uses `realpath` (absent by default on macOS); `docker.runOptions='-u $(id -u):$(id -g)'` (`nextflow.config:50`) behaves differently under Docker Desktop | Test `nextflow run CCBR/CHAMPAGNE -profile test,docker` directly (bypassing the CLI) on macOS; test the `champagne` CLI separately |
| **PC (Windows)** | unknown | Docker via WSL2: unknown | local | No Windows/WSL2 handling anywhere; Bash + `id -u` + `realpath` imply WSL2 is the only realistic path | Run under WSL2 Ubuntu with Docker Desktop; native Windows is not viable for a Bash/Nextflow pipeline |
| Biowulf *(site, not requested)* | likely | Singularity | slurm | `conf/biowulf.config`, README; auto-selected by CLI | Out of audit scope |

---

## Repository map

| Concern | Files |
|---|---|
| **Workflow entry points** | `main.nf` (main workflow + `DOWNLOAD_SRA`, `MAKE_REFERENCE`, `debug`, `version`, `LOG`); CLI `bin/champagne` → `main.py` → `src/__main__.py` (Click app; depends on external `ccbr_tools`) |
| **Configuration layers** | `nextflow.config` (root: profiles, plugins, env, manifest) → `conf/base.config` (resources/labels) → `conf/{genomes,containers,modules}.config` (always included) → site/test profile configs in `conf/` |
| **Dependencies** | `pyproject.toml` (`ccbr_tools@git+…@v0.4`, Click, pyyaml); `modules.json` (git-SHA-pinned modules) |
| **Containers** | `conf/containers.config` (param-based) + hardcoded strings in some `modules/local/*.nf` and `modules/nf-core/rmarkdownnotebook/main.nf` |
| **Reference data** | `conf/genomes.config` (prebuilt, under `params.index_dir`); custom-genome via `params.genome_fasta`/`genes_gtf`; motif assets in `assets/` |
| **Tests** | `tests/nxf/` (`ci_stub.config`, `ci_test.config`, blank refs in `tests/data/`); `tests/test_cli.py` |
| **CI** | `.github/workflows/build.yml` (stub always, container test on dispatch), `docs-mkdocs.yml`, `check-links.yml`, others |
| **User setup** | `README.md`, `docs/nextflow.md`, `docs/guide/`, `nextflow_schema.json` |

---

## Findings

### P0 — prevents execution or risks invalid results

#### P0-1. Offsite `--mode local` runs enable no container engine
- **Targets:** Linux / macOS / PC (all offsite).
- **Evidence:** `src/__main__.py:139-149` calls `ccbr_tools.pipeline.nextflow.run(mode=_mode…)`.
  In that function, profiles are only augmented with `slurm`/`hpc.name`; when no HPC is detected
  offsite, the profile set is empty and the command is plain `nextflow run main.nf -resume`
  (ccbr_tools `pipeline/nextflow.py::run`; `hpc.py::get_hpcname` returns empty offsite).
  `conf/base.config` states "Assumes that all software is installed and available on the PATH."
  README quickstart (`README.md:51-56`) shows `--mode local -profile test` with no engine.
- **Why it's a portability issue:** with no engine enabled and no bioinformatics tools on the host
  `PATH`, every real process fails (`fastqc: command not found`, etc.). Functional blocker, not style.
- **Remediation:** either (a) document `-profile docker` (or `singularity`) in the README quickstart
  alongside `--mode local`, or (b) have the CLI append a container profile for local mode.
  `docs/nextflow.md:15` already does this correctly (`-profile test,singularity`) — the README and
  CLI are the inconsistent parts.
- **Verification:** on a host with Docker but no bioinformatics tools, `champagne run -profile test
  --mode local` should fail; `champagne run -profile test,docker --mode local` should proceed.
- **Confidence:** high. **Open question:** does an internal `ccbr_tools` default add an engine not
  visible in v0.4 source? (Inspected v0.4.6 — it does not.)

#### P0-2. Required Nextflow version is not enforced
- **Targets:** all.
- **Evidence:** the `manifest` block (`nextflow.config:181-190`) sets no `nextflowVersion`. The
  pipeline uses the new output DSL: top-level `outputDir` (`nextflow.config:1`),
  `workflow.output.mode` (`:200`), and `publish:` / `output {}` blocks (`main.nf:309,332`), which
  require Nextflow ≥ 24.10 / 25.x. Version is stated only in prose (`docs/nextflow.md:10-13`).
- **Why:** a user on older Nextflow gets an opaque syntax error rather than a clear guard;
  reproducibility depends on an unpinned toolchain.
- **Remediation:** add `nextflowVersion = '>=25.10.0'` to `manifest`.
- **Verification:** `NXF_VER=23.10.0 nextflow run . -profile test -preview` should fail with a
  version message after the fix.
- **Confidence:** high.

#### P0-3. `conf/frce.config` has an unbalanced brace (parse error)
- **Targets:** frce (site, not a required target) — but any run composing this profile fails.
- **Evidence:** `conf/frce.config:29` — a stray `}` closes a block that was never opened
  (brace count 3 open / 4 close).
- **Why:** a malformed config aborts config parsing.
- **Remediation:** delete the stray `}` on line 29.
- **Verification:** `nextflow config -profile frce` parses without error.
- **Confidence:** high. *(frce is out of the required target set — high severity for that profile,
  low overall priority.)*

### P1 — major undocumented setup / reproducibility problem

#### P1-1. Mutable `:latest` container tag
- **Targets:** all.
- **Evidence:** `modules/CCBR/cutadapt/main.nf:5` — `container 'nciccbr/ncigb_cutadapt_v1.18:latest'`.
- **Why:** `latest` is mutable; two runs can pull different images, breaking reproducibility and
  resume/cache guarantees.
- **Remediation:** pin to an immutable tag or digest (`…@sha256:…`).
- **Verification:** `docker pull` the digest; grep for `:latest` returns nothing.
- **Confidence:** high.

#### P1-2. README quickstart omits the container engine (docs drift)
- **Targets:** all offsite.
- **Evidence:** `README.md:51-56` vs the correct `docs/nextflow.md:15`.
- **Why:** new users follow the README; the shown command cannot work offsite (see P0-1).
- **Remediation:** align README with docs.
- **Verification:** doc-lint / manual.
- **Confidence:** high.

#### P1-3. Test/CI reference data pulled from mutable branch URLs
- **Targets:** CI + anyone running `-profile test` / `ci_test`.
- **Evidence:** `conf/test.config` and `tests/nxf/ci_test.config` fetch
  `https://raw.githubusercontent.com/nf-core/test-datasets/atacseq/reference/genome.fa`
  (and `genes.gtf`) — a moving branch ref, not a pinned commit. Network required (no offline runs).
  No checksums.
- **Why:** live mutable URLs make tests non-hermetic and can change silently; conflicts with
  offline-behavior portability.
- **Remediation:** pin to a commit SHA in the URL and record a checksum, or vendor a tiny reference
  into `tests/data/`.
- **Verification:** re-run test with network blocked after vendoring; confirm checksum.
- **Confidence:** high.

#### P1-4. CI does not continuously exercise containers or bundled scripts
- **Targets:** all.
- **Evidence:** `build.yml:50-61` — the always-on step is `champagne run -stub …`; the real
  `-profile docker` run is gated on `workflow_dispatch` with `test_run=true`.
- **Why:** a stub run proves graph wiring only; it does not prove containers pull, commands run, or
  `bin/*.py|*.R` work. Regressions in real execution can merge undetected.
- **Remediation:** run at least one minimal containerized test on PRs (a single-sample subset to fit
  runner limits).
- **Verification:** add the job; confirm it pulls images and completes.
- **Confidence:** high.

### P2 — maintenance / usability

#### P2-1. `publish_dir_mode = 'link'` default is fragile across filesystems
- **Targets:** macOS / Docker Desktop, offsite multi-mount setups.
- **Evidence:** `nextflow.config:27` (also `biowulf.config:18`). Hardlinks fail when `work/` and
  `outputDir` are on different filesystems (common with Docker Desktop bind mounts). Test configs
  override to `symlink` (`ci_stub.config`, `ci_test.config`).
- **Remediation:** default to `copy` or `symlink` for portable use; keep `link` in the biowulf
  profile only.
- **Verification:** run with `outputDir` on a different mount than `work/`.
- **Confidence:** medium (depends on the user's mount layout).

#### P2-2. `champagne` CLI uses `realpath` (absent on stock macOS)
- **Targets:** macOS.
- **Evidence:** `bin/champagne:3`.
- **Why:** BSD/macOS lacks GNU `realpath` unless `coreutils` is installed; the CLI wrapper fails at
  line 3. (Direct `nextflow run CCBR/CHAMPAGNE` avoids this.)
- **Remediation:** use a portable shell idiom or Python for path resolution, or document
  `brew install coreutils`.
- **Verification:** run `bin/champagne --help` on macOS without coreutils.
- **Confidence:** medium.

#### P2-3. Hardcoded container strings bypass `containers.config`
- **Targets:** all (maintenance/reproducibility).
- **Evidence:** `conf/containers.config` centralizes images, yet several modules hardcode strings,
  including different base-image versions (`nciccbr/ccbr_ubuntu_base_20.04:v5`, `:v6`, `:v6.1`
  across `modules/`; `modules/local/qc.nf:8`).
- **Why:** divergent versions of the "same" base image and two sources of truth complicate updates
  and auditing.
- **Remediation:** route all images through `params.containers_*`; reconcile base-image versions.
- **Verification:** grep modules for literal `nciccbr/` outside params.
- **Confidence:** medium.

#### P2-4. Human-chromosome default for `deeptools_excluded_chroms`
- **Targets:** non-human genomes run without an override.
- **Evidence:** `nextflow.config:41` — default `"chrM chrX chrY"`. Test config correctly overrides
  to `chrM`.
- **Why:** silent no-op (not a crash) on genomes lacking those contig names; a correctness footgun.
- **Remediation:** document clearly and/or derive per-genome defaults.
- **Confidence:** medium.

### P3 — optional hardening

- **P3-1.** `docs/nextflow.md:12` shows `nxf_ver=25.10.0 nextflow …`; the Nextflow env var is
  `NXF_VER` (uppercase). Minor doc bug.
- **P3-2.** `containers_*` images give only a Docker Hub path (no `quay.io`/Galaxy depot alternative
  like the nf-core modules'). Works under Singularity via Docker Hub, but offers no mirror fallback.
  Consider documenting.
- **P3-3.** `nextflow_schema.json` lacks a `publish_dir_mode` enum (`platform_options` has no such
  property), so typos aren't caught by schema validation. Add an enum.
- **P3-4.** Commented-out/dead process bodies (`PLOT_NGSQC` inside a `/* */` block,
  `modules/local/qc.nf:241`) — harmless but confusing.

---

## Hidden dependencies

- **Host commands:** `nextflow` (≥25.10.0, enforced only in prose — see P0-2), `java` 21 (present),
  a container engine (**Docker or Singularity — required but not enabled by the local CLI path**,
  P0-1), `bash`, `id`, and `realpath` for the CLI (`bin/champagne:3`, macOS gap).
- **Python package:** `ccbr_tools@git+https://github.com/CCBR/Tools@v0.4` (`pyproject.toml`) — an
  external CCBR git dependency that governs mode→profile mapping and HPC detection; documented only
  implicitly.
- **Environment variables:** `PYTHONNOUSERSITE`, `R_PROFILE_USER`, `R_ENVIRON_USER`,
  `JULIA_DEPOT_PATH` set in `env{}` (`nextflow.config:167-172`) — assume container-internal paths;
  documented in a code comment.
- **Shared filesystems:** `params.index_dir`, `fastq_screen_db_dir` (biowulf:
  `/data/CCBR_Pipeliner/…`; frce: TODO/null). Correctly isolated to site profiles; offsite users
  supply custom-genome params — documented in README ("custom reference genome").
- **Registries:** Docker Hub (`nciccbr/*`), `quay.io/biocontainers` (nf-core modules). No
  credentials required (public).
- **Network endpoints:** `raw.githubusercontent.com` (test refs, mutable), `github.com`
  (module/CCBR-Tools resolution), Docker Hub / quay. Offline runs not supported.
- **Licenses:** MIT (`LICENSE`); bundled JASPAR/HOCOMOCO motif assets and their upstream licenses
  noted in config comments.

---

## Reproducibility inventory

| Component | Pinned? | Evidence |
|---|---|---|
| Nextflow | Prose only, not enforced | `docs/nextflow.md:10`; **no** `manifest.nextflowVersion` |
| Java | Not pinned | runtime-provided (21 in sandbox) |
| Plugins | **Pinned** | `nf-schema@2.7.1`, `nf-prov@1.4.0` (`nextflow.config:174-177`) |
| nf-core / CCBR modules | **Pinned (git_sha)** | `modules.json` |
| ccbr_tools | **Pinned (git tag `v0.4`)** | `pyproject.toml` |
| Local containers | Tag-pinned except one | `conf/containers.config`; **`:latest`** at `modules/CCBR/cutadapt/main.nf:5` |
| Container digests | **Not pinned** | no `@sha256:` anywhere |
| Python deps | Range-pinned | `pyproject.toml` (`pyyaml>=6.0`, `Click>=8.1.3`) |
| Remote test assets | **Unpinned (branch URL), no checksum** | `conf/test.config`, `tests/nxf/ci_test.config` |
| Reference genomes | Site paths; no checksums | `conf/genomes.config` |

---

## Test gaps and proposed portability matrix

- **Smallest self-contained stub/graph test (exists):** `champagne run -stub -c ci_stub.config
  --mode local` with blank refs in `tests/data/` (`build.yml:50-54`). Proves DAG wiring only —
  not execution.
- **Minimal real containerized test (partly exists, under-used):** `-profile docker -c
  ci_test.config` currently runs only on manual dispatch. Promote a *reduced* version (one ChIP +
  one input sample, 1–2 peak callers) to run on every PR so containers and `bin/*` scripts are
  exercised within runner limits. Vendor/pin its reference data (P1-3).
- **Executor/profile tests (missing):** a `nextflow config`-only lint that composes each profile
  (`docker`, `singularity`, `test,docker`, `biowulf,slurm`) and asserts it parses — this alone would
  have caught P0-3 (frce brace). Add `nextflow config -profile <p>` to CI as a fast, engine-free gate.
- **macOS / WSL2 (missing):** at minimum a macOS runner doing `nextflow run . -profile test,docker
  -preview` to catch CLI / `runOptions` / `realpath` issues without a full run.

---

## Remediation sequence

**Phase 1 — repository-only, no result impact (safe, high value):** add `manifest.nextflowVersion`
(P0-2); fix `conf/frce.config` brace (P0-3); align README with docs to include a container profile
(P0-1 doc side / P1-2); pin the cutadapt tag (P1-1); add a `nextflow config -profile …` parse-lint
to CI. None affect resume/cache or scientific output (a tag change to an equivalent image should not,
but verify the digest matches).

**Phase 2 — repository-only, test hardening:** vendor/pin test reference data + checksums (P1-3);
promote a minimal containerized run to PR CI (P1-4). Changes CI runtime, not results.

**Phase 3 — behavior-affecting, review carefully:** change the default `publish_dir_mode` from
`link` (P2-1); route hardcoded containers through params / reconcile base-image versions (P2-3).
Changing a container version **can alter scientific results** and will invalidate resume/cache for
affected processes — do these deliberately, one image at a time, with test comparison.

**Phase 4 — infrastructure-specific (kept separate):** fill in the frce
`index_dir`/`fastq_screen_db_dir`/`scratch` TODOs; any new offsite site profile. These belong in
profile configs, not workflow defaults.

---

## Unknowns

Facts that cannot be established from the repository alone. Questions are posed rather than guessed.

1. Does a live `-profile docker` run on clean Linux actually pull all `nciccbr/*` images and
   complete? *Not testable in this audit* — no Docker in the sandbox and `get.nextflow.io` was
   outside the network allowlist. **(Skipped; recorded.)**
2. Are the `nciccbr/*` Docker Hub images published for `linux/arm64` (Apple Silicon)? — *Have the
   images been built multi-arch, or must macOS users force `--platform linux/amd64`?*
3. Does `ccbr_tools` v0.4 ever add a container profile for local mode through a path not visible in
   the source inspected (v0.4.6)? *Confirm against the exact resolved commit.*
4. What is the intended offsite reference-genome workflow end to end — is `MAKE_REFERENCE` expected
   to run once and its emitted `custom_genome.config` reused? The offsite caching/checksum story is
   unspecified.
5. Is native Windows (non-WSL2) actually in scope for "PC," or is WSL2 the assumed path? A
   Bash + Nextflow pipeline cannot run on native Windows; confirming this bounds the support matrix.

---

*Observed facts are cited to files/lines; inferences and recommendations are labelled as such.
Site-specific profiles (`biowulf`, `frce`, `slurm`) are treated as valid optimizations, not defects,
where portable defaults and documented alternatives exist.*