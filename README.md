# 🏥 Emergency Room – Discrete Event Simulation

A discrete event simulation (DES) of the **follow-up treatment process in a hospital emergency department**, written in **Python** with **SymPy**. This is a Modelling & Simulation (ModSim) course project.

Patients arrive, register, wait for one of two casualty wards, and depending on their type go on to the X-ray room and/or the plaster room, sometimes coming back to a ward. We simulate the day, measure how long patients spend in the department, and try out queue limits and priority rules to improve the system.

---

## 📋 Table of Contents

- [System Overview](#-system-overview)
- [Patient Types](#-patient-types)
- [Model Parameters](#-model-parameters)
- [Tasks](#-tasks)
- [Why SymPy?](#-why-sympy)
- [Project Structure](#-project-structure)
- [Installation](#-installation)
- [Usage](#-usage)
- [Modelling Assumptions](#-modelling-assumptions)
- [Results](#-results)
- [Team](#-team)

---

## 🗺 System Overview

```
                 ┌──────────────┐
 Arrivals ──────▶│ Registration │  (1 clerk)
 (from 07:30)    └──────┬───────┘
                        │
                 ┌──────▼───────┐
                 │ Waiting area │
                 └──┬────────┬──┘
              60%   │        │   40%
            ┌───────▼──┐  ┌──▼───────┐
            │   CW1    │  │   CW2    │   (2 doctors each, from 08:00;
            │ 2 doctors│  │ 2 doctors│    CW2 = less experienced)
            └──┬────▲──┘  └──┬────▲──┘
               │    │        │    │
        ┌──────▼────┴────────▼────┴──────┐
        │  X-ray room: 2 units, 1 queue  │
        └──────────────┬─────────────────┘
                       │
               ┌───────▼────────┐
               │  Plaster room  │  (1 person)
               └────────────────┘
```

| Facility        | Servers | Notes                                             |
|-----------------|:-------:|---------------------------------------------------|
| Registration    | 1       | Assigns ward (CW1/CW2) and decides further treatment |
| Casualty Ward 1 | 2       | Experienced doctors                               |
| Casualty Ward 2 | 2       | Inexperienced doctors, for simpler cases          |
| X-ray room      | 2       | Both units share **one** queue                    |
| Plaster room    | 1       | Applies and removes plaster casts                 |

Patients queue in front of every treatment point.

---

## 🧑‍⚕️ Patient Types

| Type | Share | Description              | Route                                                    |
|:----:|:-----:|--------------------------|----------------------------------------------------------|
| 1    | 35 %  | X-ray required           | Registration → CW → X-ray → **CW** → exit                |
| 2    | 20 %  | Plaster removal          | Registration → CW → Plaster → exit                       |
| 3    | 5 %   | Plaster renewal + X-ray  | Registration → CW → X-ray → Plaster → X-ray → **CW** → exit |
| 4    | 40 %  | Wound dressing change    | Registration → CW → exit                                 |

Type 1 and type 3 patients visit a casualty ward **twice**.

---

## ⚙️ Model Parameters

**Arrivals**
- Interarrival times: exponential, parameter **0.3 min** (see [assumptions](#-modelling-assumptions))
- Patients arrive from **07:30**; doctors start at **08:00**

**Routing**
- Patient type: 35 % / 20 % / 5 % / 40 % (types 1–4)
- Ward assignment: **60 % CW1**, **40 % CW2**

**Service times**, triangular distribution `Triangular(min, mode, max)` in minutes:

| Treatment point | Min | Mode | Max |
|-----------------|:---:|:----:|:---:|
| Registration    | 0.2 | 0.5  | 1.0 |
| CW1             | 1.5 | 3.2  | 5.0 |
| CW2             | 2.8 | 4.1  | 6.3 |
| X-ray           | 2.0 | 2.8  | 4.1 |
| Plaster         | 3.0 | 3.8  | 4.7 |

---

## ✅ Tasks

### Task 1: Baseline simulation
Simulate **250 patients** and find the **average overall treatment time** (arrival to departure), overall and **by patient type (1–4)**.

### Task 2: Limit the CW2 queue
Put a cap on the CW2 queue. While the cap is reached, new patients go to CW1 instead until the CW2 queue gets shorter. **Find the best limit** by sweeping limit values and comparing the mean (and spread) of treatment time.

### Task 3: Priority ranking
Reduce the **standard deviation** of overall treatment time by adding a priority rule. Strategies we compare:
- **Returning patients first:** patients entering a treatment point for the second time (types 1 and 3) go ahead of everyone else
- *(further rules our group tests, e.g. shortest remaining route first, or prioritising by time already spent in the system)*

---

## 🔣 Why SymPy?

We use **SymPy** (`sympy.stats`) to define the model's random variables **symbolically**, which lets us:

- write the distributions exactly as they appear in the spec (`Exponential`, `Triangular`, finite type/ward distributions)
- compute **analytical means and variances** of service times (e.g. `E(T)`, `variance(T)`) and check them against the simulated values
- derive **utilisation estimates** (ρ = λ / (c·μ)) for each station, to see where bottlenecks should appear before running the simulation
- draw random samples from those same distributions with `sympy.stats.sample` to drive the simulation

```python
from sympy.stats import Triangular, E, variance

cw1 = Triangular("CW1", 1.5, 5.0, 3.2)   # (name, a=min, b=max, c=mode)
print(E(cw1))         # exact mean: 3.2333…
print(variance(cw1))  # exact variance
```

> ⚠️ Note the argument order in SymPy: `Triangular(name, a, b, c)` is **min, max, mode**, not min/mode/max as in the spec.

The event loop itself (event list, queues, servers) is plain Python built on a priority queue (`heapq`).

---

## 📁 Project Structure

```
emergency-room-sim/
├── README.md
├── requirements.txt
├── src/
│   ├── distributions.py   # SymPy random variables + sampling helpers
│   ├── entities.py        # Patient, Station, Queue classes
│   ├── simulation.py      # Event loop / scheduler
│   ├── policies.py        # Routing, CW2 queue limit, priority rules
│   └── analysis.py        # Statistics and plots
├── experiments/
│   ├── task1_baseline.py
│   ├── task2_queue_limit.py
│   └── task3_priority.py
├── results/               # Generated tables and figures
└── docs/
    └── ModSimProjectSpecification_EmergencyRoom.pdf
```

*(Adjust as the code evolves.)*

---

## 🛠 Installation

Requires **Python 3.10+**.

```bash
git clone https://github.com/<your-org>/emergency-room-sim.git
cd emergency-room-sim

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install -r requirements.txt
```

**`requirements.txt`**
```
sympy
numpy
matplotlib
pandas
```

---

## ▶️ Usage

```bash
# Task 1: baseline, 250 patients
python experiments/task1_baseline.py --patients 250 --replications 100 --seed 42

# Task 2: sweep CW2 queue limits
python experiments/task2_queue_limit.py --limits 1 2 3 4 5 6 8 10

# Task 3: compare priority rules
python experiments/task3_priority.py --rule returning-first
```

Because this is a stochastic simulation, every experiment runs **several independent replications** and reports means with **95 % confidence intervals**. Fix `--seed` to reproduce results.

---

## 📐 Modelling Assumptions

The specification leaves a few points open. Our choices:

1. **Interarrival parameter.** The spec says "exponential with parameter 0.3 minutes". Read as a **mean of 0.3 min**, ~3.3 patients arrive per minute, far above what the wards can handle, so queues grow without bound. Read as a **rate λ = 0.3 /min** (mean ≈ 3.33 min), the system is loaded but stable. The parameter is configurable; we report which reading we use. <!-- TODO: confirm with supervisor -->
2. **Opening times.** Registration runs from 07:30; patients can register before 08:00 but wait for the wards to open.
3. **Ward choice is fixed at registration.** A patient returning to a ward goes back to the **same** ward (CW1 or CW2).
4. **Queue discipline.** FIFO everywhere unless a priority rule is active (Task 3). No preemption: a patient already being treated is never interrupted.
5. **Transfer times** between rooms are ignored (0 min).
6. **Overall treatment time** = departure time − arrival time, including all waiting.
7. **Task 2 overflow rule.** A patient assigned to CW2 is sent to CW1 when the CW2 *queue* (not counting the patients being treated) has reached the limit.

---

## 📊 Results

*To be filled in once experiments are complete.*

| Patient type | Mean treatment time [min] | Std. dev. [min] | n |
|:------------:|:-------------------------:|:---------------:|:-:|
| 1            | –                         | –               | – |
| 2            | –                         | –               | – |
| 3            | –                         | –               | – |
| 4            | –                         | –               | – |
| **All**      | –                         | –               | 250 |

---

## 👥 Team

| Name | Role |
|------|------|
| Justus Becker    | Facility Management    |
| Elias Steininger | Team Lead |
| Jakob Orlik | Head of Information |
| Emil Weber | Head of Documentation | 

---

*Modelling & Simulation course project, 2026.*
