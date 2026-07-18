# Developing portable Nextflow pipelines with coding agents

Portability means that a pipeline can be checked out and run in a new
environment without relying on undocumented software, paths, data, or scheduler
settings from its original development site. It does not mean that every
environment uses the same configuration. A portable pipeline keeps workflow
logic and defaults environment-neutral, then adds explicit profiles for local
computers, container runtimes, HPC systems, and other infrastructure.

This guide has two parts:

1. a copy-and-paste prompt that asks a coding agent to audit an existing
   Nextflow repository; and
2. a checklist to use while developing a pipeline so that fewer portability
   problems need to be discovered later.

The audit is intentionally read-only. Review its evidence and priorities before
asking an agent to implement changes.

## Before using an agent

- Use an AI service approved for the repository and organization.
- Do not send protected health information, credentials, private sample
  identifiers, or other sensitive data to the agent.
- Replace production data with small synthetic fixtures.
- Commit or otherwise preserve current work before requesting large changes.
- Give the agent access to the entire repository, not only `main.nf`.
- Tell the agent which environments must be supported. Portability has no useful
  definition without a target, such as local Linux, WSL2, institutional Slurm,
  AWS Batch, Docker, or Apptainer.

## Initial repository audit prompt

Replace the values in angle brackets, then give the entire prompt to a coding
agent from the repository root.

```text
You are auditing a Nextflow repository for portability. Analyze the repository
and produce a report only; do not edit files, install dependencies, delete
files, commit changes, or access real research data.

Repository context
- Pipeline purpose: <short description or "infer from the repository">
- Required targets: <for example: local Linux with Docker; WSL2 with Apptainer;
  institutional Slurm with Apptainer>
- Optional targets: <for example: macOS development; AWS Batch>
- Known site environment: <scheduler, shared paths, modules, registries, proxies>
- Constraints: <security, offline execution, licenses, supported CPU architecture>

Method
1. Read the repository instructions and current git status first. Preserve and
   distinguish any uncommitted work.
2. Map the entry points, workflows, subworkflows, modules, scripts, configs,
   profiles, parameter schema, containers, tests, CI, and user documentation.
3. Trace one representative input through staging, execution, publication, and
   provenance. Also trace reference-data acquisition and caching.
4. Search for portability assumptions, including:
   - absolute or site-specific paths, environment variables, host tools,
     module commands, scheduler directives, filesystem behavior, and network
     access;
   - unpinned Nextflow, Java, plugins, packages, containers, remote files, and
     reference data;
   - processes without containers and containers with ambiguous registries,
     mutable tags, unavailable architectures, credentials, or incompatible
     Docker/Apptainer behavior;
   - config syntax tied to a Nextflow version, profile-order dependencies, and
     infrastructure settings mixed into workflow defaults;
   - commands that assume Bash, GNU utilities, Linux-only features, writable
     home directories, CRLF line endings, or host-installed Python/R libraries;
   - resource requests that exceed small machines, executor settings embedded
     in processes, non-portable publish modes, and cache/temp-space assumptions;
   - tests that use institutional storage, production data, live mutable URLs,
     secrets, large downloads, or stub blocks that do not represent real
     command behavior;
   - missing input validation, undocumented parameters, inconsistent defaults,
     and output paths based on the caller's working directory by accident.
5. Run only safe, read-only or disposable validation commands already supported
   by the repository when practical. Do not claim that a target works unless it
   was tested. Record skipped checks and why.
6. Treat site profiles as valid optimizations when a portable fallback exists.
   Do not report every absolute path in an explicitly site-specific profile as a
   blocker; report hidden coupling to that profile.

Assess at least these portability dimensions
- bootstrap and version compatibility;
- Linux, WSL2, and requested operating systems;
- Docker and/or Apptainer image resolution and execution;
- executors, resource scaling, and filesystem behavior;
- inputs, outputs, samplesheets, and path staging;
- reference data, checksums, caching, and offline behavior;
- parameters, schemas, defaults, and profile composition;
- shell commands and bundled Python/R/other scripts;
- reproducibility, provenance, security, and licensing;
- self-contained tests, minimal real runs, CI, and documentation;
- instructions and guardrails that would help future coding agents.

Required report

# Portability audit

## Executive summary
- Give an overall rating: portable, portable with configuration, partially
  portable, or site-bound.
- Name the environments that are evidenced as working, not merely intended.
- List the three highest-value next actions.

## Support matrix
For every requested target, give a status of verified, likely, blocked, or
unknown. State the evidence and the exact verification still needed. Keep
container runtime and executor as separate columns or fields.

## Repository map
Identify the files that define workflow entry points, configuration layers,
dependencies, containers, reference data, tests, CI, and user setup.

## Findings
Group findings as:
- P0: prevents execution or risks invalid results;
- P1: major undocumented setup or reproducibility problem;
- P2: important maintenance or usability problem;
- P3: optional hardening.

For each finding include:
- a short title;
- affected target environments;
- exact file and line evidence;
- why it is a portability issue rather than a style preference;
- a concrete remediation;
- a focused verification command or test;
- confidence and any unanswered question.

## Hidden dependencies
List required host commands, environment variables, shared filesystems,
registries, credentials, network endpoints, licenses, and manual setup. Say
where each dependency is documented or note that it is undocumented.

## Reproducibility inventory
Inventory the pinned and unpinned versions/checksums for Nextflow, Java,
plugins, containers, packages, remote assets, and reference data.

## Test gaps and proposed portability matrix
Describe the smallest self-contained stub/graph test, minimal real
containerized test, and executor/profile tests needed. A stub run alone is not
proof that commands, containers, or bundled scripts work.

## Remediation sequence
Propose small, reviewable phases. Separate repository changes from
infrastructure-specific configuration. Call out changes that may alter
scientific results or invalidate Nextflow resume/cache behavior.

## Unknowns
List facts that cannot be established from the repository. Ask focused
questions instead of guessing.

Quality rules
- Cite files and line numbers for every repository-specific claim.
- Separate observed facts, inferences, and recommendations.
- Do not label intentional site profiles as defects when neutral defaults and
  documented alternatives exist.
- Do not expose secrets or reproduce sensitive file contents in the report.
- Prefer the repository's existing conventions and nf-core patterns.
- Do not recommend a rewrite when an isolated config, test, schema, container,
  or documentation change would solve the problem.
```

