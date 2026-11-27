# MP1 Prompt Lab: reflection

Ten job snippets, four prompting strategies, gpt-4o-mini at temperature 0, scored by exact match against the golden set and by a gpt-4o judge on a 1–4 scale. The comparison table is in the notebook (Step 5). The numbers below are from my final run, after fixing the structured prompt. I mention the earlier runs where they taught me something.

## 1. Which strategy won, and on what dimension?

Few-shot, narrowly, and on reliability more than on the headline number. It tied the fixed structured prompt on accuracy (2.8 out of 3) and had the only perfect judge score (4.0). It was also the only strategy that read j05's "fresh grads welcome" as 0 years rather than null, and it did that in every run I scored. CoT cost the most ($0.00103 for ten calls, against $0.00059 for few-shot and $0.00035 for zero-shot), was the slowest, and didn't score higher. Every strategy parsed and passed schema validation 100% in the final run.

## 2. What surprised me?

Structured went from 0 to 2.8 over two fixes, and both bugs were mine. First, I pasted `JobListing.model_json_schema()` into an f-string. That inserts the Python repr of a dict, so the model answered in Python syntax (single quotes, `None`) and nothing parsed. After `json.dumps()` every reply parsed, but half of them were the schema itself, with a "value" key added next to each "type". The model couldn't tell a description of the shape from a template to fill in. One sentence fixed it: return an object that conforms to the schema, and don't repeat the schema. Both times, what the prompt *showed* beat what it *asked for*. It also taught me that valid JSON isn't the same as usable output, so I added a schema-valid check next to parse rate.

Two smaller surprises. On j05, CoT's notes say "the minimum experience required is 0 years" while its answer field says null. The answer key comes before the notes in the output, so the reasoning arrived too late to change it. And the zero-shot prompt never changed, yet on j09 it returned the string "3-5" in my first run and 3 in later ones. Temperature 0 doesn't promise the same answer twice.

All seven disagreements between exact match and the judge are "Ltd." against the golden "Ltd". The judge gave each of them a 4. My exact-match rule is stricter than it should be.

## 3. Which strategy would I reach for first in my capstone?

Few-shot. My domain is financial crime compliance on public regulatory texts. There a wrong answer becomes a finding, and conventions decide correctness: what counts as a threshold, how to read "at least", when to say "not stated". j05 is that problem in miniature. "None required" (0) and "not stated" (null) are different answers, and only worked examples got the model to tell them apart. In production I'd add constrained JSON output, so that parsing is never the failure.

## 4. What would I try with another day?

Run each strategy three to five times and report the spread, because one run can't separate 2.7 from 2.8. Normalise trailing punctuation in the exact-match scorer. Give structured and CoT one example of "none required" to see whether that alone closes the j05 gap. Then add harder snippets (ranges, "or equivalent", several roles in one post) before trusting any ranking.
