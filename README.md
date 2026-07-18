# repo-to-go 🦘

Guidelines and a checklist for making bioinformatics repositories portable and
reproducible

## Portability Checklist

[View the current checklist here.](./checklist.md)

## Usage

Copy the [example GitHub Actions
workflow](./examples/repo-portability-checklist.yml) from this repository into
the `.github/workflows/` directory of your repository.
Review and adapt the contents for your own project before committing it.
Run the action via [workflow
dispatch](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/manually-run-a-workflow)
to open an issue with the checklist.

```yaml
name: repo-portability-checklist

on:
  workflow_dispatch:
    inputs:
      repo:
        description: "The repository to open the issue in, with OWNER/REPO format. Defaults to the current repo."
        required: false
        default: "${{ github.repository }}"

permissions:
  issues: write

jobs:
  open-checklist:
    runs-on: ubuntu-latest
    steps:
      - name: Open checklist issue
        uses: caffiendFrog/repo-to-go@main
        with:
          repo: ${{ github.event.inputs.repo || github.repository }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

## About

This repository was created to share lessons learned from the work to make
[CHAMPAGNE](https://github.com/CCBR/CHAMPAGNE) more portable across execution
environments.

Initial development took place during [CoFest 2026, hosted by the Bioinformatics
Open Source Conference (BOSC)](https://www.open-bio.org/events/bosc-2026/collaborationfest/).

## License

This project is available under the [MIT License](LICENSE).
