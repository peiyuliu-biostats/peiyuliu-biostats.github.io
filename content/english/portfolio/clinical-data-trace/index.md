+++
title = "Clinical Data Trace"
credential = "Python · CDISC · lineage"
date = 2025-01-01
draft = false
math = true
type = "portfolio"
layout = "single"
hidesidebar = true
+++

{{< pf-section num="1" title="Motivation" >}}
At Bristol Myers Squibb, much of an early-career statistician's work is not only
study design and interim analysis. A large part is FDA submission — making the
data right before it goes out. One quiet piece of that work is a simple question
that is easy to ask and hard to keep answering: does the ADaM spec still say
what the SAP says?

The gap is there. A programmer could reuse a template from the last study.
The SAP moves to a new version, and the spec does not catch up. The check is
manual, and at 11pm before a database lock a tired reviewer can miss a small
thing. Nothing here is unsolved in principle. It is unsolved in practice, one
small difference at a time.

I did not think AI should make this decision. So before deciding what the tool
would do, I decided what it would not do. It does not judge which document is
right. It does something narrower and safer: it traces any analysis value back
to its source, and it puts the spec next to the SAP and points at where they
differ — without ever claiming to know which one is correct.
{{< /pf-section >}}

{{< pf-section num="2" title="Architecture" >}}
The system is a lineage graph over CDISC datasets. A node is one addressed
variable. An edge is a derivation dependency. A trace is a traversal. The study,
the graph, and the values come from fixed functions, with no model in the path.
The model is admitted only where the task is reading prose, and there its output
type is constrained by construction.

<div class="pf-tree"><pre>
src/trace_agent/
├── synth/       Deterministic RAW→SDTM→ADaM derivation; truth exported from metadata     <span class="lbl">[deterministic]</span>
├── graph/       Node/Edge model, rule-based parser, DAG builder + trace traversal        <span class="lbl">[deterministic]</span>
├── data/        Read-only DuckDB store, resolver, filtered-branch divergence replay      <span class="lbl">[deterministic]</span>
├── alignment/   Rule-based scope + pairing; model flags wording, cannot state a verdict   <span class="lbl">[agentic]</span>
├── ask/         BM25 retrieval; answers gated on an exact-substring citation              <span class="lbl">[agentic]</span>
├── narrative/   Clinician text generated only from a resolved trace                       <span class="lbl">[agentic]</span>
├── codegen/     R/SAS scaffolds from trace structure, for human review only               <span class="lbl">[agentic]</span>
├── decisions/   Append-only decision log with an explicit state machine                   <span class="lbl">[mixed]</span>
├── coverage/    Two independent coverage statistics over the graph                        <span class="lbl">[deterministic]</span>
└── llm/         Optional client + shared-key quota; no import-time network activity        <span class="lbl">[boundary]</span>
</pre></div>

<a class="pf-repo-link" href="https://github.com/peiyuliu-biostats/agent1-CDISC-datasets-trace" target="_blank" rel="noopener">Full file-level layout in the repo →</a>

**Data structure**
A node is a frozen triple `Node(layer, dataset, variable)`, with `layer` in
`{RAW, SDTM, ADaM}`. Structure and value come from separate evidence. The
lineage structure is read from `DATASET.VARIABLE` references in the
specification. The value is computed by a restricted AST evaluator over the
arithmetic in that same specification. Both paths are deterministic, and neither
calls a model. An `Edge` records one dependency and the filter context it
carries, such as a population or parameter restriction along that hop. Each edge
is tagged `explicit`, `fallback`, or `llm`, so an inferred edge is never mixed
in silently. The nodes and edges form a directed acyclic graph from ADaM down to
RAW.

**Deterministic construction**
The study is three functions composed in order: `generate_raw(seed=48)`, then
`derive_sdtm`, then `derive_adam`, each a map of `dict[str, DataFrame]`. The same
derivation metadata that produces the values also generates the specification
text and the lineage edges. The truth artifacts are exported from that metadata
without importing the graph builder, the parser, the value resolver, or the
divergence detector they are used to test. An earlier hand-labeled truth file was
removed for this reason. A maintained key can drift from the derivation it is
meant to describe. An exported one cannot.

**Trace traversal**
`trace(node)` walks toward RAW and carries the filter context up each edge, so a
value is followed only along the branch that produced it. Three terminal states
are made explicit, not hidden. A RAW node, or an `assigned` origin, closes a
branch as `complete`. A missing upstream, or a filter that conflicts with an
edge's own filter, marks it `partial`. A node already seen on the current path is
cut as a `cycle`:

