# repo-to-go

Reusable GitHub Actions and workflows for making bioinformatics repositories
portable, reproducible, and ready for production use.

## About

This repository was created to turn lessons from the portable [CHAMPAGNE](https://github.com/CCBR/CHAMPAGNE) work
into shared automation. Rather than copying CI and release logic between
projects, other repositories will be able to consume the actions and reusable
workflows maintained here.

Initial development took place during CoFest 2026, hosted by the Bioinformatics
Open Source Conference (BOSC) organization.


## Project status

This repository is under development. Reusable actions and workflows will be
documented here as they become available.

## Intended scope

Utilities in this repository may support:

- validation of portable repositories;
- reproducibility and dependency checks;
- testing across supported execution environments and container runtimes;
- release and provenance automation; and
- consistent CI practices across bioinformatics projects.

Repository-specific scientific logic and infrastructure credentials should
remain in the repositories that consume these utilities.

## Usage

Copy the desired example GitHub Actions workflow from this repository into the
`.github/workflows/` directory of the repository you want to productionize.
Review and adapt its configuration, permissions, and inputs for that project
before committing it.

Each example workflow will document its requirements and the values that
consumers are expected to customize.

## Contributing

Contributions should keep utilities broadly reusable, use least-privilege
permissions, avoid assumptions about institutional paths or infrastructure,
and include tests and consumer-facing documentation.

## License

This project is available under the [MIT License](LICENSE).
