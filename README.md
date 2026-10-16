# Policy evaluation harness: how do you tell whether a policy actually improved?

Two policies for the same task. One written by hand, one learned. On the
plant they were evaluated on, they are **identical**: 500 return, 100 %
success, 200 seeds each, and a bootstrap comparison that says the
difference is not significant.

Perturb the plant and one of them falls apart.

| | PD, handwritten | CEM, learned |
|---|---|---|
| nominal success rate | 1.00 | 1.00 |
| mean success across 24 perturbed cases | **0.93** | **0.66** |
| cases below 50 % success | 1 of 24 | **8 of 24** |

That gap is invisible to any amount of evaluation on the nominal plant.
This repo is about building the thing that makes it visible.

## Why a single number cannot answer the question

Four things this harness does that a mean return does not:

1. **Many seeds, with a confidence interval.** One rollout is a sample, not
   a result. 200 seeds per nominal condition, 60 per perturbed cell.
2. **Success rate alongside return.** They disagree as soon as a policy
   fails sometimes rather than always.
3. **A perturbation grid.** Mass, pole length, sensor noise, action delay,
   friction, each swept independently.
4. **A bootstrap comparison.** "Better" has to clear the noise floor before
   it counts as better.

## Where the learned policy breaks

Success rate, by perturbation. Trained on the nominal plant only, with
three fixed seeds, which is the whole cause.

**Pole mass** (nominal 0.1 kg)

| mass | PD | CEM |
|---|---|---|
| 0.1 | 1.00 | 1.00 |
| 0.5 | 1.00 | 1.00 |
| 1.0 | 1.00 | **0.32** |
| 2.0 | 1.00 | **0.02** |
| 4.0 | 1.00 | **0.00** |

**Action delay** (control steps, 20 ms each)

| delay | PD | CEM |
|---|---|---|
| 0 | 1.00 | 1.00 |
| 1 | 1.00 | 1.00 |
| 2 | 1.00 | **0.15** |
| 4 | 1.00 | 0.00 |
| 6 | 0.00 | 0.00 |

**Pole length** (nominal 0.5 m half length)

| length | PD | CEM |
|---|---|---|
| 1.0 | 1.00 | 0.90 |
| 1.5 | 1.00 | **0.03** |
| 2.0 | 0.83 | 0.00 |

Observation noise is the one axis where they degrade together, both
dropping to roughly 0.5 at 0.2 rad of noise. Friction moved neither.

![robustness](plots/02_robustness.png)

![distribution](plots/01_distribution.png)

## What I take from this

**The learned policy is not worse. It is narrower.** CEM found weights
`[0.04, 0.88, 1.77, 3.13]` that solve the plant it was shown, and nothing
constrained it to solve anything else. The PD controller survives because
its structure encodes something true about the problem rather than
something true about the training distribution.

**Both policies die at 6 steps of delay.** 120 ms, and the task becomes
impossible regardless of how the controller was produced. Worth separating
from the rest: that is a plant limit, not a policy failure, and no amount
of retraining fixes it. An evaluation report that does not make that
distinction will send someone off to tune a controller when they should be
fixing a sensor pipeline.

**The honest conclusion about the comparison** is that on the evidence
available at training time, there was no reason to prefer either policy.
The bootstrap says so explicitly. Any claim that the learned policy was an
improvement would have been unsupported, and the usual way that claim gets
made is by reporting a mean on the training distribution.

## Run it

```bash
pip install -r requirements.txt
python src/evaluate.py     # trains CEM, runs the grid, writes plots + data/eval_report.json
pytest -q
```

Takes a few minutes. Everything is seeded and reproducible.

## What I would do next

- Train with domain randomisation over the same grid and re evaluate. The
  expected result is that the learned policy catches up on robustness and
  gives up some nominal performance, and the point is to measure that
  trade rather than assume it.
- Add a joint perturbation grid rather than one axis at a time, since real
  degradation arrives in combinations.
- Add a regression gate: fail the run if success rate on any cell drops
  more than a threshold against the previous commit's report.
- Swap CartPole for something with contact, where the failure surface is
  sharper.

## Layout

```
src/env.py        CartPole with perturbable physics, noise and action delay
src/policies.py   handwritten PD, linear policy, CEM training
src/evaluate.py   seeds, confidence intervals, perturbation grid, bootstrap
tests/            physics checks and harness behaviour
```
