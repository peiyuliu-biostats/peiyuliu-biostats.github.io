+++
title = "vaxineR"
credential = "R · published package · epidemiology"
date = 2025-01-02
draft = false
math = true
type = "portfolio"
layout = "single"
+++

{{< pf-section num="1" title="Motivation" >}}
In September 2025, Florida proposed removing school-entry vaccine requirements. A
change like this is expected to lower childhood vaccine coverage, but the size of
the effect on disease risk is not obvious from the coverage number alone. The
relationship between coverage and outbreak risk is non-linear: below the
herd-immunity threshold, a small further drop in coverage produces a large rise
in expected infections.

vaxineR was written to make that relationship explicit for a specific setting —
Florida kindergartens. It takes county vaccination coverage and a disease, and
returns the epidemiological quantities that matter for outbreak risk: the
effective reproduction number, the probability of an outbreak, and the expected
number of infections. The aim is not to predict any single event, but to show
how risk moves as coverage moves.
{{< /pf-section >}}

{{< pf-section num="2" title="The package" >}}
vaxineR is a published R package (GPL-3, installable from GitHub). It provides
functions to compute the key metrics, generate summary tables, and produce the
plots below, and it ships with cleaned historical vaccination data for Florida
counties (2016–2024). The data behind any plot can be exported to CSV or Excel
for further analysis.

Two views carry most of the message.

{{< pf-figure src="/portfolio/vaxiner/wt-risk-curve.png" alt="Expected measles infections versus vaccination coverage, showing a non-linear tipping point below the herd-immunity threshold" caption="The expected number of infections in a school of 200, plotted against coverage, for measles ($R_0 = 15$). The curve is flat near full coverage and steepens sharply once coverage falls below the 93.5% herd-immunity line. This is the non-linear tipping point: the same one-point drop in coverage costs far more below the threshold than above it." >}}

{{< pf-figure src="/portfolio/vaxiner/wt-history.png" alt="Historical kindergarten vaccination coverage for Florida and two counties, trending downward below the 95% target" caption="Historical kindergarten coverage for Florida and two large counties, 2016 onward. The state average and both counties have fallen below the 95% target, and Broward drops steeply in the most recent year. The package uses real county data, not illustrative numbers." >}}

Beyond measles, the package models pertussis and chickenpox, and a custom
disease can be modelled by supplying its $R_0$.
{{< /pf-section >}}

{{< pf-section num="3" title="Methods" >}}
The model is a simple, homogeneous-mixing transmission model for an outbreak
seeded by a single case in a school. Writing $p$ for coverage and $VE$ for
vaccine efficacy, the susceptible proportion is

$$ s = 1 - p \cdot VE, $$

and the effective reproduction number is

$$ R_e = R_0 \, s. $$

From $R_e$ the package computes the probability of a major outbreak,

$$ p_{\text{major}} = 1 - \frac{1}{R_e}, \qquad R_e > 1, $$

and the expected outbreak size from the final-size equation, solving

$$ 1 - z = e^{-R_e z} $$

for the attack rate $z$. For measles the package uses $R_0 = 15$ and $VE = 0.97$,
which require coverage above roughly 93.5% to hold $R_e < 1$. At the 2024 Florida
statewide average of 89.8%, this gives $R_e = 1.93$, an 85.5% probability of at
least one secondary case, and a 48.3% probability of a major outbreak. If
coverage fell to 85%, the expected outbreak in a school of 200 rises to about 32
infections.

The model assumes homogeneous mixing and does not include interventions such as
case isolation or contact tracing, so its numbers describe a worst-case outbreak
rather than a forecast.
{{< /pf-section >}}

{{< pf-section num="4" title="Context" >}}
vaxineR grew out of a vaccine-policy assessment prepared by the University of
Florida College of Medicine Faculty Council, evaluating the potential effect of
declining vaccine coverage on disease burden in Florida. The measles
transmission analysis used in that work is the same model implemented in the
package. The work was carried out with the group of Matt Hitchings and Ira
Longini at UF.

The package makes that analysis reproducible and reusable: the equations, the
Florida data, and the plots are packaged so that others can run the same
calculations for their own coverage figures.
{{< /pf-section >}}
