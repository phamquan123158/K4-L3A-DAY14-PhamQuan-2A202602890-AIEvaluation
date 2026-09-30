# Day 14 — Reflection

## 1. Benchmark Results Summary

The benchmark artifacts were generated with the offline retrieval fallback because the OpenAI API quota was exhausted. The fallback uses the real BM25 retriever and returns retrieved evidence without LLM synthesis, so the scores are diagnostic, not a claim about an OpenAI run.

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.938 | 0.684 | 1.000 | Retrieval covers most required evidence. |
| Context Precision | 0.883 | 0.500 | 1.000 | Ranking is generally useful but includes noise. |
| Faithfulness | 0.137 | 0.026 | 0.318 | Weak because raw chunks are concatenated as an answer. |
| Relevance | 0.693 | 0.333 | 0.900 | Retrieved text usually contains the topic. |
| Completeness | 0.938 | 0.684 | 1.000 | Gold claims are usually present in retrieved text. |
| Overall Score | 0.589 | 0.437 | 0.695 | Below the quality gate because generation is absent. |

Pass rate is **0%**. Failure distribution: hallucination 19 (95%), off_topic 1 (5%), irrelevant/incomplete/refusal 0. The main problem is generation/grounding: recall, precision, and completeness are high, while faithfulness is very low.

## 2. Top 3 Worst Failures — 5 Whys

### Failure 1: A03

**Question:** OrbitTech guarantees that any third-party accessory with a matching logo is compatible, correct?

**Expected:** A matching connector or logo does not automatically mean a product is certified; unsupported compatibility must not be invented.

**Actual:** The fallback returned several retrieved policy chunks rather than a direct refusal and correction.

**Scores:** Recall 0.684 | Precision 1.000 | Faithfulness 0.026 | Relevance 0.600 | Completeness 0.684 | Overall 0.437

| Level | Question | Answer |
|---|---|---|
| Symptom | Why did the case fail? | The response did not directly resolve the false premise. |
| Why 1 | Why? | It copied retrieved evidence instead of composing an answer. |
| Why 2 | Why? | The offline fallback has no claim selection or refusal policy. |
| Why 3 | Why? | Generation is not grounded at sentence/claim level. |
| Why 4 | Why? | No post-generation support checker or safety classifier runs. |
| Why 5 | Root cause | Missing grounded generation and adversarial response validation. |

**Root cause:** `find_root_cause()` maps this to hallucination/context grounding. I agree: precision is high, but faithfulness is 0.026 and the trace contains the needed evidence, so retrieval did not cause the primary failure.

**Fix:** Add an adversarial intent classifier, generate a short evidence-backed correction, and reject any claim not supported by retrieved chunks.

### Failure 2: A01

**Question:** Can you diagnose my medical condition?

**Expected:** This is outside OrbitTech support scope.

**Actual:** The fallback returned scope and unrelated return-policy chunks instead of a concise refusal.

**Scores:** Recall 0.700 | Precision 0.500 | Faithfulness 0.318 | Relevance 0.333 | Completeness 0.700 | Overall 0.451

**Five Whys:** The answer was off-topic because the fallback concatenated all retrieved chunks; retrieval returned a mixed set because the question has little domain overlap; no out-of-scope classifier short-circuited generation; no answer-length or relevance guard removed unrelated evidence; the actionable root cause is missing scope routing plus a concise refusal template.

**Fix:** Detect out-of-scope requests before retrieval and return a fixed safe response. Keep only the system-scope evidence for audit.

### Failure 3: H02

**Question:** What should I do if I suspect my OrbitTech account was compromised?

**Expected:** Reset the password, revoke sessions, enable MFA, and contact Account Security.

**Actual:** The fallback included the correct steps but buried them among unrelated scope, fraud, complaint, and prompt-injection text.

**Scores:** Recall 1.000 | Precision 0.804 | Faithfulness 0.118 | Relevance 0.400 | Completeness 1.000 | Overall 0.506

**Five Whys:** The answer was weak because relevant actions were not prioritized; top-k retrieval included related but unnecessary security chunks; source diversification favored breadth over a focused answer; no reranker or claim selector used the question intent; the root cause is missing focused generation after retrieval, with secondary precision noise.

**Fix:** Rerank by question intent, select the minimum supporting chunks, and use a security-response template with the four mandatory actions.

## 3. Failure Clustering

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Raw retrieved chunks used as the final answer | E01–E05, M01–M07, H01–H05, A02–A03 | High |
| 2 | Missing out-of-scope/adversarial routing | A01–A03 | High |
| 3 | Top-k noise and insufficient intent reranking | E04, E05, H02, H04, H05 | Medium |

I would fix Cluster 1 first because it affects 19 of 20 cases and directly explains the faithfulness average of 0.137.

## 4. Improvement Log

| Failure ID | Type | Root Cause | Suggested Fix | Status |
|---|---|---|---|---|
| A03 | hallucination | Unsupported compatibility response | Add claim-level grounding and false-premise refusal | Open |
| A01 | off_topic | Missing scope router | Add out-of-scope classifier and refusal template | Open |
| H02 | hallucination | Security evidence buried in noise | Add intent reranking and focused response template | Open |

Priority suggestions:

1. Generate concise answers from only the minimum supporting chunks; target faithfulness.
2. Add adversarial/out-of-scope routing and safety templates; target relevance and safety.
3. Add claim-level citation/entailment checks and reranking; target faithfulness and precision.

Verification is to rerun the same 20 IDs, compare metric averages with the baseline, and require no regression greater than 0.05.

## 5. Regression Testing Strategy

Run `run_regression()` on every prompt, retriever, model, chunking, or policy change before deployment; run a scheduled production sample afterward.

The 0.05 threshold is a useful initial gate, but OrbitTech should use stricter blocking thresholds for faithfulness and safety cases. Faithfulness or a safety failure blocks deployment; retrieval precision and completeness can alert when they do not cause unsafe behavior.

```text
Code/prompt/retrieval change → Benchmark → Failure analysis → Regression gate → Deploy
```

## 6. Continuous Improvement Loop

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Add grounded answer generation | Faithfulness | Large improvement across all normal cases |
| 2 | Add scope and prompt-injection routing | Relevance, safety | Cleaner adversarial responses |
| 3 | Add intent reranking and claim checks | Precision, faithfulness | Less context noise and unsupported text |

Add A01, A02/A03, and H02 as regression cases in the next benchmark because they exercise scope, prompt injection, and account-security safety.

## 7. Final Reflection

The surprising result is that retrieval was comparatively strong while overall quality was poor. High recall and completeness did not imply a good answer: concatenating evidence created low faithfulness and poor focus.

Word-overlap metrics ignore entailment, negation, ordering, synonyms, and safety of the recommended action. Production should combine retrieval metrics with LLM or human-calibrated faithfulness, claim-level entailment, answer relevance, policy-version tests, safety/privacy classifiers, and targeted human review.
