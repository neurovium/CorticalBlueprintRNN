# 05 — Tasks

Notebook section: **Tasks** + **Generate datasets for training**

Three task generators are defined as standalone Python classes. Each one is instantiated once per call to a corresponding `Generate datasets…` subsection, which produces the `(X_train, y_train)`, `(X_val, y_val)`, `(X_test, y_test)` numpy arrays consumed by the training loops.

For all three tasks: training = 5,120 trials, validation = 2,560 trials, test = 2,560 trials, batch size 128.

## Task 1 — One-Choice Inference (`mazeGeneratorI`)

The Achterberg et al. 2023 one-step navigation task: read a goal, see candidate moves, output the move that goes toward the goal.

```python
class mazeGeneratorI():
    def __init__(self, goal_presentation_steps, delay_steps, choices_presentation_steps):
```

| stage | duration | what the network sees |
|---|---|---|
| goal presentation | 20 steps | one of 4 goal channels active |
| delay | 10 steps | constant input |
| choices presentation | 20 steps | 2 of 4 candidate-direction channels active |

- **Input shape:** `(trials, 50, 8)` — 8 binary channels (4 goal + 4 directions).
- **Output:** 4-way softmax (`L / U / R / D`), produced at the final time step.
- **What it probes:** maintenance across the delay + relational inference between goal and choices.

## Task 2 — Go/NoGo (`GoNogoGeneratorI`)

Standard Go/NoGo from Zhang et al. 2019.

```python
class GoNogoGeneratorI:
    def __init__(self, fixation_steps, stimulus_steps, delay_steps, decision_steps):
```

| stage | duration | what the network sees |
|---|---|---|
| fixation | 5 steps | fixation channel only |
| stimulus | 20 steps | one of 2 stimulus channels active (Go or NoGo) |
| delay | 10 steps | fixation channel only |
| decision | 5 steps | fixation channel only; network outputs at the last step |

- **Input shape:** `(trials, 40, 3)` — 1 fixation + 2 stimulus channels.
- **Output:** 2-way softmax (Go / NoGo), at the final time step.
- **What it probes:** short-term memory across the delay + output gating (don't respond until the decision phase).

## Task 3 — Perceptual Decision-Making (`PerceptualDecisionMakingGeneratorI`)

Britten et al. 1992 random-dot motion-style task: noisy evidence in two channels, identify the dominant one.

```python
class PerceptualDecisionMakingGeneratorI:
    def __init__(self, dt=100, timing=None, cohs=None, sigma=1.0, dim_ring=2, seed=None):
```

| stage | duration | what the network sees |
|---|---|---|
| fixation | 1 step | fixation channel |
| stimulus | 30 steps | 2 evidence channels at coherence ∈ {0, 6.4, 12.8, 25.6, 51.2}% |
| delay | 10 steps | fixation channel |

- **Input shape:** `(trials, 41, 3)` — 1 fixation + `dim_ring = 2` evidence channels.
- **Output:** 2-way softmax (which alternative is dominant), at the final time step.
- **What it probes:** evidence integration across time at varying signal-to-noise.

Coherence parameterizes difficulty: 0% is chance, 51.2% is near-deterministic.

## How the datasets are wired into training

Each `Generate datasets for training / TASK N` subsection:

1. Instantiates the generator with the timing parameters above.
2. Calls it to produce `X_*`, `y_*` arrays of the shapes listed.
3. One-hot encodes `y` if the generator hasn't already.
4. Hands the arrays to the per-task training loop in **Model Training / TASKN**, where every variant fits to the same data and the same `(train, val, test)` split. The split is fixed per-task so model variants are compared on identical inputs.

The Gaussian-noise input layer (`σ = noise_level = 0.05`) is applied inside the model graph, not in the dataset.
