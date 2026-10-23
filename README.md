<!--
  README for the anonymized review repository (ECIS 2027).
  Placeholders in <angle brackets> and lines marked TODO must be filled from the code.
  Before uploading to anonymous.4open.science: remove names, affiliations, e-mail addresses,
  funding information, and links to non-anonymous repositories (also in code comments and DESCRIPTION).
-->

# Simulating Collusion and Enforcement

An R tool that generates **ground-truth data** on cartels. It simulates the birth-and-death process of cartels in a population of heterogeneous industries over time and returns both the **population** of all cartel episodes and the **sample** of detected ones. Because the same estimator can be applied to both, its deviation from the truth becomes measurable.

The tool accompanies the paper *Simulating Collusion and Enforcement: A Ground-Truth Artifact for Evaluating Empirical Methods* (under review). Section and table numbers below refer to that paper.

---

## 1. Quick Start

**Requirements**

- R ≥ `<version>`  <!-- TODO -->
- Packages: `<list, e.g. data.table, survival, glmnet, ggplot2>`  <!-- TODO -->

```r
# install dependencies
<install command, e.g. renv::restore() or source("install.R")>   # TODO

# reproduce all tables and figures of the paper
source("<scripts/run_all.R>")                                      # TODO
```

Runtime: about `<x>` minutes on a standard laptop.  <!-- TODO -->
All results are written to `output/`.

---

## 2. Repository Structure

```
<repo>/
├── R/            simulation engine: modules M1–M6, period loop, output layer
├── config/       scenario configurations (Models I, IIA, IIB, III)
├── analysis/     estimators: descriptive statistics, OLS, Weibull, LASSO-CV,
│                 Bryant–Eckard, duration bias
├── scripts/      reproduction scripts, one per table or figure
├── output/       generated datasets, tables, figures (not under version control)
└── README.md
```

<!-- TODO: replace with the actual tree, e.g. via `tree -L 2` -->

---

## 3. Architecture

The tool consists of four layers (paper, Figure 1): a configuration layer, a simulation engine with six modules, an output layer, and an analysis layer.

| Layer / module | What it does | Paper | File / function |
|---|---|---|---|
| Configuration layer | Holds all parameters, distributions, horizon $T$, number of industries $k$, seed, scenario | 4.1, Table 2 | `<config/…>` |
| M1 Industry generator | Draws $\eta_i$, $\kappa_i$ and, in Model IIB, $\Pr(L)_i$ once per industry | 4.2 | `<…>` |
| M2 Market shocks | Draws the profit realization $\pi$ in each period | 4.2 | `<…>` |
| M3 Collusion decision | Evaluates the ICC (Eq. 1): form or continue a cartel iff $\pi \le \Phi_i(\sigma_t)$ | 4.2 | `<…>` |
| M4 Enforcement | Detects active or just-collapsed cartels with probability $\sigma_t$, unless leniency was filed | 4.3 | `<…>` |
| M5 Leniency | On collapse, files leniency if $\sigma_t > \theta$ (always in IIA; with probability $\Pr(L)_i$ in IIB) | 4.3 | `<…>` |
| M6 CA capacity | Updates conviction probability $s_t$ from the caseload (Eq. 2); $\sigma_t = q \cdot r \cdot s_t$ | 4.3 | `<…>` |
| Period loop | Runs steps 1–6 below for each period and industry | 4.4 | `<…>` |
| Output layer | Converts the event log into population, sample, and panel datasets | 4.5 | `<…>` |
| Analysis layer | Applies each estimator to sample and population and compares the results | 5 | `analysis/` |

---

## 4. Model Logic

### 4.1 Collusion decision

Industry $i$ colludes in period $t$ if and only if

$$
\pi \le \Phi_i(\sigma_t) = \frac{\delta(1-\sigma_t)(1-\kappa_i)\,\mu - [1-\delta(1-\kappa_i)]\,\max\{\sigma_t-\theta,0\}\,\gamma\mu}{(\eta_i-1)\,[1-\delta(1-\kappa_i)]}
$$

This is the ICC of Harrington and Chang (2015) with two implementation choices:

- $\alpha = 0$ (Bertrand competition with homogeneous goods and constant marginal cost);
- the value of collusion $Y$ is replaced by the mean profit $\mu$ ($Y := \mu$). This departs from the subgame-perfect equilibrium but makes simulation over many periods tractable.

Both choices are set in `<file / function>` and can be replaced there.  <!-- TODO -->

### 4.2 Period loop

In each period $t$, for every industry $i$:

