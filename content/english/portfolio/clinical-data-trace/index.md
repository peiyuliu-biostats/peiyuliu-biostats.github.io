+++
title = "Clinical Data Trace"
credential = "Python · CDISC · lineage"
date = 2025-01-01
draft = false
math = true
type = "portfolio"
layout = "single"
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
*How it is built, not what it does. The worked example is in §3.*

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

**The data structure.**
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

**Construction is deterministic, and the truth is exported, not labeled.**
The study is three functions composed in order: `generate_raw(seed=48)`, then
`derive_sdtm`, then `derive_adam`, each a map of `dict[str, DataFrame]`. The same
derivation metadata that produces the values also generates the specification
text and the lineage edges. The truth artifacts are exported from that metadata
without importing the graph builder, the parser, the value resolver, or the
divergence detector they are used to test. An earlier hand-labeled truth file was
removed for this reason. A maintained key can drift from the derivation it is
meant to describe. An exported one cannot.

**The trace is a filter-aware depth-first walk with an explicit cycle guard.**
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

**Where the model is admitted, its output type is fixed.**
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

<blockquote class="pf-spine">The graph, the values, the divergence — the parts that must be reproducible — never touch a model. The model only reads two passages and asks whether they still agree.</blockquote>
{{< /pf-section >}}

{{< pf-section num="3" title="Walkthrough" >}}
<!-- TODO: copy pending. One narrated investigation of subject 1042.
     Figures via pf-figure, captions = what the step REVEALS. -->
{{< /pf-section >}}

{{< pf-section num="4" title="Reliability" >}}
<!-- TODO: copy pending. Regulated-setting credibility, not tech stack. -->
{{< /pf-section >}}

{{< pf-section num="5" title="Methods" >}}
<!-- TODO: copy pending. Math via $…$ inline and pf-math for display blocks. -->
{{< /pf-section >}}

{{< pf-section num="6" title="Acknowledgments" >}}
This tool grew out of conversations across several teams at Bristol Myers
Squibb — Statistical Methodology & Data Science, Clinical, SAS Programming, and
Drug Development. I am grateful for the questions they asked. They shaped what
this tool does, and just as much, what it deliberately does not do.
{{< /pf-section >}}
