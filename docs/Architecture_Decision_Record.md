# Capstone Architecture Decision Record

**Status:** Draft, updated after MP1 (Week 5, 27 Nov 2026)
**Author:** Gopalakrishnan Andavar
**Version:** 0.1

> All source material is public: FATF Recommendations, RBI Master Directions (KYC/AML),
> EU AML Directives and Wolfsberg Group papers. No employer or client data is used.

## Idea

A **regulatory compliance assistant for financial-crime analysts**. An analyst asks a
question such as "what does RBI require for periodic KYC updation of a high-risk customer?"
or pastes a screening alert narrative. The assistant then:

1. extracts the structured facts from the text (entity, jurisdiction, obligation,
   threshold, time period), and
2. answers from the public regulations, citing the exact clause it relied on.

In compliance a wrong answer is not just a bad user experience. It can become a regulatory
finding. The design therefore puts traceability and "say you don't know" ahead of fluency.

## Inputs

- Public regulatory PDFs: FATF Recommendations, RBI Master Directions, EU AMLD texts and
  Wolfsberg guidance. These are chunked and embedded for retrieval.
- Free-text analyst questions and synthetic (made-up) alert narratives.
- A small golden set of question → expected-fact/expected-clause pairs, built the same way
  as the MP1 golden set, including deliberate traps where the answer is "not stated".

## Tools

- LLM API: gpt-4o-mini for extraction and answering, and a stronger model only as judge
  (MP1 pattern).
- Async batch runner (asyncio, the OpenAI async client) for evaluation runs.
- Pydantic models as the single source of truth for every structured output.
- Vector store and retriever (course weeks to come; not decided yet).
- Python 3.12, `uv`, ruff, mypy (strict), pytest.

## Memory

- No long-term user memory in v1. Each query stands alone, which keeps answers reproducible.
- An evaluation cache: model outputs and judge scores are pickled per run (as in MP1), so
  re-scoring does not re-pay for calls and runs can be compared like for like. Caches are
  git-ignored.

## Outputs

- A JSON object that validates against a Pydantic schema: the extracted fields plus
  `answer`, `citations` (document, section/clause) and `confidence`.
- Any field the source does not state is returned as `null`, never guessed.
- A short human-readable answer rendered from that JSON.

## Autonomy Level

**Assistive (human in the loop).** The system suggests and cites; the analyst decides.
It never closes an alert, files a report or changes a customer's risk rating.

## Decision Boundaries

- **May:** retrieve, extract, summarise, cite and flag ambiguity.
- **Must not:** invent a threshold, date or obligation that is not in the retrieved text; give
  an answer without a citation; make a disposition decision on an alert.
- **Must escalate:** when the retrieved passages conflict, when no passage supports an answer,
  or when the extraction fails schema validation twice.

## Success Metrics