1. **Draw** the profit realization $\pi$ (M2).
2. **Cool-down.** If the industry's cartel ended in $t-1$, it stays competitive in $t$.
3. **Formation.** A non-cartelized industry obtains an opportunity to cartelize with probability $\kappa_i$ and forms a cartel if the ICC holds (M3).
4. **Continuation or collapse.** A cartel continues if the ICC holds and collapses otherwise; on collapse, M5 decides on leniency.
5. **Detection.** A cartel that is active or collapsed in $t$ without a leniency application is detected with probability $\sigma_t$ (M4). Detection ends the cartel.
6. **Bookkeeping.** States and events are written to the event log; in Model III, M6 updates the caseload and $s_{t+1}$.

### 4.3 Implementation details

<!-- TODO: answer these three questions; they are also open in the paper (% CHECK comments). -->

| Question | Answer |
|---|---|
| Is the profit $\pi$ drawn per industry ($\pi_{it}$) or once per period for all industries ($\pi_t$)? | `<…>` |
| Which caseload enters $s_t$ in Model III: the previous period only, or a backlog window of several periods? | `<…>` |
| Is *times caught* (TC) counted up to the start of each episode, or over the full horizon? | `<…>` |

---

## 5. Configuration

All inputs are set in the configuration layer; changing them requires no change to the model logic.

| Code name | Symbol | Meaning | Default | Scenarios |
|---|---|---|---|---|
| `<k>` | $k$ | Number of industries | 1'000 | all |
| `<T>` | $T$ | Number of periods | 1'000 | all |
| `<seed>` | — | Random seed | `<…>` | all |
| `<delta>` | $\delta$ | Common discount factor | 0.85 | all |
| `<alpha>` | $\alpha$ | Competitive profit as share of collusive profit | 0 | all |
| `<eta>` | $\eta_i$ | Deviation profit multiple | LN(1, 1.5) on [1.1, 10] | all |
| `<kappa>` | $\kappa_i$ | Opportunity to cartelize | U(0, 0.1) | all |
| `<prL>` | $\Pr(L)_i$ | Probability of leniency on collapse | $(1-10\kappa_i)/2$ | IIB |
| `<pi>` | $\pi$ | Profit realization | LN(0, 1.5) on [1, 100] | all |
| `<sigma>` | $\sigma$ | Penalty probability (exogenous) | 0.048 | I, IIA, IIB |
| `<q>`, `<r>` | $q$, $r$ | Discovery and investigation probability | 0.2, 0.3 | III |
| `<tau>`, `<xi>`, `<nu>`, `<rho>`, `<lambda>` | $\tau, \xi, \nu, \rho, \lambda$ | Parameters of the capacity function (Eq. 2) | 0.8, 1, 500, 1.5, 0.5 | III |
| `<gamma>` | $\gamma$ | Fine multiplier | 2 | all |
| `<theta>` | $\theta$ | Fine share paid by leniency applicant | 1 (I); 0.035 (II, III) | all |

LN(m, s): lognormal with meanlog m and sdlog s, truncated to the stated range.

### Scenarios

| Scenario | Config file | Settings |
|---|---|---|
| Model I: no leniency | `<config/…>` | $\theta = 1$, exogenous $\sigma$ |
| Model IIA: deterministic leniency | `<config/…>` | $\theta < \sigma$, all collapsing cartels apply |
| Model IIB: probabilistic leniency | `<config/…>` | as IIA, cartels apply with $\Pr(L)_i$ |
| Model III: CA capacity | `<config/…>` | as IIA, $\sigma_t = q \cdot r \cdot s_t$ with endogenous $s_t$ |

### Seeds and comparability

All random draws come from one seeded stream. Industry characteristics and profit realizations are **identical across scenarios** (common random numbers), so differences between scenarios stem from the enforcement assumptions alone. `<Describe how this is implemented, e.g. draws generated once and passed to all scenarios.>`  <!-- TODO -->

To run a scenario with several seeds:

```r
<example call>   # TODO
```

---

## 6. Output Data

### 6.1 Population (one row per cartel episode)

| Column | Type | Description |
|---|---|---|
| `<industry>` | integer | Industry identifier $i$ |
| `<start>` | integer | First period of the episode |
| `<end>` | integer | Last period of the episode |
| `<duration>` | integer | Number of periods the cartel was active |
| `<end_type>` | factor | `collapse` (undetected), `detected` (CA), `leniency`, `active` (still running at $T$) |
| `<detected>` | 0/1 | Ended by CA detection |
| `<leniency>` | 0/1 | Ended by leniency application |
| `<sample>` | 0/1 | In the detected sample (`detected` or `leniency`) |
| `<eta>` | numeric | $\eta_i$ |
| `<kappa>` | numeric | $\kappa_i$ |
| `<prL>` | numeric | $\Pr(L)_i$ (Model IIB) |
| `<TC>` | integer | Times caught (see Section 4.3) |
| `<repeat>` | 0/1 | Repeat offender |

<!-- TODO: align column names and add any further columns -->

