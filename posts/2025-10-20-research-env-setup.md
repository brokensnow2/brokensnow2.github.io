---
layout: post
title: "My Research Environment Setup (2025 Edition)"
date: 2025-10-20
description: "The tools and configs I use for deep learning research: conda, PyTorch, wandb, Hydra, and a few quality-of-life tricks."
categories: [programming, tools]
---

Every researcher has a setup they've converged on after too many broken environments and lost experiments. This is mine.

This post is for anyone starting a new project or machine and wanting a solid baseline quickly. I'll cover environment management, experiment tracking, config management, and a few things I wish I'd started using earlier.

## Environment: conda + pip hybrid

I use `conda` to manage Python versions and system-level dependencies (CUDA, cudnn), and `pip` for Python packages. Mixing `conda install` and `pip install` in the same environment is fine as long as you install conda packages first.

```bash
# Create a fresh env
conda create -n myproject python=3.10 -y
conda activate myproject

# CUDA-aware PyTorch (adjust cuda version)
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121

# Verify GPU access
python -c "import torch; print(torch.cuda.get_device_name(0))"
```

I keep a `requirements.txt` per project, but I don't pin every transitive dependency — just the direct ones. Over-pinning is its own maintenance burden.

## Config Management: Hydra

[Hydra](https://hydra.cc) replaces `argparse` and ad-hoc config dicts. Once you get used to it, going back is painful.

Project structure:

```
myproject/
├── conf/
│   ├── config.yaml        # defaults
│   ├── model/
│   │   ├── transformer.yaml
│   │   └── lstm.yaml
│   └── data/
│       ├── r2r.yaml
│       └── reverie.yaml
├── train.py
└── ...
```

`conf/config.yaml`:
```yaml
defaults:
  - model: transformer
  - data: r2r
  - _self_

trainer:
  lr: 1e-4
  batch_size: 32
  max_epochs: 20
```

`train.py`:
```python
import hydra
from omegaconf import DictConfig

@hydra.main(config_path="conf", config_name="config", version_base=None)
def main(cfg: DictConfig):
    print(f"LR: {cfg.trainer.lr}, Model: {cfg.model._target_}")
    # Hydra handles output dirs, logging, multirun automatically

if __name__ == "__main__":
    main()
```

Overriding from CLI:
```bash
# Single run
python train.py trainer.lr=5e-5 model=lstm

# Grid search (multirun)
python train.py --multirun trainer.lr=1e-4,5e-5 model=transformer,lstm
```

The multirun output is automatically organized by Hydra into `outputs/` with timestamps and configs saved per run.

## Experiment Tracking: wandb

[Weights & Biases](https://wandb.ai) is my default. Setup:

```bash
pip install wandb
wandb login  # paste your API key
```

Integration:

```python
import wandb

wandb.init(
    project="vln-experiments",
    config=dict(cfg),  # log Hydra config
    name=f"lr{cfg.trainer.lr}-{cfg.model._target_}"
)

# Inside training loop
wandb.log({
    "train/loss": loss.item(),
    "val/sr": success_rate,
    "val/spl": spl,
    "epoch": epoch,
})

wandb.finish()
```

Things I always log: loss curves, val metrics per epoch, GPU memory, and at least one sample prediction per validation run. The last one saves hours of debugging.

## Reproducibility

```python
import random
import numpy as np
import torch

def set_seed(seed: int = 42):
    random.seed(seed)
    np.random.seed(seed)
    torch.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)
    # For full determinism (slower):
    torch.backends.cudnn.deterministic = True
    torch.backends.cudnn.benchmark = False
```

Call this at the very start of training, before any model or dataloader initialization. Log the seed to wandb.

## Cluster: SLURM + submitit

For multi-GPU or multi-node jobs on a SLURM cluster, [submitit](https://github.com/facebookincubator/submitit) wraps SLURM into Python cleanly:

```python
import submitit

executor = submitit.AutoExecutor(folder="slurm_logs/%j")
executor.update_parameters(
    slurm_partition="gpu",
    gpus_per_node=4,
    nodes=1,
    slurm_time="12:00:00",
    mem_gb=64,
)

job = executor.submit(main, cfg)
print(f"Job ID: {job.job_id}")
```

Combined with Hydra's multirun, this lets you launch hyperparameter sweeps directly to the cluster from a single command.

## Editor: VS Code + Remote SSH

I edit locally, execute remotely. The VS Code Remote SSH extension makes this seamless. Key extensions I use:

- **Pylance** — type checking and autocomplete
- **Python** — linting, formatting
- **GitLens** — blame annotations, history
- **Remote SSH** — edit files on the cluster directly

Format on save with `black` and `isort`:

```json
// .vscode/settings.json
{
  "editor.formatOnSave": true,
  "python.formatting.provider": "black",
  "[python]": {
    "editor.codeActionsOnSave": {
      "source.organizeImports": true
    }
  }
}
```

## A Few Things I Wish I'd Done Earlier

1. **Log everything from day one.** Storage is cheap; re-running ablations because you forgot to log something is not.

2. **Use `torch.compile()` for free speedups.** On PyTorch 2.x, wrapping your model with `model = torch.compile(model)` often gives 20–40% speedup with zero code changes.

3. **Profile before optimizing.** `torch.profiler` tells you exactly where time is spent. More than once I've optimized the wrong bottleneck.

4. **Keep a lab notebook.** Not for code — for hypotheses, results, and why you tried things. I use a private Notion page. Future-you will thank present-you.

---

Setup evolves. I'll update this post when something significant changes — probably when the next major PyTorch release lands.