{{< pf-code lang="python" >}}
if current in path:
    saw_cycle = True
    return TraceNode(
        node=current,
        filters=dict(filters),
        break_reason="cycle detected",
        spec_text=text,
        incoming_confidence=incoming_confidence,
        incoming_origin=incoming_origin,
    )
{{< /pf-code >}}

A cycle does not crash and does not loop. It becomes a labeled node in the tree,
and the trace returns a status of `complete`, `partial`, or `cycle`.

**Bound on the model**
In `alignment/`, scope and pairing are rule-based. An ADaM derivation is matched
to a fixed SAP section by variable name and section number, so the same inputs
always pair the same way. Only then does the model read the two passages, and it
returns a proposed wording difference, or nothing. It cannot return a verdict.
The response schema admits a boolean and neutral descriptive fields only, a
banned-term filter rewrites any evaluative word, and every quoted span must be an
exact substring of the source passage or it is dropped. The categorical
judgment — `intended`, `discrepancy`, or `equivalent` — is entered by a reviewer
and written to `decisions/`. The same shape governs `ask/`, `narrative/`, and
`codegen/`.
{{< /pf-section >}}

{{< pf-section num="3" title="Walkthrough" >}}
*This is one investigation of a single value, from the SAP check to the recorded
decision. The subject is real study data.*

**The spec-to-SAP check**

The first screen is not a trace. It is a check that runs before any single value
is in doubt. The tool pairs each ADaM specification derivation with the SAP
passage it should follow, and reports where the wording no longer lines up.

{{< pf-figure src="/portfolio/clinical-data-trace/wt-sap-check.png" alt="SAP alignment table checking ADaM specification derivations against SAP passages" caption="Ten key analysis variables are checked against the SAP; fourteen structural variables carry no SAP definition and are left unchecked rather than assumed correct. The model marks passages that read differently and asks a human to confirm. It does not rule on which document is right." >}}

This is the quiet part of submission work made visible: not fixing an error
after it surfaces, but seeing where the specification has drifted from the SAP
before a database lock.

**Tracing one value**

From the overview, one number is pulled down to the raw record. Subject 1048,
`ADTTE.AVAL = 4.2053` months of progression-free survival. The trace resolves in
seven hops, complete to RAW, and a banner states at once whether anything
diverged along the way.

{{< pf-figure src="/portfolio/clinical-data-trace/wt-trace-banner.png" alt="Trace result header showing ADTTE.AVAL equals 4.2053 in seven hops with a divergence banner" caption="The value resolves through seven hops down to the raw randomization date. The banner at the top reports the outcome before the reader scrolls: one divergence, at RS.RSDTC, between the investigator and the independent assessor. The signal and the evidence below it come from the same deterministic pass." >}}

The divergence is not a data error. It is two readers of the same scans.

{{< pf-figure src="/portfolio/clinical-data-trace/wt-trace-rs-rows.png" alt="SDTM RS records for one subject showing investigator and independent assessor rows" caption="At the SDTM RS record, the same subject carries two evaluations. The investigator recorded progression on 22 May (green, the traced main chain); the independent assessor recorded it earlier, on 3 May (amber). This evaluator discordance is a routine feature of oncology endpoints, not a mistake to be removed." >}}

{{< pf-figure src="/portfolio/clinical-data-trace/wt-divergence-review.png" alt="Divergence review showing the independent-assessor value recomputing to 3.6 months across nine of sixty subjects" caption="The tool quantifies the alternative: under the independent assessor the value recomputes to 3.6 months, and nine of sixty subjects show the same pattern. It also states plainly that the two branches use different row-selection rules, so this is not a like-for-like comparison. It measures the difference; it does not declare a winner." >}}

**Recording the decision**

The Authorisation panel is honest about a gap: no SAP section is linked to this
variable, so the specification documents how the value is derived but not who
authorised it. The reviewer supplies that judgment, and the tool records it.

{{< pf-figure src="/portfolio/clinical-data-trace/wt-decision-log.png" alt="Decision log entry with author, rationale, and timestamp for the traced value" caption="The reviewer confirms the main chain follows the investigator read per the ADaM spec, and that the independent-assessor branch is an expected sensitivity analysis, not a discrepancy to resolve. The decision is saved with author, rationale, and timestamp, and cannot be created except from a trace. The tool presents the difference; the person owns the verdict." >}}

**Two other entry points**

The investigation above can begin from plain language, and it can be widened to
the whole study. Asking "PD" returns two distinct meanings — protocol deviation
and progressive disease — each tied to its own source document and dataset,
answered only where a citation supports it. Trace coverage steps back from the
single value: of forty-seven variables, 57.5% of specification rows parse into
traceable links and 55.3% trace end-to-end to raw. The tool does not inflate
either figure — real specifications are mostly prose, and the two numbers are
reported as the different claims they are.

