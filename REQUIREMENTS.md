# Requirements

**You do not need to install any of this by yourself.** `./install.sh` builds all
of it, and it is the only supported path:

```bash
./install.sh
```

This page is the reference: what gets installed, which version, and into which
environment. It is useful if you are reviewing the software, reproducing an
analysis, or building the environments manually on a machine where you cannot
run the installer (an HPC login node, for example).

## Why there is no single `requirements.txt`

A flat pip requirements file cannot describe VEDA, for three reasons. Each one
is a genuine conflict, not a matter of taste:

1. **Two incompatible scikit-learn versions are required at once.** The
   prediction tools ship pickled models saved between 2017 and 2024, and
   scikit-learn 1.3 changed the decision-tree node struct. AlgPred2 (models
   from 0.19.2), ToxinPred2 (0.22), IL4Pred2 and IFNEpitope2 (0.24.1) load
   only on scikit-learn **below** 1.3; IL2Pred's models (1.5.2) load only
   **at or above** it. No single version runs both, so they live in separate
   environments.
2. **Two dependencies are not pip-installable.** `veda-tools` needs BLAST+
   (`blastp`, `makeblastdb`) and Perl, which come from conda, not PyPI.
3. **The correct PyTorch wheel depends on your GPU driver.** The installer
   reads the driver version and picks a CUDA 12.1, CUDA 11.8 or CPU build. A
   static requirement would be wrong on most machines.

Anyone who ran `pip install -r requirements.txt` into one environment would
get a VEDA whose predictors import cleanly, print help, and then fail to load
their models. The machine-readable specs are `installer/envs/*.yml` and
`installer/envs/*.pip.txt`.

## System packages

Installed with `apt-get` by the `system` stage. The installer resolves each
name against your release, so these differ slightly between Ubuntu versions.

| | |
|---|---|
| Required | `ca-certificates` `curl` `wget` `git` `git-lfs` `tcsh` `perl` `unzip` `bzip2` `xz-utils` `build-essential` `pkg-config` `gnupg` `lsb-release` |
| Required, renamed | `libglib2.0-0t64` on Ubuntu 24.04+, `libglib2.0-0` before it |
| Optional | `libgl1` (or `libgl1-mesa-glx` before 24.04), `gnuplot-nox` |

`tcsh` and `perl` are not optional: the DTU wrappers are tcsh scripts, and
GibbsCluster and the MERCI motif locator are Perl.

You also need **Anaconda or Miniconda already installed** — the installer uses
it to bootstrap a private micromamba under `~/.veda/mamba`, and never modifies
your conda installation or your `~/.condarc`. **Docker** is optional; only
Vaxign-ML needs it.

## The five Python environments

Built by the `envs` stage into `~/.veda/mamba/envs`, from
`installer/envs/<name>.yml`. They are separate because their pins conflict.

### `veda-ui` — Python 3.10

The Streamlit application, the dashboards, and DeepLoc2's host environment.

| Source | Packages |
|---|---|
| conda | `setuptools<81` `numpy=1.26.*` `pandas=2.*` `scipy=1.13.*` `scikit-learn=1.7.*` `joblib` `biopython` `openpyxl` `matplotlib-base` |
| pip | `streamlit==1.52.1` `plotly==6.3.0` `matplotlib-venn==1.1.2` `pybiolib==1.2.1602` `onnxruntime==1.23.2` `fair-esm==2.0.0` `transformers==5.5.1` `pytorch-lightning==2.5.0` `sentencepiece==0.2.1` `watchdog` `psutil` `requests` `tqdm` |
| separately | PyTorch 2.5.1 (see below), and DeepLoc2 from `resources/tools/deeploc-2.1.All` |

### `veda-tools` — Python 3.9

AlgPred 2.0 and ToxinPred 2.0, and the BLAST binaries every tool shares.

| Source | Packages |
|---|---|
| conda | `setuptools<81` `numpy=1.26.*` `pandas=2.*` `joblib` **`scikit-learn=1.2.*`** `blast` `perl` |

The workers locate BLAST as `Path(config.TOOL_PYTHON).parent / "blastp"`, so
it must be inside *this* environment's `bin/` even though other tools use it.

