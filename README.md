# VEDA

**V**accine **E**pitope **D**iscovery and **A**nalysis — a pipeline for
prioritising candidate epitopes from immunopeptidomics data.

VEDA takes a peptide list (for example FragPipe output), filters it, and runs
it through eleven established prediction tools, keeping the intermediate
results so you can see why a candidate survived or was dropped. It is a
Streamlit application with a browser interface; the prediction tools run as
separate processes in their own pinned environments.

```bash
git clone https://github.com/vedasoham/veda_epitope-discovery.git
cd veda_epitope-discovery
./install.sh
veda
```

> **Status.** VEDA is research software developed for our own work and opened
> up in case it is useful. It is tested on Ubuntu with an x86-64 CPU. Expect
> rough edges, and please open an issue rather than assuming it is you.

## What it does

| Step | Uses | Question it answers |
|---|---|---|
| Upload & filter | — | which peptides and source proteins are worth carrying forward? |
| MHC class-I binding | NetMHCpan 4.2 | will this peptide be presented on class I? |
| MHC class-II binding | NetMHCIIpan 4.3 | and on class II? |
| Motif clustering | GibbsCluster 2.0, Seq2Logo 2.2 | what binding motifs are present? |
| Subcellular localisation | DeepLoc 2.1 | is the source protein surface-exposed or secreted? |
| Protegenicity | Vaxign-ML | does the protein look like a protective antigen? |
| Allergenicity | AlgPred 2.0 | would this peptide risk an allergic response? |
| Toxicity | ToxinPred 2.0 | is it predicted toxic? |
| Cytokine induction | IL2Pred, IL4Pred 2.0, IFNEpitope 2.0 | does it induce IL-2, IL-4 or IFN-γ? |
| Figures | — | publication-ready summaries |