## How to review the report

An agent's report is a starting point, not proof. Before implementation:

- Confirm the required support matrix with maintainers.
- Reproduce every P0 and P1 finding or inspect its cited evidence.
- Separate scientific behavior changes from packaging and infrastructure work.
- Decide which remote assets may be redistributed and which require a license
  or user-provided path.
- Turn each accepted finding into a small issue with an observable acceptance
  test.
- Ask for implementation one phase at a time and require tests before moving to
  the next phase.

## Portability-first development checklist

### Scope and architecture

- [ ] Required operating systems, container runtimes, executors, CPU
      architectures, network constraints, and filesystems are documented.
- [ ] `main.nf` and included workflows contain scientific orchestration, not
      site setup or scheduler-specific commands.
- [ ] DSL2 modules and subworkflows have explicit typed inputs and outputs where
      the supported Nextflow version permits them.
- [ ] Reusable nf-core or organizational modules are preferred over copied
      commands, and generated/vendor modules are not edited without a documented
      reason.
- [ ] Optional branches are independently testable and fail with actionable
      messages when their prerequisites are unavailable.
- [ ] Channels for optional files are created only after checking that the
      parameter is set; helpers such as `Channel.fromPath` never receive `null`.

### Versions and host bootstrap

- [ ] A supported Nextflow version or range is enforced in
      `manifest.nextflowVersion`.
- [ ] The documented installation method resolves to the same Nextflow range;
      it does not silently install an incompatible newest release.
- [ ] Java and host-side helper requirements are documented and machine
      installable, for example with an environment file or setup script.
- [ ] Nextflow plugins and host packages are version-pinned.
- [ ] Compatibility with the newest intended Nextflow parser and runtime has
      been tested before widening the supported version range.
- [ ] Host utility assumptions such as Bash, GNU `date`, `readlink`, `sort`, or
      `awk` are either portable, packaged, or explicitly documented.

### Parameters and configuration

- [ ] Environment-neutral defaults live in the base config.
- [ ] Executor, queue, account, scratch, proxy, module, and shared-path settings
      live in named site profiles.
- [ ] Local, Docker, and Apptainer profiles can be composed without relying on
      an undocumented profile order.
- [ ] Every user-facing parameter has one canonical name, type, default, and
      description in both config and the parameter schema.
