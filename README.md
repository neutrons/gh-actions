# Neutrons Conda Actions for GitHub

## Overview

This repository contains GitHub actions for common conda package workflows, including installing a package into a test environment, verifying that it imports correctly, removing old packages from Anaconda Cloud, and publishing packages to Anaconda Cloud.

Some actions assume you have already built a `.conda` package. When using a local package artifact, place it in a conda-style channel directory (see [conda-index](https://github.com/conda/conda-index)).

Available actions:

- [pkg-install](#pkg-install): Create a micromamba environment and install a conda package into it.
- [pkg-verify](#pkg-verify): Verify an already-installed conda package by importing it in Python and checking that the conda and Python versions match.
- [pkg-remove](#pkg-remove): Clean up old conda packages from Anaconda Cloud.
- [publish](#publish): Publish a conda package to Anaconda Cloud.
- [grype](#grype): Run an Anchore Grype vulnerability scan and upload the SARIF results to GitHub Security.

## pkg-install

GitHub action to create a micromamba environment, optionally index a local conda channel, and install a conda package.

#### Usage

Full list of available inputs in [`pkg-install/action.yml`](pkg-install/action.yml).

Inputs:

| Input | Description | Required | Default |
| ---------------- | --------------------------------------------------------------------------------- | -------- | ------- |
| `package-name` | Name of the conda package to install | Yes | - |
| `local-channel` | Path to a local conda channel containing the package | No | - |
| `python-version` | Python version to install into the test environment (for example `3.10`) | No | - |
| `extra-channels` | Additional conda channels to use during installation | No | - |
| `post-cleanup` | Micromamba cleanup mode passed to `setup-micromamba` | No | `shell-init` |

Outputs:

| Output | Description |
| ------------------- | ---------------------------------------- |
| `conda_env` | Name of the created conda environment |
| `conda_install_dir` | Filesystem path of the created env |

Example:

```yaml
jobs:
  pkg-install:
    runs-on: ubuntu-latest
    defaults:
      run:
        shell: bash -el {0}
    steps:
      - name: Download conda package artifact
        uses: actions/download-artifact@main
        with:
          name: artifact-conda-package
          path: /tmp/local-channel/linux-64

      - name: Install Conda Package
        id: install
        uses: neutrons/conda-actions/pkg-install@main
        with:
          local-channel: /tmp/local-channel
          package-name: ${{ env.PKG_NAME }}
          python-version: "3.11"
          extra-channels: mantid neutrons pyoncat
```

## pkg-verify

GitHub action to verify a conda package that is already installed in a conda environment. The action imports the package in Python and ensures that the version reported by conda and Python match.

#### Usage

Full list of available inputs in [`pkg-verify/action.yaml`](pkg-verify/action.yaml).

Inputs:

| Input | Description | Required | Default |
| ---------------- | --------------------------------------------------------------------------------- | -------- | ------- |
| `package-name` | Name of the conda package | Yes | - |
| `module-name` | Name of the Python module to import (if different from package name) | No | - |
| `conda-env-name` | Name of the conda environment where the package is already installed | Yes | - |
| `extra-commands` | Additional shell commands to run during verification (newline-separated) | No | - |

Example usage in a GitHub workflow:

```yaml
jobs:
  # First, build your conda package and upload it as an artifact:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Build conda package
        run: |
          # steps to build your .conda package

      - name: Upload conda package as artifact
        uses: actions/upload-artifact@main
        with:
          name: artifact-conda-package
          path: ${{ env.PKG_NAME }}-*.conda

  # Then install and verify the conda package:
  pkg-verify:
    needs: build
    runs-on: ubuntu-latest
    defaults:
      run:
        shell: bash -el {0}
    steps:
      - name: Download conda package artifact
        uses: actions/download-artifact@main
        with:
          name: artifact-conda-package
          path: /tmp/local-channel/linux-64

      - name: Install Conda Package
        id: install
        uses: neutrons/conda-actions/pkg-install@main
        with:
          local-channel: /tmp/local-channel
          package-name: ${{ env.PKG_NAME }}
          extra-channels: mantid neutrons pyoncat

      - name: Verify Conda Package
        uses: neutrons/conda-actions/pkg-verify@main
        with:
          package-name: ${{ env.PKG_NAME }}
          conda-env-name: ${{ steps.install.outputs.conda_env }}
```

## pkg-remove

GitHub action to remove old packages of a specific label from [anaconda.org](https://anaconda.org),
keeping the N most recent versions.

#### Usage

Full list of available inputs in [`pkg-remove/action.yaml`](pkg-remove/action.yaml).

Inputs:

| Input | Description | Required | Default |
| ---------------- | --------------------------------------------------------------------- | -------- | ------- |
| `anaconda_token` | Anaconda.org API token | Yes | - |
| `organization` | Anaconda.org organization or user name | Yes | - |
| `package_name` | Name of the conda package to clean up | Yes | - |
| `label` | Label to target for cleanup (e.g., `dev`, `nightly`, `rc`) | No | `dev` |
| `keep` | Number of most recent package versions to keep | No | `5` |
| `dry_run` | If `true`, only print what would be deleted without actually deleting | No | `false` |

Outputs:

| Output | |
| ------------- | ------------------------------------- |
| `num_removed` | Number of files that would be deleted |

Example:

```yaml
jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Remove old dev packages
        uses: neutrons/conda-actions/pkg-remove@main
        with:
          anaconda_token: ${{ secrets.ANACONDA_TOKEN }}
          organization: neutrons
          package_name: my-package
          label: dev
          keep: 5
```

## grype

GitHub action to run an [Anchore Grype](https://github.com/anchore/grype) vulnerability scan on a directory and upload the SARIF results to GitHub Security.

#### Usage

Full list of available inputs in [`grype/action.yml`](grype/action.yml).

Inputs:

| Input | Description | Required | Default |
| ------------- | -------------------------------------------------------- | -------- | ------- |
| `path` | Path to scan (e.g. a conda environment directory) | Yes | - |
| `fail-build` | Fail the build if vulnerabilities are found | No | `false` |
| `only-fixed` | Only report vulnerabilities that have a fix available | No | `true` |

Example:

```yaml
jobs:
  scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
      actions: read
    steps:
	  # only need grype configuration
      - uses: actions/checkout@main
        with:
          sparse-checkout: |
            .grype.yaml
          sparse-checkout-cone-mode: false

      - name: Install Conda Package
        id: install
        uses: neutrons/conda-actions/pkg-install@main
        with:
          package-name: ${{ env.PKG_NAME }}

      - name: Scan with Grype
        uses: neutrons/conda-actions/grype@main
        with:
          path: ${{ steps.install.outputs.conda_install_dir }}
```

## publish

GitHub action to publish a pre-built conda package to Anaconda Cloud.

This action assumes that:

- The package has already been built and is available at the path given by `package-path`
- Either `anaconda-client` is available in `PATH`, or `pixi` is available so the action can run or install `anaconda-client`

If `label` is not provided, the action will attempt to determine it from `github-ref`:

- If the ref is tagged `refs/tags/v*rc*`, the package will be published to the `rc` label
- If the ref is tagged `refs/tags/v*`, the package will be published to the `main` label
- If the ref is tagged `refs/heads/next`, the package will be published to the `dev` label
- If the label cannot be determined from the ref, the action will fail

#### Usage

Full list of available inputs in [`publish/action.yaml`](publish/action.yaml).

Inputs:

| Input | Description | Required | Default |
| ---------------- | ------------------------------------------------------------ | -------- | ---------- |
| `anaconda-token` | Anaconda.org API token | Yes | - |
| `organization` | Anaconda.org organization or user name | Yes | - |
| `package-path` | Path to the conda package to publish | Yes | - |
| `github-ref` | GitHub ref (for example `refs/tags/v1.0.0`) used when inferring the label | No | `github.ref` |
| `label` | Label to apply to the package (e.g., `main`, `dev`, `nightly`, `rc`) | No | inferred from `github-ref` |
| `force` | If `true`, overwrite existing package with the same version | No | `false` |
| `dry-run` | If `true`, print the upload command and skip publishing | No | `false` |

Example:

```yaml
jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@main

      - uses: prefix-dev/setup-pixi@main

      - name: Build package
        run: |
          # steps to build your .conda package, for example:
          pixi build

      - name: Publish package to Anaconda Cloud
        uses: neutrons/conda-actions/publish@main
        with:
          anaconda-token: ${{ secrets.ANACONDA_TOKEN }}
          organization: neutrons
          package-path: my-package-*.conda
```
