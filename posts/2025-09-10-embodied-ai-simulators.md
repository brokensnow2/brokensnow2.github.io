---
layout: post
title: "Embodied AI Simulators: Habitat, AI2-THOR, and Gibson"
date: 2025-09-10
description: "A practical comparison of the main simulation platforms used in Embodied AI research."
categories: [research, embodied-ai]
---

If you're starting out in Embodied AI, one of your first decisions is: *which simulator do I use?*

The choice matters more than it might seem. Different simulators make different assumptions about physics, rendering, action spaces, and datasets. Picking the wrong one can mean weeks of wasted effort when you try to transfer results.

Here's what I've learned comparing the main platforms.

## The Contenders

### Habitat (Meta AI)

[Habitat](https://aihabitat.org) is the dominant platform for navigation tasks. It's built for speed — the simulator runs headless on a server cluster and can simulate thousands of steps per second per GPU.

**Strengths:**
- Extremely fast rendering (uses mesh-based rendering, not physics)
- Supports Matterport3D, Gibson, HM3D, and Replica datasets
- Tight integration with the `habitat-lab` RL framework
- Standard benchmark for PointNav, ObjectNav, and VLN tasks

**Weaknesses:**
- Physics is minimal — objects don't interact realistically
- Limited manipulation support (though Habitat 3.0 added humanoid agents)
- Configuration via YAML files gets unwieldy for complex experiments

**When to use it:** Navigation tasks (VLN, ObjectNav, ImageNav). It's the de facto standard — if your paper doesn't have Habitat numbers, reviewers will ask why.

### AI2-THOR (Allen Institute)

[AI2-THOR](https://ai2thor.allenai.org) prioritizes **interaction**. It's built on Unity and simulates household environments where agents can pick up objects, open drawers, toggle lights, and cook meals.

**Strengths:**
- Rich object interaction (86 interactive object types)
- Realistic physics via Unity PhysX
- Procedural scene generation (ProcTHOR: 10k+ synthetic rooms)
- Standard for object manipulation and task planning benchmarks

**Weaknesses:**
- Slower than Habitat (Unity overhead, no headless GPU rendering until recently)
- Visual fidelity is lower than Matterport/HM3D scans
- Action space is discrete by default (though continuous mode exists)

**When to use it:** Embodied task completion — ALFRED, TEACh, AgentTHOR. Anything requiring manipulation.

### Gibson / iGibson

[Gibson](http://gibsonenv.stanford.edu) is built around **real-world scan fidelity**. The original Gibson dataset uses point clouds from real buildings; iGibson 2.0 added fully interactive objects and support for tasks involving state changes (e.g., filling a glass with water).

**Strengths:**
- High visual realism (actual scan textures)
- iGibson 2.0 supports complex state-change tasks
- Good for sim-to-real transfer research

**Weaknesses:**
- Smaller scene library than HM3D
- Slower community adoption vs. Habitat
- Setup is more involved

**When to use it:** Sim-to-real work, or tasks requiring realistic state changes that AI2-THOR doesn't model well.

## Quick Comparison

| | Habitat | AI2-THOR | iGibson |
|---|---|---|---|
| Speed | ⚡⚡⚡ | ⚡ | ⚡⚡ |
| Navigation | ✅ Best | ✅ | ✅ |
| Manipulation | ⚠️ Limited | ✅ Best | ✅ |
| Physics | Minimal | Unity PhysX | Bullet |
| Scan realism | ✅ (Matterport) | ❌ Synthetic | ✅ |
| Community | Very large | Large | Medium |

## Setting Up Habitat (Quick Start)

```bash
# Create environment
conda create -n habitat python=3.9 -y
conda activate habitat

# Install habitat-sim (headless, with CUDA)
conda install habitat-sim withbullet headless -c conda-forge -c aihabitat -y

# Install habitat-lab
pip install habitat-lab

# Download a small scene to test
python -m habitat_sim.utils.datasets_download \
    --uids habitat_test_scenes
```

Then a minimal navigation episode:

```python
import habitat

config = habitat.get_config("benchmark/nav/pointnav/pointnav_habitat_test.yaml")
with habitat.Env(config=config) as env:
    obs = env.reset()
    print("Observation keys:", obs.keys())
    # obs contains 'rgb', 'depth', 'gps', 'compass'

    done = False
    while not done:
        action = env.action_space.sample()
        obs, reward, done, info = env.step(action)

    print("SPL:", info["spl"])
```

## My Take

For VLN research specifically, **Habitat + Matterport3D** is the path of least resistance. The ecosystem (datasets, baselines, evaluation scripts) is mature, and SOTA comparisons are easy to make.

If your research involves any manipulation or object interaction, you'll need AI2-THOR or iGibson. The two-simulator setup (Habitat for nav, AI2-THOR for manipulation) is increasingly common in papers that tackle full embodied task completion.

The real frontier is **sim-to-real** — none of these simulators produce policies that transfer cleanly to physical robots without significant adaptation. That's where 3D perception (the next post) becomes critical.