**Right-censoring.** Episodes still running at $T$ remain in the population as undetected, with duration measured up to $T$, but cannot enter the sample.

### 6.2 Detected sample

The subset of the population with `sample == 1`, i.e. the same filter a researcher implicitly applies when collecting conviction data. Same schema as the population.

### 6.3 Panel and aggregates

| Dataset | Unit | Content |
|---|---|---|
| `<panel>` | industry × period | State (competitive, cartel, cool-down) and events |
| `<aggregates>` | period | Shares of colluding, CA-detected, and leniency industries (paper, Figure 2) |

---

## 7. Analysis Functions

Each function is applied to the sample and to the population with the same code. Results feed into the method-by-assumption matrix (paper, Table 5).

### 7.1 Descriptive statistics and duration bias

`<function>`: mean and median duration, and the duration bias of Harrington and Wei (2017):

$$
\text{Bias} = 100 \cdot \frac{\text{ADT} - \text{ACD}}{\text{ACD}}
$$

with ADT the average duration of detected cartels and ACD the average duration of all cartels.

### 7.2 Bryant–Eckard detection probability

`<function>`: per-period probability of detection, following Bryant and Eckard (1991).

- **Sample estimate.** Under a constant per-period hazard of detection, the maximum-likelihood estimate from detected cartels is
  $\hat p = n_{\text{detected}} / \sum \text{duration} = 1/\text{ADT}$.
  <!-- TODO: describe your exact implementation: discrete (geometric) vs. continuous (exponential) time, treatment of truncation near T, standard errors. -->
- **Population benchmark.** The realized hazard of exit by detection or leniency:
  number of detected episodes / sum of durations of all population episodes (all cartel-periods).
- **Output.** `<…>`

### 7.3 Duration regressions

`<function>`: OLS and Weibull hazard models.

- Dependent variable: Ln(Duration).
- Regressors: Ln($\eta$), $\kappa$, Ln(TC+1), and, in Models II and III, a leniency dummy. $\Pr(L)$ is excluded because it is a linear function of $\kappa$.
- Estimated on: CA cases, leniency cases, the full sample, undetected cartels, and the population.

### 7.4 Heckman selection correction

`<function or "not implemented">`. In all four scenarios, detection ends the cartel, so no variable affects detection without affecting duration, and the correction is not identified (paper, Section 5.3). An evaluation requires an extension in which detection is possible after a cartel's natural collapse.

### 7.5 LASSO-CV feature selection

`<function>`: selects covariates by cross-validated LASSO, including the interactions Ln($\eta$) × Leniency and Ln($\eta$) × Ln(TC+1), then re-estimates OLS on the selected set. Reports the change in adjusted $R^2$.

### 7.6 Bias heterogeneity

`<function>`: groups cartels by identical industry characteristics ($\eta$, $\kappa$), computes the duration bias per group, and regresses it on Ln($\eta$) and $\kappa$ (paper, Section 5.4, Figure 4).

---

## 8. Reproducing the Paper

| Paper element | Content | Script |
|---|---|---|
| Table 4 | Population vs. detected sample | `<scripts/…>` |
| Table 5 | Method-by-assumption matrix | `<scripts/…>` |
| Figure 2 | Colluding, detected, and leniency industries over time | `<scripts/…>` |
| Figure 3 | OLS coefficients by scenario and subsample | `<scripts/…>` |
| Figure 4 | Duration bias by group | `<scripts/…>` |
| Online appendix | Full regression tables, Weibull and LASSO results | `<scripts/…>` |

---

## 9. Extending the Simulator

Modifications fall into three tiers by implementation effort (paper, Table 3). Because the output schema stays the same, the analysis functions can be reused without change.

**Tier 1: configuration only.** Change any parameter, the number of industries, the horizon, or the seed.

```r
<example: run Model III with r = 0.5>   # TODO
```

**Tier 2: local amendment of the ICC or a distribution.** Examples: interpret $\eta$ as the number of firms or as a concentration measure; use a uniform or skewed distribution for $\eta$, $\kappa$, or $\pi$; change the leniency propensity $\Pr(L)$; introduce leniency mid-horizon as a policy shock.

```r
<example: replace the distribution of eta>   # TODO
```

**Tier 3: rewrite of the ICC or a new module.** Examples: negative instead of positive demand shocks as the cause of collapse (Green and Porter, 1984); incomplete cartels; entry and exit; recidivism with higher detection after a first conviction; detection after a natural collapse (needed to evaluate the Heckman correction).
New modules go into `<R/…>` and are called from the period loop in `<file / function>`.  <!-- TODO -->

---

## 10. Citation and License

<!-- Review version: keep anonymous. After acceptance, add the citation and author information. -->

License: `<e.g. MIT>`  <!-- TODO -->
