
# Kiri Pull Request GitHub Action

This is a convenient and easy way to run [Kiri](https://github.com/leoheck/kiri) against a Pull Request using GitHub Actions.

The base Kiri image is hosted in the GitHub Container Repository here <https://github.com/USA-RedDragon/kiri-github-action/pkgs/container/kiri>,
which is based on the Kiri image at <https://github.com/leoheck/kiri-docker>

## PR HTML Preview Setup

In order to provide PRs with a link to preview the changes, this action pushes to the `gh-pages` branch of the source
repository and hosts the Kiri output in subdirectories.

For this to work properly, you'll need to make an empty `gh-pages` branch with the file `.nojekyll` (to avoid `_KIRI_` folders returning a 404).

### Deleting PRs on close

Running this action in the context of `pull_request.closed` will delete any Kiri previews that were made for the PR.

## Action inputs

All inputs are **optional**.

|           Name           |                               Description                                |
| ------------------------ | ------------------------------------------------------------------------ |
| `all`                    | If set, include all commits even if schematics/layout don't have changes |
| `last`                   | Show last N commits                                                      |
| `newer`                  | Show commits up to this one                                              |
| `older`                  | Show commits starting from this one                                      |
| `skip-cache`             | If set, skip usage of -cache.lib on plotgitsch                           |
| `skip-kicad6-schematics` | If set, skip ploting Kicad 6 schematics (.kicad.sch)                     |
| `force-layout-view`      | If set, force starting with the Layout view selected                     |
| `pcb-page-frame`         | If set, disable page frame for PCB                                       |
| `archive`                | If set, archive generated files                                          |
| `remove`                 | If set, remove generated folder before running it                        |
| `output-dir`             | If set, change output folder path/name                                   |
| `project-file`           | Path to the KiCad project file                                           |
| `extra-args`             | Extra arguments to pass to Kiri                                          |
| `kiri-debug`             | If set, enable debugging output                                          |

## Examples

### Quick Start

```yaml
# .github/workflows/pr-kicad-diff.yaml
name: KiCad Pull Request Diff

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

on:
  pull_request:
    types:
    - opened
    - synchronize
    paths:
    - kicad/*.kicad_pcb
    - kicad/*.kicad_sch
    - kicad/*.kicad_pro

jobs:
  kiri-diff:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
      with:
        ref: ${{ github.event.pull_request.head.sha }}
    - name: Kiri
      uses: usa-reddragon/kiri-github-action@v1
      with:
        project-file: kicad/productname.kicad_pro
```

```yaml
# .github/workflows/pr-kicad-diff-delete.yaml
name: KiCad Diff Delete

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false

on:
  pull_request:
    types:
    - closed

jobs:
  kiri-delete:
    runs-on: ubuntu-latest
    steps:
    - name: Kiri
      uses: usa-reddragon/kiri-github-action@v1
```

### Requiring Kiri for PR merges

The `on.pull_request.paths` object is the typical method of filtering to only
run Kiri on PRs with KiCad file changes. Unfortunately, making a GitHub Action
required for merging means that the workflow must run for all PRs, even
those without KiCad changes.

```yaml
# .github/workflows/pr-kicad-diff.yaml
name: KiCad Pull Request Diff

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

on:
  pull_request:
    types:
    - opened
    - synchronize
    paths:
    - kicad/*.kicad_pcb
    - kicad/*.kicad_sch
    - kicad/*.kicad_pro

jobs:
  kiri-diff:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
      with:
        ref: ${{ github.event.pull_request.head.sha }}
    # If the PR hasn't changed any KiCad files, we don't need to run Kiri.
    - name: Check for KiCad files
      id: kicad-files
      run: |
        if git diff --name-only ${{ github.event.pull_request.base.sha }} | grep -q '\.kicad\(_pro\|_sch\|_pcb\)\?$'; then
          echo "changed=true" >> $GITHUB_OUTPUT
        else
          echo "changed=false" >> $GITHUB_OUTPUT
        fi
    - name: Kiri
      uses: usa-reddragon/kiri-github-action@v1
      if: steps.kicad-files.outputs.changed == 'true'
      with:
        project-file: kicad/productname.kicad_pro
```
