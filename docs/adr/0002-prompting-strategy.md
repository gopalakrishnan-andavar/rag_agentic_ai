# ADR 0002: Prompting strategy for structured extraction

- **Status:** Accepted (MP1, 27 Nov 2026). To be revisited once the capstone use case in `docs/Architecture_Decision_Record.md` is confirmed.
- **Evidence:** `mp1/mp1_prompt_lab.ipynb`. Four runs of 40 calls on 27 Nov, across three versions of the structured prompt, gpt-4o-mini at temperature 0, gpt-4o judge (1–4). The table shows the final run, after both fixes to the structured prompt.

## Context

MP1 extracted three fields (company, role, minimum years of experience) from ten job snippets and compared four prompts: zero-shot, few-shot, structured (persona plus schema) and chain-of-thought. The task is a small version of what my capstone needs. It pulls structured, checkable fields out of unstructured text, and "not stated" must come back as null rather than a guess.

My capstone sits in financial crime compliance and works only on public regulatory documents (FATF Recommendations, RBI Master Directions, EU AML texts, Wolfsberg papers). The fields it will extract are things like a threshold, a deadline, an obligation or the entity an obligation applies to. In this domain a fabricated value is worse than a missing one, because a wrong answer becomes a compliance finding.

## Results that drove the decision

Final run:

| Strategy | Accuracy (of 3) | Parse rate | Schema-valid | Judge | Cost ($, 10 calls) | Latency p50 (s) |
|---|---|---|---|---|---|---|
| Zero-shot | 2.7 | 1.0 | 1.0 | 3.9 | 0.00035 | 2.7 |
| Few-shot | 2.8 | 1.0 | 1.0 | 4.0 | 0.00059 | 2.5 |
| Structured | 2.8 | 1.0 | 1.0 | 3.9 | 0.00077 | 2.5 |
| Chain-of-thought | 2.7 | 1.0 | 1.0 | 3.9 | 0.00103 | 2.9 |

How structured got there:

| Version | Structured prompt | Parsed | Usable answers | What went wrong |
|---|---|---|---|---|
| 1 | Schema inserted as a Python dict | 0/10 | 0/10 | The f-string pasted the dict's repr (single quotes, `None`), so the model answered in Python syntax |
| 2 | Schema serialised with `json.dumps` | 10/10 | 5/10 | 5 replies echoed the schema with a `"value"` added to each property, instead of giving an answer |
| 3 | Adds "return an object that conforms to the schema… do not repeat the schema" | 10/10 | 10/10 | Only the shared j05 and "Ltd." misses |

Other observations:
- Few-shot was the only strategy that read "fresh grads welcome" (j05) as 0, and it did so in every run that could still be checked. The other strategies returned null.
- On j05, chain-of-thought's notes said "0 years" while its answer said null. The answer key comes before the notes in its output, so the reasoning arrived too late to change the answer.
- Zero-shot returned j09 as the string "3-5" in the first run and as 3 in the final run, with an identical prompt. Temperature 0 does not make the output deterministic.
- All seven judge vs exact-match disagreements in the final run come from "Ltd." against the golden "Ltd".

## Decisions

1. **Few-shot is the default prompting strategy for extraction.** Three to five worked examples drawn from real public texts, including at least one where the right answer is null and one edge value such as 0. On this task it matched the best accuracy, was the only strategy to get the edge case right, and cost less than structured or CoT.
2. **Show the shape of the answer, not just its description.** Any schema or example in a prompt is serialised with `json.dumps`, never inserted as a Python object. A bare JSON Schema is ambiguous to a model, which may complete the schema instead of producing an instance. Either state that an instance is wanted or, better, show one filled-in example. The model copies what it sees.
3. **Validate the shape, not only the syntax.** Valid JSON is necessary but not sufficient. The echoed schemas in run 2 were valid JSON and useless. Every extraction is checked with `Model.model_validate(data, strict=True)`. Strict mode stops Pydantic quietly turning `"5"` into `5`, so a wrong type is caught rather than fixed silently.
4. **Use constrained output in production.** MP1 kept the format in the prompt so that parse rate measured the prompt. The capstone will use the API's structured outputs with a Pydantic model, which rules out both the syntax bug and the schema echo. Nullable fields are typed `X | None` so the model is never forced to invent a value.
5. **Reasoning comes before the answer, or not at all.** Chain-of-thought is kept for cases where it pays, such as multi-step interpretation of a clause, and there the reasoning field is placed before the answer field. It is not the default. Here it cost about three times as much as zero-shot and gained no accuracy.
6. **Every prompt states the null rule, and examples carry the edge conventions.** The null rule is "If the text does not state it, return null. Do not infer or guess." An instruction alone did not get "no experience required" to 0. Only an example did.
7. **Evaluation uses several scores and repeat runs.** Normalised exact match (case, whitespace and trailing punctuation) is the floor. Schema validity is reported as a separate rate. An LLM judge with a written rubric that penalises fabrication is the main quality score, and disagreements between the scores are reviewed by hand. Temperature 0 is not deterministic, so any comparison between strategies uses at least three runs. Results are cached so that each analysis refers to one run.

## Consequences

- Few-shot examples have to be curated and versioned alongside the golden set, and must not overlap with it.
- Few-shot prompts are longer, so each call costs about 1.7× zero-shot in MP1. That is acceptable at this volume.
- In the final run the strategies are only 0.1 apart on accuracy, which is within run-to-run noise. The real differences are on edge cases (j05), cost, and how robust a prompt is to formatting mistakes. Before this decision is locked for the capstone, it needs repeat runs and a golden set of at least 20 domain items.

## Alternatives considered

- **Zero-shot.** Cheapest, but it leaves types and conventions open: a string range in one run, and null for "no experience required".
- **Structured persona plus schema, without examples.** It reached few-shot's accuracy, but only after two prompt fixes and at a higher cost. It is fragile, because small details of how the format is presented decide whether you get an answer or an echoed schema.
- **Chain-of-thought as the default.** It gained no accuracy on this task, had the highest cost and latency, and its reasoning contradicted its own answer on j05.