| Metric | Target (v1) | How measured |
|---|---|---|
| Field accuracy on golden set | ≥ 90% | Strict match plus a normalisation layer |
| Schema-valid outputs | 100% (after one retry) | Pydantic validation |
| Fabrication rate on "not stated" traps | 0 | Golden-set trap items (MP1's j10 pattern) |
| Citation present and correct | ≥ 95% | Judge plus spot-check |
| Judge vs deterministic agreement | Disagreements reviewed by hand | MP1 disagreement table pattern |
| Cost per query | Tracked per run | Token counts × rates |

---

## Decisions

### ADR-001 · Prompting strategy: few-shot first, CoT only where needed

- **Context.** MP1 compared four strategies on gpt-4o-mini at temperature 0 over 10 extraction
  snippets. The final run is below. Full numbers and the run history are in
  `docs/adr/0002-prompting-strategy.md`.
  - Few-shot: 2.8 / 3 accuracy, judge 4.0 / 4, 100% parse and schema-valid, ≈1.7× zero-shot cost.
  - Structured (after two prompt fixes): same 2.8 accuracy, judge 3.9, but ≈1.3× few-shot cost.
  - CoT: 2.7, ≈3× zero-shot cost and the slowest median latency (2.9 s).
  - Zero-shot: 2.7.
- **Decision.** Few-shot is the default for extraction, with 3–5 worked examples drawn from
  the compliance domain. At least one example must show a field that is genuinely absent
  (`null`), and one must show an explicit zero or "none required".
- **Why.** The examples taught the edge cases that rules alone did not. Only few-shot told
  "no experience required" (0) apart from "not stated" (null).
- **Consequences.** The examples are part of the prompt and must be versioned and reviewed
  like code. Each example adds tokens to every call.

### ADR-002 · Structured output is enforced by code, not by the prompt

- **Context.** The MP1 structured strategy failed twice, both times because of my prompt.
  1. The Pydantic schema went in as a Python dict repr, so the model answered in Python
     syntax (single quotes, `None`) and nothing parsed.
  2. After `json.dumps`, everything parsed, but 5 of 10 replies were the schema itself with
     a `"value"` key added next to each `"type"`. That was valid JSON and useless.
  3. Asking for "a JSON object that conforms to the schema … do not repeat the schema"
     fixed it, and structured reached 2.8 / 3.
- **Decision.**
  - Serialise schemas with `json.dumps` whenever they go into a prompt. A bare schema is
    ambiguous, so state that an instance is wanted or, better, show one filled-in example.
  - Use the API's JSON / structured-output mode for production calls.
  - Validate every response with Pydantic `model_validate(..., strict=True)`, and report a
    schema-valid rate alongside the parse rate.
  - Retry once on failure, then escalate.
- **Why.** The model mirrors what it is shown, not what it is told. "Parses as JSON" and
  "is usable" are different checks. Regex-based recovery is a fallback, not a design.

### ADR-003 · Chain-of-thought is gated and its answer is cross-checked

- **Context.** On MP1 j05 the CoT reasoning concluded "0 years", but the answer field said
  `null`. The answer key came before the notes in the output, so the reasoning could not
  influence it. Only the answer was scored.
- **Decision.**
  - Use CoT only for queries flagged as ambiguous, such as conflicting clauses or ranges.
  - Keep the reasoning in a separate field placed *before* the answer fields.
  - Add a consistency check between the reasoning and the final value.
  - Never show the reasoning to the analyst as if it were a citation.

### ADR-004 · "Not stated" is `null`, and fabrication is a hard failure

- **Context.** MP1 j10 ("no specific years requirement") tested fabrication. Every strategy
  that parsed returned `null`, helped by the explicit rule "do not invent values".
- **Decision.**
  - Every prompt carries an explicit "return null if not stated; do not infer" rule.
  - Every golden set includes trap items.
  - A fabricated value counts as a failure whatever the other fields score.

### ADR-005 · Evaluation: deterministic scoring plus an LLM judge, with disagreements reviewed

- **Context.** In MP1's final run the judge and the exact-match scorer disagreed on 7 of 40
  rows. All seven were a trailing full stop ("Ltd." vs "Ltd"), where my normalisation was too
  strict. In an earlier run, zero-shot returned the range on j09 as the string `"3-5"`, and
  as 3 in later runs with the same prompt. A single run at temperature 0 is not
  reproducible.
- **Decision.**
  - Keep both scores.
  - Normalise punctuation and whitespace, and coerce numbers before matching.
  - Use the judge at temperature 0 with a written rubric, reasoning first and then the score.
  - Review every disagreement by hand before trusting either score.
  - Judge with a stronger model than the one under test.
  - Report schema validity as a separate rate.
  - Base any comparison between strategies on at least three runs, not one.

### ADR-006 · Async batching and result caching for every evaluation run

- **Context.** In MP1, 40 calls took 3.6–6.1 s wall time against about 107 s of summed
  latency once run with `asyncio.gather`.
- **Decision.**
  - Run all evaluation calls concurrently, relying on the client's built-in retries.
  - Cache raw results and scores per run.
  - Record cost and p50 latency per strategy, so model and prompt changes are compared on
    cost as well as quality.

### Open questions

- Done: structured was re-run with both fixes and matched few-shot on accuracy (2.8) but
  not on cost or on j05, so ADR-001 holds.
- Does a single "none required" example close structured's and CoT's j05 gap?
- Does few-shot still win on longer, messier regulatory text, where retrieval adds noise?
  Check this with 10 domain snippets (the MP1 "personal calibration" step).
- Measure judge variance (3 runs, median) before setting hard targets on judge scores.
