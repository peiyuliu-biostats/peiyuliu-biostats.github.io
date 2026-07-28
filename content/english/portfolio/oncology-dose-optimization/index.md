+++
title = "Oncology Dose Optimization"
credential = "R Shiny · early-phase oncology · dose optimization"
date = 2025-01-03
draft = false
math = true
type = "portfolio"
layout = "single"
hidesidebar = true
+++

{{< pf-section num="1" title="Motivation" >}}
The FDA's Project Optimus has moved early-phase oncology away from the maximum
tolerated dose toward the optimal biological dose — the dose that best balances
efficacy and toxicity, not simply the highest tolerable one. This shift has
produced a set of new designs in the statistical literature: BOIN12, PKBOIN-12,
TITE-PKBOIN-12, STEIN, and the exposure-response Clinical Utility Score. Each
lives in its own paper, with its own notation and its own reference
implementation, if any.

What was missing was a single place to actually use them. This platform brings
the five methods under one interface, so that a statistician can run OBD
selection — simulate a design's operating characteristics, or replay it on real
trial data — without reimplementing each method from its paper.

It also extends one of them. The Clinical Utility Score was formulated for binary
endpoints; here it is extended to continuous and time-to-event outcomes, so that
efficacy and safety measured on different scales can be combined in one score.
{{< /pf-section >}}

{{< pf-section num="2" title="Architecture" >}}
The app is three independent dashboards, each with its own isolated reactive
state, so one dashboard cannot disturb another. Within each, the pure computation
is separated from the Shiny interface: the `basic/` layer holds standalone
functions that can be tested on their own, while `panel/`, `data/`, and
`present/` handle the interface and reactive plumbing.

<div class="pf-tree"><pre>
functions/
├── boin/
│   ├── basic/      Pure design logic: boundaries, decision, OBD, PAVA, simulate     <span class="lbl">[computation]</span>
│   │               fun_boin_decision.R · fun_pkboin_pk.R · fun_tite_pkboin_simulate.R
│   ├── data/       Reactive state for the PK-BOIN12 dashboard                        <span class="lbl">[state]</span>
│   └── panel/      Shiny UI modules                                                  <span class="lbl">[interface]</span>
├── stein/
│   ├── basic/      STEIN boundaries, decision, OBD, PAVA                             <span class="lbl">[computation]</span>
│   │               fun_stein_boundaries.R · fun_stein_decision.R · fun_stein_obd.R
│   ├── data/       Reactive state for the STEIN dashboard                            <span class="lbl">[state]</span>
│   └── panel/      Shiny UI modules                                                  <span class="lbl">[interface]</span>
└── cus/
    ├── basic/      ER fitting, utility maps, CUS math                                <span class="lbl">[computation]</span>
    │               fun_survival_rmst.R · fun_continuous_ecdf_map.R · fun_logistic_fit_fast.R
    ├── data/       Reactive state + bootstrap (bootstrap_CUS_data.R)                 <span class="lbl">[state]</span>
    ├── individual/ Per-endpoint utility / ER modules                                 <span class="lbl">[interface]</span>
    ├── panel/      Dashboard panels                                                  <span class="lbl">[interface]</span>
    └── present/    Plotly figures and result tables                                  <span class="lbl">[interface]</span>
</pre></div>

The PK-BOIN12 suite is a graceful-degradation chain rather than three separate
code paths. TITE-PKBOIN-12 reduces to PKBOIN-12 when no outcomes are pending;
PKBOIN-12 reduces to BOIN12 when the PK criterion is switched off. The shared
`basic/` functions carry this: `fun_pkboin_pk.R` adds the PK admissibility layer
on top of the BOIN12 decision, and `fun_tite_pkboin_simulate.R` adds
time-to-event handling on top of that.

Each dashboard offers two modes: *Simulate*, to study a design's operating
characteristics under user-specified true dose-response curves over many
replications, and *Upload*, to replay a design on real cohort- or patient-level
data. The CUS dashboard ships with curated real-world oncology datasets
(loncastuximab tesirine, polatuzumab vedotin). Analysis is interactive
throughout — Plotly figures, decision tables, debounced controls.
{{< /pf-section >}}

{{< pf-section num="3" title="Walkthrough" >}}
Two paths show the two kinds of question the app answers.

**Scoring exposure-response with CUS.**

{{< pf-figure src="/portfolio/oncology-dose-optimization/wt-cus-endpoint.png" alt="CUS endpoint settings with continuous, binary, and survival endpoint types and their regression models" caption="Endpoints can be continuous, binary, or time-to-event in the same analysis. Here efficacy is continuous, one safety endpoint is binary, and another is survival — with a baseline hazard family and a restricted-mean-survival-time horizon. This is the extension of the Clinical Utility Score beyond its original binary formulation." >}}