- [ ] Required parameters are validated before expensive work starts.
- [ ] Parameters referenced by the schema and documentation are exercised by
      workflow code; renamed or inert parameters cannot silently drift.
- [ ] Optional parameters have safe defaults such as `null`; absent
      site-specific values do not trigger undefined-parameter warnings.
- [ ] Repository assets use `projectDir`; user-relative inputs are resolved
      intentionally from `launchDir` or converted to staged `path` inputs.
- [ ] Output directories and publication modes behave correctly across local,
      network, object, and Windows-mounted filesystems.
- [ ] Resource labels are centralized and capped by configurable maximum CPU,
      memory, and time values.
- [ ] Scheduler-specific retry and queue behavior is isolated from scientific
      process definitions.

### Containers and commands

- [ ] Every real process has a container, including validation and report
      scripts that are easy to overlook.
- [ ] Images use immutable versions; release pipelines prefer digest pins where
      the registry and runtime support them.
- [ ] No production process uses `latest` or another floating image tag.
- [ ] Registry names are fully qualified when ambiguity exists.
- [ ] Mixed Docker Hub, Quay, Galaxy, private, and institutional registries are
      tested with every supported container runtime.
- [ ] Private images have a documented authentication-free fallback or explicit
      credential setup; credentials never appear in code or config.
- [ ] Images support the required CPU architectures and do not require root,
      a writable home directory, or an undeclared bind mount.
- [ ] Process scripts use strict shell behavior where appropriate and quote
      paths that can contain spaces or shell metacharacters.
- [ ] Pipes preserve upstream failures, and intentional non-zero exit statuses
      are handled explicitly.
- [ ] Python, R, Perl, and other helper scripts run inside the process container,
      not against accidental user-site libraries.
- [ ] Each module emits tool versions and has a realistic `stub` block when
      whole-pipeline stubs depend on it.
- [ ] Module `versions.yml` outputs are aggregated and published as a software
      inventory rather than left disconnected.

### Inputs, references, and outputs

- [ ] A schema validates samplesheet columns, types, uniqueness, file existence,
      read pairing, and incompatible options before execution.
- [ ] Samplesheet and parameter paths are tested with spaces and their support
      or rejection of URLs and object-store paths is explicit.
- [ ] Small synthetic inputs are committed or fetched from immutable,
      checksum-verified locations.
- [ ] No test or example contains patient identifiers or internal storage paths.
- [ ] Reference resources can be supplied by users without editing repository
      files.
- [ ] A site cache may provide prebuilt references, but a documented portable
      path can fetch/build them or clearly explains why redistribution is not
      possible.
- [ ] Remote URLs are versioned and checksum-verified; decompression and index
      construction are workflow processes that Nextflow can cache.
- [ ] Reference builds record source versions, checksums, parameters, tool
      versions, and contig naming conventions.
- [ ] Cache and temporary directories are configurable and sized for image
      conversion, downloads, sorting, and reference builds.
- [ ] Outputs are published without silently depending on symlink support.
- [ ] Reports, trace, timeline, DAG, parameters, software versions, and
      provenance are retained with results.

### Operating systems and filesystems

- [ ] Executable text files are committed with LF endings, enforced with
      `.gitattributes`.
- [ ] Linux is stated as the execution platform; Windows instructions use WSL2
      when commands and containers are Linux-only.
- [ ] WSL instructions distinguish PowerShell commands from Linux commands.
- [ ] WSL tests cover repositories, work directories, container caches, and
      build temp space on both native Linux storage and mounted Windows drives.
- [ ] The pipeline does not assume symlinks, executable bits, case-sensitive
      paths, file locking, or atomic renames without documenting filesystem
      requirements.
- [ ] Apptainer bind paths, cache variables, temp variables, and registry
      behavior are documented without leaking host temp settings into tasks.
- [ ] macOS is described as a launch host only if Linux containers or a remote
      executor provide the actual task environment.

### Testing and continuous integration

- [ ] A fast config/graph or stub test is self-contained and runs without
      institutional paths, databases, containers, or network access.
- [ ] Stub tests do not accidentally execute uncontainerized validation or
      reporting scripts.
- [ ] A minimal real test runs actual commands in each supported container
      runtime on synthetic data.
- [ ] Modules and subworkflows have focused `nf-test` coverage, including
      failure cases and metadata/channel shape.