### `veda-ifn` — Python 3.10

IFNEpitope 2.0.

| Source | Packages |
|---|---|
| conda | `setuptools<81` `numpy=1.26.*` `pandas=2.*` `joblib` **`scikit-learn=1.2.*`** `tqdm` |

### `veda-il2` — Python 3.9

IL2Pred, alone, because its models need a scikit-learn the others cannot use.

| Source | Packages |
|---|---|
| conda | `setuptools<81` `numpy=1.26.*` `pandas=2.*` `joblib` **`scikit-learn=1.5.*`** |

### `veda-tmhmm` — Python 3.9

DeepTMHMM through pybiolib. Deliberately minimal — the worker imports nothing
numeric, so there is nothing here to conflict.

| Source | Packages |
|---|---|
| pip | `pybiolib==1.2.1602` `typing_extensions` |

## Pins that are load-bearing

Changing any of these breaks VEDA at run time, not at install time, which is
why they are version-bounded rather than left to float:

| Pin | Why |
|---|---|
| `scikit-learn` 1.2 / 1.5 as above | model pickles straddle the 1.3 node-struct change |
| `setuptools<81` | setuptools after 80.x dropped the bundled `pkg_resources`, which DeepLoc2 and `ifnepitope2.py` both import directly |
| `numpy=1.26.*` | PyTorch 2.5 and onnxruntime both break against numpy 2.x, and the model pickles predate the numpy 2 ABI |
| `pandas=2.*` | the tool wrappers restore `DataFrame.append`, removed in pandas 2.0 |

The installer asserts the scikit-learn and setuptools versions on every run
and repairs them in place, because a version that has drifted produces tools
that import, answer `--help`, and then cannot predict.

## PyTorch

Needed by DeepLoc2. Version **2.5.1**, with the build chosen from your NVIDIA
driver:

| Driver | Wheel index | Build |
|---|---|---|
| ≥ 525 | `https://download.pytorch.org/whl/cu121` | CUDA 12.1 |
| ≥ 450 | `https://download.pytorch.org/whl/cu118` | CUDA 11.8 |
| none, or older | PyPI default | CPU |

`./install.sh --cpu-only` forces the CPU build. Run `./veda-doctor --system`
to see what your machine resolves to before installing. The installer does
**not** install NVIDIA drivers or the CUDA toolkit; if you want GPU support,
install the driver first (`ubuntu-drivers install`) and re-run.

## External prediction tools

Twelve programs by other authors, none of them in this repository. Six are
fetched automatically; five are licence-gated and you request them yourself;
DeepTMHMM runs as a service. Sizes, sources and instructions are in
[resources/tools/README.md](resources/tools/README.md) and
[resources/tools/DTU_TOOLS.md](resources/tools/DTU_TOOLS.md).

## Disk and platform

| | |
|---|---|
| Platform | Linux x86-64 (developed and tested on Ubuntu) |
| Environments | ~8.5 GB under `~/.veda/mamba` |
| Tools | ~1.7 GB under `resources/tools` |
| DeepLoc model weights | ~2.5 GB, downloaded on first use |
| `$VEDA_HOME` path length | must stay under 64 characters — the DTU binaries hold it in a fixed buffer |

## Building the environments by hand

If you cannot run the installer, the `.yml` files are ordinary conda specs:

```bash
for e in veda-ui veda-tools veda-ifn veda-il2 veda-tmhmm; do
    conda env create -f installer/envs/$e.yml
done

conda run -n veda-ui pip install -r installer/envs/veda-ui.pip.txt
conda run -n veda-tmhmm pip install -r installer/envs/veda-tmhmm.pip.txt
conda run -n veda-ui pip install torch==2.5.1 \
    --index-url https://download.pytorch.org/whl/cu121      # or cu118, or omit for CPU
```

Set `PYTHONNOUSERSITE=1` for those pip commands. Without it pip may report the
packages already satisfied from `~/.local/lib/pythonX.Y/site-packages` and
install nothing into the environment, leaving it empty but apparently working.

Doing this by hand skips the `configure` stage, so you must still write
`~/.veda/veda_paths.json` yourself or run `./install.sh --only configure` to
point VEDA at the interpreters you built.