{{< pf-figure src="/portfolio/oncology-dose-optimization/wt-cus-score.png" alt="Clinical Utility Score curve across exposure with the maximizing dose marked" caption="Each endpoint's exposure-response curve is scored to a unit-scale utility, and the utilities are combined by weight into a single Clinical Utility Score across exposure. The exposure that maximizes it is the recommended dose — here Max CUS = 0.211 at PK = 1.61. The aggregation is the weighted geometric mean shown in the toggle." >}}

**Designing a PK-BOIN12 trial.**

{{< pf-figure src="/portfolio/oncology-dose-optimization/wt-pkboin-design.png" alt="PK-BOIN12 design panel showing derived decision boundaries and the utility table" caption="The design panel derives the decision boundaries from the target toxicity, and shows the utility table over the four joint efficacy-toxicity outcomes that defines the rank-based desirability score." >}}

{{< pf-figure src="/portfolio/oncology-dose-optimization/wt-pkboin-flow.png" alt="PK-BOIN12 dose-assignment decision flow from cohort enrollment to OBD" caption="The dose-assignment logic is made explicit: enroll a cohort, update counts, apply the toxicity and desirability rules within the admissible set, and either stop for safety and futility or continue to the final OBD selection." >}}

{{< pf-figure src="/portfolio/oncology-dose-optimization/wt-pkboin-oc.png" alt="PK-BOIN12 operating characteristics with selection probability and allocation over replications" caption="Run over many replications, the design reports its operating characteristics: selection probability and mean allocation per dose, correct-OBD selection, overdose exposure, and early-stopping — the numbers a statistician needs to judge a design before using it." >}}
{{< /pf-section >}}

{{< pf-section num="4" title="Methods" >}}
The app implements each design as published; the equations below are the core of
each, with full derivations, decision rules, and operating characteristics in the
app's Methods Guide.

**Clinical Utility Score.** For endpoint $i$ an exposure-response model gives a
fitted rate, mapped to a unit-scale utility $U_i(x)$ — efficacy higher-is-better,
safety lower-is-better. With normalized weights $\sum_i w_i = 1$, the score is the
weighted geometric mean

$$ CUS(x) = \prod_{i=1}^{m} U_i(x)^{w_i} = \exp\!\Big\{\sum_{i=1}^{m} w_i \log U_i(x)\Big\}, $$

and the recommended exposure is $x^* = \arg\max_x CUS(x)$. Continuous endpoints
are mapped through their empirical CDF; time-to-event endpoints enter through
restricted mean survival time under a specified baseline hazard family.

**BOIN12 and its extensions.** For dose $d$ the utility posterior is
$U_d \mid \text{data} \sim \text{Beta}(1 + x_d,\, 1 + n_d - x_d)$, and the
rank-based desirability score is

$$ RDS_d = \Pr(U_d > u_b \mid \text{data}). $$

PKBOIN-12 adds a pharmacokinetic admissibility rule, excluding doses with strong
evidence of inadequate exposure below a PK cutoff; TITE-PKBOIN-12 replaces
pending outcomes with approximated-likelihood quasi-observations so decisions can
be made during ongoing accrual.

**STEIN.** STEIN derives toxicity boundaries and an efficacy cutoff from target
rates and anchors, assigns cohorts within a local admissible set, and selects the
OBD from monotonized toxicity and unimodal-isotonic efficacy estimates.
{{< /pf-section >}}

{{< pf-section num="5" title="References" >}}
- Cheng Y, Chu S, Pu J, et al. Exposure-response-based multiattribute clinical utility score framework to facilitate optimal dose selection for oncology drugs. *Journal of Clinical Oncology*, 2024.
- Lin R, Zhou Y, Yan F, Li D, Yuan Y. BOIN12: Bayesian optimal interval phase I/II trial design for utility-based dose finding in immunotherapy and targeted therapies. *JCO Precision Oncology*, 2020.
- Sun H, Tu J. PKBOIN-12: A Bayesian optimal interval phase I/II design incorporating pharmacokinetics outcomes to find the optimal biological dose. *Pharmaceutical Statistics*, 2025.
- Yuan Y, Lin R, Li D, et al. Time-to-event Bayesian optimal interval design to accelerate phase I trials. *Clinical Cancer Research*, 2018.
- Lin R, Yin G. STEIN: A simple toxicity and efficacy interval design for seamless phase I/II clinical trials. *Statistics in Medicine*, 2017.
{{< /pf-section >}}