{{< pf-figure src="/portfolio/clinical-data-trace/wt-ask-coverage.png" alt="Ask study documents returning two distinct cited meanings for the abbreviation PD" caption="Natural-language questions stay anchored to sources, and coverage is reported without rounding away what prose leaves untraceable." >}}
{{< /pf-section >}}

{{< pf-section num="4" title="Reliability" >}}
**Read-only access**

Every data access is read-only, and every access is logged. There is no
free-text query box. A query is generated from the lineage graph and the node
the reviewer clicked, and each one is written to an audit trail with a timestamp.
A reviewer can see which rows were read to answer a question, and when.

**Offline core**

The trace, the values, and the divergence detection do not call a model. They are
the parts a submission must be able to defend, so they run in deterministic code
that returns the same answer with or without an API key. The model is used only
for reading prose. If it is unavailable, the narrative and the wording flags
disappear, and the core result is unchanged.

**Reproducible build**

The synthetic study, its specifications, and its lineage truth are generated
together from one derivation, so the answer key cannot drift from the derivation
it describes. The environment is pinned with a Dockerfile, a Makefile, pinned
dependencies, and continuous integration, so the same inputs produce the same
outputs on another machine. This is what makes the result auditable and
repeatable.

**Graceful degradation**

The optional model features share a small quota, ten calls per session and three
hundred per day. When the quota is spent, the tool does not crash. It falls back
to its offline path, and the core trace and divergence still work. A tool a
reviewer depends on before a database lock has to stay usable when an external
service is not.

Stack, for reference: Streamlit · DuckDB · Pydantic.
{{< /pf-section >}}

{{< pf-section num="5" title="Methods" >}}
**Layered derivation**

The study is built in layers. Writing $R$ for the raw records, $S$ for SDTM, and
$A$ for ADaM,

$$ S = f_{\text{SDTM}}(R), \qquad A = f_{\text{ADaM}}(S) = (f_{\text{ADaM}} \circ f_{\text{SDTM}})(R). $$

The lineage of a value $a$ is its dependency closure, the raw records it depends
on:

$$ L(a) = \{\, r \in R : a \text{ depends on } r \text{ under } f_{\text{ADaM}} \circ f_{\text{SDTM}} \,\}. $$

Because $f$ is deterministic, $L(a)$ is fixed by the code. The ground truth is
induced by the derivation, not written by hand. An earlier hand-labelled truth
file was removed for this reason: a maintained key can drift, an induced one
cannot.

A divergence is the same derivation under a different branch condition:

$$ f_{\text{ADaM}}(S \mid \varphi) \neq f_{\text{ADaM}}(S \mid \varphi'). $$

For subject 1048, $\varphi = \text{INVESTIGATOR}$ gives 4.2053 months and
$\varphi' = \text{INDEPENDENT ASSESSOR}$ gives 3.5811. Both are real
recomputations.

**Bound on the model**

Pairing is deterministic. A rule $\pi$ maps a variable to its two passages,
$\pi : v \mapsto (d_v, s_v)$, by variable name and section number, so the same
inputs always pair the same way. The model then computes only

$$ \text{flag}(d_v, s_v) \in \{\text{possible difference},\ \text{none}\}. $$

It cannot return a verdict. The categorical judgment

$$ \delta_v \in \{\text{intended},\ \text{discrepancy},\ \text{equivalent}\} $$

is entered by a reviewer and written to the decision log. Reading prose is where
a model is strong. The regulatory verdict is not the model's to give.

**Coverage**

Coverage is reported as two independent numbers, because they answer different
questions:

$$ \text{parse} = \frac{\#\{\text{spec rows resolved to an edge}\}}{\#\{\text{spec rows}\}}, \qquad \text{lineage} = \frac{\#\{v : L(v)\text{ reaches } R\}}{\#\{\text{variables}\}}. $$

By construction $\text{lineage} \le \text{parse}$: a row can parse cleanly and
still break several hops upstream. For this study, parse is 57.5% (27 of 47) and
lineage 55.3% (26 of 47). The figure is meant to be well under 100%. Real
specifications are mostly prose, and unparsed rows are kept verbatim as partial,
never guessed.

**Tests**

The tests assert properties, not numbers: the truth file is generated rather than
hand-written, divergence recall is complete on the seeded cases, the core runs
with no API key, and the data layer is read-only.

Full derivations and run instructions are in the repository.
{{< /pf-section >}}

{{< pf-section num="6" title="Acknowledgments" >}}
This tool grew out of conversations across several teams at Bristol Myers
Squibb — Statistical Methodology & Data Science, Clinical, SAS Programming, and
Drug Development. I am grateful for the questions they asked. They shaped what
this tool does, and just as much, what it deliberately does not do.
{{< /pf-section >}}
