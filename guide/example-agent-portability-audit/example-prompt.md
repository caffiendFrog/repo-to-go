You are auditing a Nextflow repository (https://github.com/CCBR/CHAMPAGNE) for portability. Analyze the repository
and produce a report only; do not edit files, install dependencies, delete
files, commit changes, or access real research data.

Repository context
- Pipeline purpose: CHromAtin iMmuno PrecipitAtion sequencinG aNalysis pipEline
- Required targets: local Linux with Docker, macOS and PC
- Optional targets: None
- Known site environment: None
- Constraints: Offsite execution

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