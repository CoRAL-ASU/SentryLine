# SentryLine: System Description for EMNLP Demonstration

*CoRAL Lab, Arizona State University*

This document describes the current implementation of SentryLine, an uncertainty-aware clinical question-answering system that grounds oncology answers in living (versioned) ASCO clinical practice guidelines. It is written to be adapted directly into the System Description / Architecture section of an EMNLP system demonstration paper. All details below reflect the actual, current codebase, verified against the running implementation rather than design documents.

---

## 1. Overview

SentryLive takes a natural-language clinical question, retrieves the most relevant passages from an indexed corpus of ASCO oncology guidelines using a **vectorless, hierarchical retrieval method** (PageIndex, rather than embedding similarity search), and returns a **role-specific** answer (clinician or patient) with:

- inline, page-level citations,
- an independent **factual verification report**,
- an independent **temporal verification report** (is the answer using the most recent guideline version?),
- inline **drift-detection notes** when a recommendation has changed across guideline versions,
- and an interactive evidence panel that opens the cited PDF and highlights the exact paragraph an answer was grounded in.

The system is built as a seven-step pipeline orchestrated behind a FastAPI backend, with a browser-based chat frontend. Two of the seven steps are currently stubs reserved for a planned production upgrade (see §7).

---

## 2. Pipeline Architecture

The orchestrator (`pipeline/orchestrator.py`) runs the following steps in sequence for every query:

### 2.1 Semantic answer cache (pre-pipeline short-circuit)

Before invoking the pipeline, standalone queries (no active multi-turn context, no forced refresh) are checked against a **clinician-curated answer cache**: a Jaccard-similarity match over stop-word-filtered content tokens (threshold 0.38), scoped by role. Clinicians can explicitly save an answer as a canonical response after review; a subsequent near-duplicate query (≥0.95 token overlap) updates that entry in place rather than creating a duplicate. This gives clinicians a lightweight mechanism to curate a trusted answer set over time without retraining anything.

### 2.2 Step 1 — Concept Extraction

A Gemini call extracts structured clinical entities from the raw query — `treatment`, `cancer_type`, `biomarker`, `treatment_line`, `population` — into a typed schema. Because this is free-text JSON prompting (Gemini has no native structured-output guarantee in this configuration), any field can come back as a list instead of a scalar string when a query mentions more than one value for that slot (e.g., multiple drug names). A validator coerces list-valued fields into a comma-joined string before the schema is accepted, so this class of malformed response can no longer cause a pipeline failure regardless of which field triggers it.

### 2.3 Step 2A — Guideline Retrieval (vectorless RAG)

Retrieval is performed by a Gemini agent with access to the **PageIndex MCP tool**, which builds a hierarchical tree index of each guideline PDF and uses LLM reasoning to navigate it, rather than chunking documents into embeddings and doing nearest-neighbor search. The retrieval agent is instructed to append a citation tag `<doc=FILENAME;page=N>` immediately after every passage it retrieves, using the literal, verbatim document identifier returned by the tool call — never a shortened or invented slug, and never a tag for content it did not actually retrieve.

For multi-turn conversations, the query passed to retrieval is enriched with the previous turn's question and a truncated summary of the previous answer, so follow-up questions retrieve with adequate context (e.g., "What about the biomarker eligibility for X?" resolves correctly).

**Known limitation (see §7):** retrieval is not scoped to the locally uploaded document set at the MCP tool-call level - the agent selects and names documents itself. This is mitigated post-hoc by a citation-grounding guardrail (§3.4) rather than prevented at the source.

### 2.4 Step 2C — Dual Memory Retrieval (stub)

Reserved for short-term (last-N-turns) and long-term (persistent clinician-corrected) memory via Redis. Currently returns empty lists; multi-turn coherence is instead handled by passing full conversation history directly into generation (§2.6) and by the correction-storage mechanism in §2.7.

### 2.5 Step 3 — Context Merge

Combines the retrieval stream and memory stream into one context object with an explicit priority order: **clinician corrections > retrieved guideline passages > short-term history**. For clinician-role queries, the merged context is further augmented with two pieces of ground truth derived from upload-time metadata rather than left to the model to infer:

- a guideline version note stating the full range of years available and which is most recent, so the model is explicitly told to prioritize it;
- an explicit list of which specific documents are **confirmed guideline updates** (see §3.5), so drift annotation is grounded in a verified signal rather than purely inferred from passage wording.

### 2.6 Step 5 — Role Routing

Two hand-authored prompt templates - one for clinicians, one for patients - share the same context/question slots but diverge sharply in response requirements:

