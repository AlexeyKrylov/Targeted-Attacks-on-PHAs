# LLM-Guided Program Evolution for Targeted Black-Box Attacks on Perceptual Hash Algorithms

**Aleksei S. Krylov, Denis S. Rakhov, Veronica S. Veselova, Oleg Y. Rogov**

Official code for the paper. <!-- TODO: add paper link (DOI / arXiv) once available -->

Perceptual hash algorithms (PHAs) map visually similar images to nearby compact
descriptors. We study the **targeted second-image attack** in a black-box
setting: modify a source image so that its hash falls within the matching
threshold of a chosen target hash, while keeping the image visually close to the
source. The attacker sees only hash distances returned by an oracle.

The framework has three parts:

1. **A unified hash-oracle evaluator** for pHash, PDQ, PhotoDNA, and NeuralHash.
   It recomputes every metric from the returned image and never trusts numbers
   reported by the attack itself.
2. **Per-hash hyperparameter optimization** (Optuna) of five gradient-free seed
   attacks: ATKScopes, NES, Prokos, SimBA, and ZO-SignSGD.
3. **LLM-guided program evolution** with [GigaEvo](https://github.com/FusionBrainLab/gigaevo-core)
   and [OpenEvolve](https://pypi.org/project/openevolve/). These frameworks
   change the attack procedure itself (search space, step schedules, restarts,
   post-success distortion reduction), not only its hyperparameters.

## Results

All results use the 30-pair ImageNet benchmark (images resized to 256x256).
Lower composite score `S` is better:

```
S = (mean L2 + 0.02 * mean queries) / max(ASR, 0.05)   (+10000 if ASR = 0)
```

Query budgets per pair: 10k for pHash and PDQ, 20k for PhotoDNA, 30k for
NeuralHash.

| Hash | Threshold | Best optimized seed | S<sub>opt</sub> | Best evolved | S<sub>evo</sub> | Improvement |
|---|---|---|---|---|---|---|
| pHash (64-bit, Hamming) | 12 | ATKScopes | 86.08 | OpenEvolve | 56.80 | **34.0%** |
| PDQ (256-bit, Hamming) | 92 | ATKScopes | 44.25 | OpenEvolve | 27.31 | **38.3%** |
| PhotoDNA (144-d, L1) | 3855 | SimBA | 153.64 | GigaEvolve | 141.22 | **8.1%** |
| NeuralHash (96-bit, Hamming) | 17 | ATKScopes | 654.89 | GigaEvolve | 384.84 | **41.2%** |

GigaEvolve is our pipeline built on GigaEvo. Full per-attack tables, LPIPS, and
an analysis of what the evolved programs changed are in the paper. The eight
final evolved programs are in [`evolved_programs/`](evolved_programs/), with a
summary of their changes in
[`evolved_programs/evolution_changes_summary.md`](evolved_programs/evolution_changes_summary.md).

## Repository structure

```
attacks/              seed attacks: ATKScopes, NES, Prokos, SimBA, ZO-SignSGD
hashes/               pHash, PDQ, PhotoDNA, NeuralHash wrappers (+ defenses)
oracle.py             query-counting hash oracle (the attack's only feedback)
evaluator.py          external evaluator: recomputes ASR, L2, LPIPS, queries, score
parallel_eval.py      parallel evaluation over image pairs
optimizer.py          Optuna-based configuration optimization
stage1_find_ASR/      stage 1: ASR screening of seed configurations (+ results)
stage2_optimize/      stage 2: configuration optimization notebooks
stage3_common/        shared prompts, problem notes, and validation for evolution
stage3_gigaevolve/    stage 3: GigaEvo integration
stage3_openevolve/    stage 3: OpenEvolve integration
stage4_benchmark/     stage 4: final benchmark, tables, figures
evolved_programs/     final evolved attack programs
results/              original and optimized attack configurations
data/                 benchmark image pairs (ImageNet)
third_party/          GigaEvo pin and the patch applied on top of it
```

## Installation

Tested with Python 3.12 on Windows. PhotoDNA runs natively only on Windows; the
benchmark skips it on other platforms.

The code is imported as the Python package `evohash`, so **clone the repository
into a folder named `evohash`**:

```bash
git clone https://github.com/AlexeyKrylov/Targeted-Attacks-on-PHAs.git evohash
cd evohash

# GigaEvo core at the pinned commit + our patch (details: third_party/README.md)
git clone https://github.com/FusionBrainLab/gigaevo-core third_party/gigaevolve_core
git -C third_party/gigaevolve_core checkout 7a3ab40
git -C third_party/gigaevolve_core apply ../gigaevo-core-evohash.patch

python -m venv .venv
.venv\Scripts\activate          # Linux/macOS: source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

`requirements.txt` installs GigaEvo in editable mode, so clone it first.

- **GigaEvo** stores its population in Redis: `docker compose up -d redis`.
- **LLM access** for evolution: copy `.env.example` to `.env` and set
  `OPENROUTER_API_KEY`. Alternatively, point the notebooks to a local
  OpenAI-compatible server (`LLM_SOURCE = 'local'`).
- **Running modules from the command line**: put the parent folder on
  `PYTHONPATH` so that `import evohash` works.

### Hash binaries

pHash and PDQ work out of the box. The PhotoDNA library and the NeuralHash model
are third-party binaries, so they are **not** distributed here:

- **PhotoDNA**: place `PhotoDNAx64.dll` into `hashes/dll/` (or set
  `PHOTODNA_DLL`). The DLL ships inside the AccessData FTK 7.0.0 installer ISO.
  On Linux, `evohash.hashes.photodna.setup_photodna()` downloads it and runs it
  under Wine, following [pyPhotoDNA](https://github.com/jankais3r/pyPhotoDNA).
- **NeuralHash**: follow [AppleNeuralHash2ONNX](https://github.com/AsuharietYgvar/AppleNeuralHash2ONNX)
  to export the model. Place the ONNX model as `hashes/model/model.onnx` and the
  seed file (`neuralhash_128x96_seed1.dat`) as `hashes/model/model.dat`, or set
  `NEURALHASH_MODEL_DIR`.

### Data

- `data/dataset_v1_public.zip` contains the 30 public ImageNet pairs used for
  optimization, evolution, and the reported results.
- `data/dataset_v1_full.zip` contains the public pairs plus 100 held-out
  (`secret`) pairs for additional validation.

## Reproducing the pipeline

Each stage has one notebook with a configuration cell at the top (`HASH_ID`,
budget, number of pairs, LLM). Open each notebook from its own folder.

| Stage | Notebook | What it does |
|---|---|---|
| 2 | [`stage2_optimize/optimize.ipynb`](stage2_optimize/optimize.ipynb) | Optuna tuning of seed attacks for pHash / PDQ / PhotoDNA |
| 2 | [`stage2_optimize/NeuralHash_optimize.ipynb`](stage2_optimize/NeuralHash_optimize.ipynb) | the same for NeuralHash (separate search space) |
| 3 | [`stage3_openevolve/OpenEvolve.ipynb`](stage3_openevolve/OpenEvolve.ipynb) | program evolution with OpenEvolve |
| 3 | [`stage3_gigaevolve/GigaEvolve.ipynb`](stage3_gigaevolve/GigaEvolve.ipynb) | program evolution with GigaEvo (needs Redis) |
| 4 | [`stage4_benchmark/Benchmark.ipynb`](stage4_benchmark/Benchmark.ipynb) | original vs optimized configs vs evolved programs, LaTeX tables |

Stage 1 screening results are stored in [`stage1_find_ASR/results/`](stage1_find_ASR/results/).
The configurations produced by stage 2 are in
[`results/attack_configs_optimized.json`](results/attack_configs_optimized.json).
That file is the seed context for evolution and the "optimized" side of the
benchmark.

Quick checks: generate an evolution problem package without starting a run. Run
these from the repository folder with `PYTHONPATH` set to its parent.

```bash
python -m evohash.stage3_gigaevolve.run --hash phash --dataset data/dataset_v1_public.zip --dry-run
python -m evohash.stage3_openevolve.run --hash phash --dataset data/dataset_v1_public.zip --llm-source none --dry-run
```

Exact numbers depend on the random seed, the resize settings, the hash backend
(CPU/GPU for NeuralHash), and the LLM. Program evolution is stochastic, so a
rerun produces different programs.

## Responsible use

This code is released to support research on the robustness of perceptual
hashing systems and on defenses against collision attacks. Do not use it to
evade or abuse content-matching systems deployed in production.

## Citation

```bibtex
@article{krylov2026llmguided,
  title   = {LLM-Guided Program Evolution for Targeted Black-Box Attacks on Perceptual Hash Algorithms},
  author  = {Krylov, Aleksei S. and Rakhov, Denis S. and Veselova, Veronica S. and Rogov, Oleg Y.},
  journal = {TODO},
  year    = {2026}
}
```

## License

The code is released under the [MIT License](LICENSE). The license does not
cover the third-party hash binaries or the ImageNet images in `data/`, which
remain subject to their owners' terms.

## Acknowledgements

This work builds on [GigaEvo](https://github.com/FusionBrainLab/gigaevo-core),
[OpenEvolve](https://pypi.org/project/openevolve/), the ATKScopes and Prokos et al.
attacks on perceptual hashes, [AppleNeuralHash2ONNX](https://github.com/AsuharietYgvar/AppleNeuralHash2ONNX),
and [pyPhotoDNA](https://github.com/jankais3r/pyPhotoDNA).