Transmembrane topology via DeepTMHMM is wired up but disabled in the
interface; see [Known gaps](#known-gaps).

Every step writes its raw tool output alongside the parsed table, so results
are auditable rather than a black box.

## Requirements

- **Ubuntu or Debian**, x86-64. The prediction tools ship as Linux x86-64
  binaries with no ARM build, so Apple Silicon and ARM servers cannot run them.
- **~25 GB free disk**, **8 GB RAM** or more.
- **sudo**, for a handful of apt packages (`tcsh`, `perl`, `git`, build tools).
- **An NVIDIA GPU is optional.** With a driver, DeepLoc runs on the GPU; without
  one it runs on the CPU, slower. The installer picks the right PyTorch build
  from your driver version and **never installs a driver or CUDA itself**.
- **An existing Anaconda or Miniconda is fine and is left completely alone.**
  VEDA installs its own private micromamba under `~/.veda` and does not touch
  your conda, your environments, or your `~/.condarc`.

Check a machine before installing anything:

```bash
./veda-doctor --system
```

That reports your distro, GPU and the PyTorch build it implies, existing conda
installations, Docker, package names resolved for your release, and anything in
your shell environment that would interfere. It installs nothing.

## Installing

```bash
./install.sh
```

Ten stages, each recorded so an interrupted run resumes where it stopped:
system packages, a private micromamba, five pinned Python environments, the
prediction tools, Docker images, configuration, a `veda` launcher, then
verification that runs every tool against a real input.

Useful flags:

| Flag | Effect |
|---|---|
| `--cpu-only` | force CPU PyTorch even if a GPU is present |
| `--skip-docker` | skip Docker (only Vaxign-ML needs it) |
| `--only <stage>` | run one stage, reusing what exists |
| `--only <stage> --force` | rebuild that stage |
| `--dry-run` | show what would happen |
| `--help` | all options |

[INSTALL.md](INSTALL.md) documents every stage, what gets modified, and
troubleshooting.

## The prediction tools

**VEDA does not bundle any of them.** They belong to their authors, and five
are licensed in a way that forbids redistribution. The installer handles this
in two halves.

**Fetched automatically** — cloned from the authors' own repositories:

| Tool | Licence |
|---|---|
| [AlgPred 2.0](https://github.com/raghavagps/algpred2) | GPL-3.0 |
| [ToxinPred 2.0](https://github.com/raghavagps/toxinpred2) | GPL-3.0 |
| [IL2Pred](https://github.com/raghavagps/il2pred) | see repository |
| [IL4Pred 2.0](https://github.com/raghavagps/il4pred2) | GPL-3.0 |
| [IFNEpitope 2.0](https://github.com/raghavagps/ifnepitope2) | see repository |
| [Vaxign-ML](https://github.com/VIOLINet/Vaxign-ML-docker) | GPL-3.0 |

**You download these five yourself.** They are free for academic use, but DTU
Health Tech issues a personal, single-use download link after a registration
form, and the licence does not allow us to pass the programs on — so no script
can fetch them for you.

Open each page, complete the form, and choose the **Linux x86-64** build:

| Tool | Download page | Unpacks to |
|---|---|---|
| NetMHCpan 4.2 | [services.healthtech.dtu.dk/services/NetMHCpan-4.2](https://services.healthtech.dtu.dk/services/NetMHCpan-4.2/) | `netMHCpan-4.2/` |
| NetMHCIIpan 4.3 | [services.healthtech.dtu.dk/services/NetMHCIIpan-4.3](https://services.healthtech.dtu.dk/services/NetMHCIIpan-4.3/) | `netMHCIIpan-4.3/` |
| GibbsCluster 2.0 | [services.healthtech.dtu.dk/services/GibbsCluster-2.0](https://services.healthtech.dtu.dk/services/GibbsCluster-2.0/) | `gibbscluster-2.0/` |
| Seq2Logo 2.2 | [services.healthtech.dtu.dk/services/Seq2Logo-2.0](https://services.healthtech.dtu.dk/services/Seq2Logo-2.0/) | `seq2logo-2.2/` |
| DeepLoc 2.1 | [services.healthtech.dtu.dk/services/DeepLoc-2.1](https://services.healthtech.dtu.dk/services/DeepLoc-2.1/) | `deeploc-2.1.All/` |

### Where to put them

Leave the archives as they arrived — **do not unpack or rename them** — drop
them into VEDA's staging directory, and re-run the tools stage:

```bash
mkdir -p ~/.veda/staging
mv ~/Downloads/netMHCpan-4.2*.tar.gz ~/.veda/staging/    # and the others

./install.sh --only tools
```

Do not rename or extract them: the installer finds each archive by name,
unpacks it into `resources/tools/`, rewrites the installation paths baked into
its `tcsh` wrappers and `envfile`s, fixes permissions `tar` did not preserve,
and checks that the data payload arrived and not just the launcher script.
Unpacking by hand skips all of that.

Already extracted one elsewhere? Move or symlink the folder into
`resources/tools/` under the directory name in the table above.

NetMHCpan and NetMHCIIpan carry their scoring matrices in a `data/` directory
*inside* the archive — 190 MB and 125 MB respectively. If a download is
interrupted and that directory is short, the tool prints a single
`Unable to open(r) file .../data/version` and then **exits 0 having predicted
nothing**, which looks like success to anything checking only the exit code.
The installer refuses to count such a tool as installed, and `veda doctor`
reports it by name.

**[resources/tools/DTU_TOOLS.md](resources/tools/DTU_TOOLS.md)** has the
step-by-step version, including what to do when one misbehaves. Budget about
ten minutes, plus however long DTU takes to email the links.

**VEDA runs without them.** The steps they power are greyed out and everything
else works. Add them whenever you like and re-run
`./install.sh --only tools`.

## Using it

```bash
veda                 # start the web interface
veda doctor          # check every tool, running each on a real input
veda doctor --system # profile the machine
veda paths           # show resolved interpreter and tool paths
veda shell           # a shell inside VEDA's main environment
veda --help
```

Start the app with `veda`, not `streamlit run src/main.py`. The launcher sets
`PYTHONNOUSERSITE=1` and cleans `PATH` and `LD_LIBRARY_PATH` for the
subprocesses; without it, packages in your `~/.local` silently override the
versions VEDA pins.

### A note on data

The DeepTMHMM step, if you enable it, runs in cloud mode and **uploads your
sequences to BioLib's servers**. Everything else runs entirely on your machine.
If you work with unpublished or patient-derived sequences, know this before
enabling that step.

## What installing changes

Only these paths:

```
~/.veda/                      environments, caches, logs, state
~/.local/bin/veda             launcher symlink
~/.local/share/applications/  desktop entry
<repo>/resources/tools/       the tools themselves
<repo>/veda_paths.json        generated machine-local paths
~/.bashrc                     one guarded PATH line, only if you agree
```

Tool files that are rewritten are backed up next to the original as
`*.veda-orig`. `./uninstall.sh` reverses it, and `./uninstall.sh --all` also
removes the downloaded tools.

## Known gaps

Honest list, so you do not rediscover these:

- **DeepTMHMM is disabled in the interface.** Cloud mode through pybiolib
  works and is verified by `veda doctor`. The local Docker path cannot be
  provisioned: there is no public DeepTMHMM image to pull, and the code refers
  to a bare image ID that only resolves where a previous biolib run cached it.
- **`$VEDA_HOME` must be shallow.** The DTU binaries hold their scratch path in
  a 64-byte buffer: 63 characters works, 64 aborts with
  `*** buffer overflow detected ***`. The installer refuses to proceed
  otherwise. The default `~/.veda` is 15 characters.
- **x86-64 only**, as above.
- **IL4Pred 2.0 is installed and verified but not wired into the pipeline.**
  The installer fetches it and `veda doctor` runs a real prediction through
  it, so the tool itself works. What is missing is the worker that would call
  it during an analysis: the results parser and display code exist
  (`parse_il4pred_results`, `render_il4pred_results`) but nothing invokes the
  tool, so no IL-4 step appears in the interface. To get IL-4 predictions
  today, run it directly from `resources/tools/il4pred2-main` with the
  `veda-tools` interpreter.
- The Vaxign-ML image name is inconsistent in one code path; the installer
  pulls the image the workers use.

### Three different scikit-learn versions, on purpose

If you look inside `~/.veda/mamba/envs` and wonder why the pins disagree: the
bundled model files were pickled across six years of scikit-learn releases,
and version 1.3 changed the decision-tree node struct so that a forest cannot
cross that boundary in either direction.

| Environment | scikit-learn | Tools (and the version that pickled their models) |
|---|---|---|
| `veda-tools` | 1.2 | AlgPred2 (0.19.2), ToxinPred2 (0.22), IL4Pred2 (0.24.1) |
| `veda-ifn` | 1.2 | IFNEpitope2 (0.24.1) |
| `veda-il2` | 1.5 | IL2Pred (1.5.2) |

Please do not consolidate these onto one version — there is no single version
that works, and the failure mode is quiet. A tool with the wrong scikit-learn
still imports, still prints its banner and still answers `--help`; it fails
only when it tries to load its model:

```
ValueError: node array from the pickle has an incompatible dtype
```

The installer asserts all three versions on every run and repairs them in
place, and verification runs a real prediction through each tool rather than
checking that it responds to `-h`.

## Authors

VEDA was developed at **Gujarat Biotechnology University**.

| | |
|---|---|
| Soham Bhatt | development |
| Arpita Tiwari | development |
| Dr Dhaval Patel | supervision |
| Dr Ritesh Kumar | supervision |

## Citing

VEDA orchestrates other people's methods. **Please cite the tools you actually
used** alongside VEDA — each has its own paper, linked from its repository or
its DTU service page. A prediction in your figure is their work.

## Licence

[MIT](LICENSE). Copyright © 2026 Soham Bhatt, Arpita Tiwari, Dr Dhaval Patel,
Dr Ritesh Kumar, and Gujarat Biotechnology University.

Use it, change it, build on it, commercially or otherwise; just keep the
copyright notice. There is no warranty.

This covers VEDA's own code only. The prediction tools are separate programs
under their own licences, invoked as subprocesses and never redistributed
here, so their terms do not constrain this one — five of them are licensed by
DTU Health Tech to a named individual and must not be passed on. See
[resources/tools/DTU_TOOLS.md](resources/tools/DTU_TOOLS.md).

## Contributing

Issues and pull requests welcome. Useful things to include in a bug report:

```bash
./veda-doctor --system > system-report.txt
./veda-doctor -v       > doctor.txt
```

Those two files describe the machine and every component, which is usually
enough to tell a VEDA bug from a tool or environment problem. Please do not
attach tool binaries or research data.