- **Clinician template:** requires GRADE evidence levels, trial citations, biomarker eligibility criteria, and dosing/contraindication context where available; requires a citation `[Author et al. Year, p.N]` after every factual claim; and instructs the model to actively cross-reference recommendations across all guideline years present in context, adding an inline drift note (`↳ Updated [year]: ...`) whenever a recommendation demonstrably changed between versions - with an explicit instruction never to fabricate a "previous standard" the passages don't state.
- **Patient template:** plain, non-clinical language; empathetic but not patronizing tone; redirects to a care team when appropriate, framed as a helpful next step rather than a dead end.

Both templates instruct the model to answer only from the provided context and to explicitly say the information isn't available rather than guessing - this instruction is soft (prompt-level) and is independently checked, not enforced, by the verification layer in §3.

### 2.7 Step 6 — Answer Generation

The routed prompt, prepended with full conversation history, is sent to Gemini at a low, configurable temperature (default 0.1) for consistent, deterministic clinical output. A token-level confidence-scoring module exists but is currently dormant: the deployed Gemini model does not expose log-probabilities, so the two independent verifier verdicts (§3) serve as the system's operative confidence signal instead of a numeric score.

Conversation history longer than three turns is automatically compressed: the orchestrator replaces all prior turns with a single Gemini-generated summary plus the most recent turn, keeping the context window bounded in long sessions without losing clinical continuity.

### 2.8 Step 7 — Clinician Feedback Storage

Clinicians can submit a free-text correction after any answer. Corrections are stored as timestamped JSON records and reloaded at the start of every subsequent clinician-role query, injected into the merged context (§2.5) so future similar questions can incorporate prior corrections. The current implementation loads all stored corrections unconditionally regardless of relevance to the new query - acceptable at prototype scale, flagged for embedding-based relevance filtering at production scale.

---

## 3. Grounding and Safety Verification Layer

This is the layer most relevant to a demonstration of *trustworthy* clinical QA - it runs independently of generation and is designed so that no single failure mode (fabricated content, a wrong citation, a stale recommendation) can pass through silently.

### 3.1 Factual Verifier

A second, independent Gemini call re-checks every factual claim in the generated answer against the retrieved guideline passages, explicitly instructed to **ignore whatever citation the answer attributes to a claim** and judge the claim against passage text alone - this specifically prevents a confident-looking but fabricated citation from causing the claim itself to be treated as verified. Returns an overall verdict (`VERIFIED` / `PARTIALLY_VERIFIED` / `UNVERIFIED`), a confidence level, and a list of any unsupported claims.

### 3.2 Page-Level Citation Accuracy Check

An extension to the factual verifier: for every distinct `(document, page)` citation in the answer, the actual on-disk PDF page is opened and its real text extracted, independent of whatever the retrieval stage's passage blob claims - since retrieval can itself carry forward a page-number error. The verifier is asked a second, separate question per citation: is the claim actually supported by what is really printed on *that specific page*, not merely supported somewhere in the guideline in general? If a citation's page number doesn't hold up, an otherwise - `VERIFIED` answer is downgraded to `PARTIALLY_VERIFIED` - a wrong page number is treated as a real defect (it breaks the evidence-navigation guarantee described in §4, and wastes a clinician's time locating the actual source) even when the underlying clinical claim is accurate.

This check was added after an empirical failure was traced during development: a generated answer correctly and accurately described a real clinical trial finding, cited to the correct document but the wrong page (page 6 instead of page 3), and the existing factual verifier passed it at `VERIFIED, HIGH confidence` - because its design explicitly ignores citation attribution when judging content. The page-level check closes this specific gap without weakening the original content check.

### 3.3 Temporal Verifier

