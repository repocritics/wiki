# Farama-Foundation/Gymnasium

> A standard API for single-agent reinforcement learning environments — the maintained successor to OpenAI Gym.

[GitHub repo](https://github.com/Farama-Foundation/Gymnasium) ·
[Official website](https://gymnasium.farama.org) ·
[License: MIT](https://github.com/Farama-Foundation/Gymnasium/blob/main/LICENSE)

## Overview

Gymnasium defines the `reset()` / `step()` interface that the reinforcement-learning ecosystem uses to connect learning agents to environments. It is a hard fork of OpenAI's Gym, taken over by the maintainer team after OpenAI stepped away from the project, and is where all future work now happens[^1]. The library ships the API contract plus a set of reference environments (Classic Control, Box2D, Toy Text, MuJoCo) and standard tooling — spaces, wrappers, and vectorization — that the rest of the RL stack builds on.

The single most important thing to understand is that Gymnasium is *not* a training library. It contains no agents, no algorithms, no neural networks. It is the socket that algorithm libraries (Stable-Baselines3, CleanRL, Ray RLlib, Tianshou) plug into. Its value is entirely in being a stable, widely-adopted standard, which is why the API changes it inherited from Gym 0.26 — chiefly splitting the old `done` boolean into separate `terminated` and `truncated` signals — mattered so much: they broke essentially every environment and training loop written against pre-2022 Gym[^2].

The defining tension is backward compatibility versus correctness. Gym's original 4-tuple `step` return conflated "the episode reached a real terminal state" with "we cut the episode off at a time limit," which silently corrupts bootstrapping in value-based methods. Gymnasium fixed this correctly but at the cost of a migration that the RL community is, years later, still not fully done with — a large corpus of tutorials, papers, and third-party environments still targets the old signature.

## Getting Started

```bash
pip install gymnasium
# environment families are optional extras:
pip install "gymnasium[box2d]"     # or [mujoco], [atari], [all]
```

```python
import gymnasium as gym

env = gym.make("CartPole-v1", render_mode="rgb_array")

observation, info = env.reset(seed=42)
for _ in range(1000):
    action = env.action_space.sample()          # random policy
    observation, reward, terminated, truncated, info = env.step(action)

    if terminated or truncated:
        observation, info = env.reset()
env.close()
```

Note the two migration-critical details: `reset()` takes the seed (there is no separate `seed()` method), and `step()` returns a five-tuple. Treating `terminated or truncated` as the loop-ending condition is correct for control flow, but the two flags must be kept distinct for the learning update.

## Architecture / How It Works

The core is the abstract `gymnasium.Env` class with four things every environment declares:

- **`observation_space`** and **`action_space`** — typed `Space` objects (`Box`, `Discrete`, `MultiDiscrete`, `Dict`, `Tuple`, `Text`, `Graph`, `Sequence`). Spaces are self-describing and know how to `sample()` and `contains()`, which is how generic code can operate on any environment without knowing its specifics.
- **`reset(seed, options)`** → `(observation, info)`.
- **`step(action)`** → `(observation, reward, terminated, truncated, info)`.
- **`render()`** — driven by the `render_mode` fixed at `make()` time, not passed per-call.

**Registration and versioning.** Environments are registered under string IDs (`"CartPole-v1"`) and `gym.make` constructs them. Every ID carries a version suffix; when a change would alter learning results, the version is bumped rather than mutated in place, so a paper citing `-v1` reproduces regardless of what `-v2` later does[^3]. This is a deliberate reproducibility guarantee inherited from Gym.

**Wrappers** are the composition mechanism — `ObservationWrapper`, `RewardWrapper`, `ActionWrapper`, and lifecycle wrappers like `TimeLimit`, `RecordEpisodeStatistics`, and `FrameStackObservation`. `gym.make` silently applies `TimeLimit` and other wrappers by default, which is where `truncated` usually originates. Wrappers stack, and `env.unwrapped` reaches the base environment.

**Vectorization.** `gym.make_vec` / `SyncVectorEnv` / `AsyncVectorEnv` run N copies of an environment to feed batched samples to a policy. `Sync` runs them in-process in a loop; `Async` uses multiprocessing with shared-memory buffers. The 1.0 release rewrote this subsystem, changing the autoreset semantics so a sub-environment resets on the step *after* it terminates rather than in the same step[^4] — a subtle behavioral change that affects how you read the batched `terminated`/`truncated` arrays.

The physics-backed families delegate outward: MuJoCo environments now use Google DeepMind's official `mujoco` Python bindings (the old `mujoco-py`, which required a license key, is gone), and Atari lives in a separate package (`ale-py` / the ALE project) rather than in the core repo.

## Production Notes

**The migration tax is the real cost.** If you are integrating any code written before ~2023, expect the four-vs-five-tuple mismatch. Gymnasium ships compatibility shims (`gymnasium.make(..., apply_api_compatibility=True)` and the `Shimmy` package for wrapping legacy Gym / dm_env / OpenSpiel environments), but they are a bridge, not a fix — new work should target the native five-tuple.

**Bootstrapping correctness.** The whole point of `terminated` vs `truncated` is that you bootstrap the value function on truncation but not on termination. A large amount of copied-from-tutorial code does `done = terminated or truncated` and then uses `done` in the TD target, silently reintroducing the exact bug the split was meant to eliminate. This is the most common latent correctness error in RL code today.

**Environment install pain.** `pip install gymnasium` gives you almost nothing runnable beyond Classic Control and Toy Text. Box2D needs `swig` and a working C toolchain; MuJoCo pulls a large binary wheel; Atari requires accepting ROM licensing (historically via `AutoROM`). `[all]` frequently fails on at least one platform, which is why the maintainers keep the families as separate extras. Windows is accepted-but-unsupported — PRs are merged, but CI runs Linux and macOS only.

**Vector env footguns.** `AsyncVectorEnv` pickles your environment across process boundaries; closures, un-picklable handles, and CUDA contexts in the constructor break it. Multiprocessing overhead can make `Async` slower than `Sync` for cheap environments — measure before assuming parallel is faster. And the 1.0 autoreset change means final observations are now surfaced through `info` (`final_obs`) rather than inline, breaking loops written for 0.2x vector envs.

**It is a spec, not a runtime.** Do not benchmark "Gymnasium performance" — throughput is dominated by the underlying simulator (MuJoCo, Box2D) and your policy's inference, not by the API layer. For accelerator-native throughput (millions of steps/sec), the Gymnasium API is a bottleneck by design, and JAX-based alternatives exist.

## When to Use / When Not

**Use when:**
- You are training or evaluating single-agent RL and want the interface every algorithm library already speaks.
- You need reproducible benchmark environments with versioned semantics for a paper or comparison.
- You are authoring a custom environment and want it to work with off-the-shelf trainers for free.
- You want wrappers and vectorization as solved problems rather than reinventing them.

**Avoid when:**
- Your problem is multi-agent — use PettingZoo, its sibling project, instead.
- You need accelerator-resident, massively-parallel rollouts — the Python-object step loop caps you; reach for JAX-native envs.
- You are locked to a large legacy Gym codebase and cannot afford the terminated/truncated migration right now.
- You expected training algorithms in the box — this library deliberately has none.

## Alternatives

- openai/gym — the deprecated predecessor; unmaintained, use Gymnasium instead unless pinned by a legacy dependency.
- Farama-Foundation/PettingZoo — use instead when the problem has more than one agent.
- RobertTLange/gymnax — use instead when you need JAX-native, GPU-vectorized environments for high-throughput training.
- google/brax — use instead when you want differentiable, accelerator-parallel physics rather than CPU MuJoCo.
- google-deepmind/dm_control — use instead when you want DeepMind's continuous-control suite under the `dm_env` interface.

## History

| Version | Date | Notes |
|---------|------|-------|
| (Gym 0.26) | 2022-09 | Upstream Gym splits `done` into `terminated`/`truncated`; basis for the fork[^2]. |
| 0.26.x | 2022-10 | First releases under the Gymnasium name after the Farama fork[^1]. |
| 0.27–0.28 | 2023 | Wrapper reorganization, experimental functional API, vector env improvements. |
| 0.29 | 2023-07 | Last of the 0.2x line; broad ecosystem adoption baseline. |
| 1.0.0 | 2024-10 | Vector env rewrite, autoreset semantics change, wrapper/API cleanup[^4]. |
| 1.x | 2025–2026 | MuJoCo/Atari maintenance, Python 3.13/3.14 support, ongoing 1.x releases. |

## References

[^1]: Gymnasium README — "This is a fork of OpenAI's Gym library by its maintainers ... where future maintenance will occur going forward." https://github.com/Farama-Foundation/Gymnasium
[^2]: Gymnasium migration guide, "Handling Termination and Truncation." https://gymnasium.farama.org/introduction/migration_guide/
[^3]: Gymnasium documentation, "Environment Versioning." https://github.com/Farama-Foundation/Gymnasium#environment-versioning
[^4]: Towers et al., "Gymnasium: A Standard Interface for Reinforcement Learning Environments," arXiv:2407.17032, 2024. https://arxiv.org/abs/2407.17032

## Tags

python, reinforcement-learning, machine-learning, rl-environments, api-standard, openai-gym, simulation, mujoco, atari, vectorized-environments