- [ ] Reference download/build and prebuilt-reference branches are both tested.
- [ ] Local and required scheduler profiles are tested at an appropriate
      cadence.
- [ ] Every shipped profile receives at least a config parse or preview test,
      including profiles that are not used by routine CI.
- [ ] CI enforces the supported Nextflow version instead of implicitly testing
      whichever version is newest that day.
- [ ] CI validates the parameter schema, formatting, lint, unit tests,
      documentation build, and expected output inventory.
- [ ] CI actions and test dependencies are version-pinned and use least
      privilege.
- [ ] Test logs and `.nextflow.log` are retained on failure without uploading
      sensitive data.
- [ ] Automated runs use isolated launch/work directories so concurrent
      Nextflow sessions cannot collide.
- [ ] Resume behavior is tested for changes to publication, references, and
      optional branches where cache correctness matters.

### Documentation and agent readiness

- [ ] The README has one copy-and-paste quickstart for every supported class of
      environment.
- [ ] Setup docs list Nextflow, Java, container runtime, storage, network, and
      reference requirements with verification commands.
- [ ] Examples use synthetic data, explicit profiles, and versioned pipeline
      releases.
- [ ] Expected runtime, memory, CPU, disk, image-cache, and reference-build costs
      are stated for test and representative real runs.
- [ ] Troubleshooting maps exact symptoms to causes and verified fixes.
- [ ] Repository instructions tell coding agents which files are generated,
      which commands verify changes, which data is sensitive, and which
      environments are authoritative.
- [ ] Development setup executes the checked-out source, for example through an
      editable package install, rather than a stale globally installed wrapper.
- [ ] Agent instructions prohibit secrets/PHI, preserve uncommitted work, and
      require evidence before claiming an environment is supported.
- [ ] User-facing changes update the schema, docs, examples, tests, changelog,
      and provenance together.
- [ ] A human reviews generated Nextflow and shell code for scientific intent,
      quoting, channel semantics, resource behavior, and test validity.
- [ ] Optional completion hooks, usage logging, and report helpers cannot hide
      the pipeline's primary failure when a telemetry dependency is absent.

### Release readiness

- [ ] The portability support matrix is reviewed for the release.
- [ ] Clean-clone setup and the documented quickstarts have been tested.
- [ ] Container images and remote references remain accessible from every
      supported environment.
- [ ] The release tag, Nextflow range, schema, docs, and changelog agree.
- [ ] Known unsupported targets and site-only features are stated explicitly.
- [ ] A release records enough provenance to reproduce both software and
      reference inputs.

## Lessons illustrated by CHAMPAGNE

The work required to run CHAMPAGNE away from its original Biowulf environment
provides several transferable examples:

- **Pin the runtime before debugging the workflow.** Nextflow 26 enables a
  stricter config parser that rejects syntax accepted by CHAMPAGNE's supported
  25.10 line. The manifest, host environment, CI, and documentation must agree
  on the supported range.
- **A containerized pipeline still has host dependencies.** Nextflow, Java, the
  container runtime, shell utilities, filesystem behavior, and helper CLIs all
  run or interact with the host. On newer Ubuntu/WSL installations, a
  non-GNU-compatible `date` implementation was enough to break Nextflow's task
  wrapper.
- **Image names are runtime-dependent.** Bare BioContainers and organizational
  image names can resolve differently under Docker and
  Singularity/Apptainer. Mixed registries require explicit names or a tested
  runtime profile.
- **Shared references are an optimization, not a default.** Biowulf can use
  prebuilt indexes, while portable runs need versioned public sources,
  checksums, workflow-managed builds, and an optional persistent cache.
- **Resource limits are part of portability.** Process labels may request
  production-scale memory. Configurable resource caps allow the same graph to
  run small tests on laptops and CI without rewriting process definitions.
- **A stub is necessary but insufficient.** Stubs expose graph and staging
  errors, but a validation script without a stub or a bad container can remain
  hidden. Keep an offline stub test and a minimal real containerized test.
- **Filesystem details are operational dependencies.** LF line endings,
  symlink support, executable bits, bind mounts, image caches, and temporary
  space can determine whether an otherwise correct workflow runs under WSL or
  on a shared filesystem.

These examples should guide what an audit looks for, not become assumptions
about another repository. Every finding still needs evidence from that
repository and validation in its required environments.