A third, independent Gemini call checks specifically whether the answer prioritizes the most recent available guideline version, using only the year metadata and passages actually retrieved for the query. Any claim flagged as outdated must be accompanied by an **exact quoted excerpt from a superseding passage** - a flag without a verbatim supporting quote is discarded rather than surfaced, which is the primary anti-hallucination guardrail for this verifier (it cannot rely on the model's own training knowledge of guideline history to justify a flag).

### 3.4 Citation Grounding Guardrail

After both verifiers run (while citation tags are still in their raw `<doc=filename;page=N>` form), a citation-formatting stage converts them into human-readable references like `[Author et al. 2024, p.2]`. This stage additionally cross-checks every citation - across all of the different raw citation dialects the system has observed Gemini produce - against the set of documents actually uploaded to the corpus. Any citation that does not correspond to a real, uploaded document is rendered as a distinct, visibly flagged form (`[⚠ Author et al. Year, p.N - unverifiable source]`) rather than being formatted identically to a genuine citation; the frontend renders this as a non-clickable red warning badge rather than the normal clickable blue evidence link.

This guardrail exists because of the retrieval-scoping limitation noted in §2.3 and §7: during development, a real user-facing exchange produced a fluent, well-formatted citation to a document ("Harrigan et al. 2024") that corresponded to no PDF anywhere in the uploaded corpus - a plausible real ASCO publication the model likely knew about from pretraining, surfaced as if it had been retrieved. The guardrail does not prevent retrieval from reaching outside the corpus; it ensures that when it does, the result is visibly distinguishable from a grounded citation rather than indistinguishable from one.

### 3.5 Drift Detection

Two complementary mechanisms:

- **Upload-time, document-level:** each guideline PDF is scanned (first three pages) for explicit ASCO update language ("guideline update," "focused update," "rapid recommendation update," etc.) at ingestion time. The resulting flag is surfaced two ways at query time: as an explicit "confirmed updates" list injected into the clinician generation prompt (§2.5), and as an "Updated guideline" badge on the corresponding evidence card in the UI.
- **Inline, per-recommendation:** the clinician prompt template instructs the model to actively compare recommendations across all guideline years present in the retrieved context and annotate a specific, documented change with a distinctly styled `↳ Updated [year]: ...` note directly beneath the affected recommendation - with an explicit instruction that a drift note must never be added unless the change is explicitly evidenced in the passages, and never fabricated when a "previous standard" isn't documented.

---

## 4. Evidence Grounding and Interactive PDF Highlighting

A distinguishing demo feature: every citation an answer produces is parsed into a structured evidence item (document, page, ranked candidate passage text, human-readable citation key, topic label, upload-time update flag, and a direct link to the source PDF). These accumulate in a persistent "Evidence Sources" panel across the *entire conversation*, not just the latest turn, deduplicated by document+page.

Clicking "View in PDF" on an evidence card opens an embedded PDF.js viewer that does not merely jump to the cited page - it locates and highlights the **exact paragraph** the answer was grounded in. This is done by normalizing both the target page's live text layer and the candidate passage text down to a bare, lowercase alphanumeric stream (Unicode-normalized so ligatures, curly quotes, and mathematical symbols don't break matching), locating the passage within that stream with an exact-match → head/tail-anchor → progressively-shorter-anchor fallback chain (robust to minor PDF text-extraction noise such as line-break hyphenation), and painting highlight rectangles per matched line rather than one coarse bounding box - so highlighting survives multi-column layouts and partial-line matches.

---

## 5. System Architecture

The backend is a FastAPI application. Each conversation is persisted to disk as JSON (query, answer, per-turn evidence, factual verdict, temporal verdict, timestamp), with a separately maintained, de-duplicated cumulative evidence list per conversation so the evidence panel reflects everything cited across the whole session. The frontend is a single-page browser chat client with role toggling (Clinician/Patient), a conversation history sidebar, an expandable verification-report panel (click a verdict badge to see the full independent verifier report rather than a one-line tooltip), and the PDF evidence viewer described in §4.

---

## 6. Comparison Systems

For evaluation, the project implements several baseline retrieval strategies against the same guideline corpus and query set, spanning a spectrum from no retrieval to standard embedding-based RAG:

- **Direct prompting** — all guideline PDFs loaded directly into the model's context window, no retrieval step.
- **Naive / sparse / dense RAG** — standard chunk-and-embed retrieval (dense uses OpenAI `text-embedding-ada-002` with cosine similarity).
- **Hybrid RAG** — combines sparse and dense retrieval.
- **Agentic RAG** — an LLM-driven retrieval agent without the PageIndex hierarchical index.

These serve as the comparison points for the vectorless, hierarchical-index retrieval (PageIndex) SentryLive uses in place of embedding similarity search.

---

## 7. Evaluation Methodology

The evaluation set is a 405-conversation clinical RAG benchmark spanning breast and prostate cancer guidelines, stratified across four question categories (Factual, Reasoning, Contrasting/Temporal, Role-Specific) and both target roles, generated via an LLM pipeline (base Q&A generation via Gemini in Google AI Studio, multi-turn expansion to three-turn conversations via Amazon Bedrock batch inference), and split 40% train / 40% test / 20% validation, with the validation split annotated by clinicians.

A CLI evaluation runner executes the pipeline (or a baseline) over an annotation sheet and scores each response with an LLM judge against clinician-authored gold answers on a multi-dimensional rubric: accuracy (1–3), grounding (0–1), safety (0–1), role-appropriateness (1–3), temporal accuracy (0–1), multi-turn context retention (0–1), and coherence (1–3).


---

*Prepared from the current SentryLine implementation, CoRAL Lab, Arizona State University.*
