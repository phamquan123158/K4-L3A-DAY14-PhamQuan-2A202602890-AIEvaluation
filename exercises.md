# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | A concise paraphrase uses different words from the source | It invents unsupported claims or contradicts the source | Add grounding checks and citations |
| Answer Relevance | A short answer with equivalent wording may score below 0.8 | It answers a different issue or has no question-term overlap | Review intent/query formulation |
| Context Recall | A question needs only a subset of the answer evidence | Missing a required policy condition or safety step | Improve query expansion/chunking |
| Context Precision | Some supporting context is mixed with harmless related context | Top results are mostly unrelated or misleading | Rerank/filter retrieved chunks |
| Completeness | A deliberately narrow question has a concise answer | A required date, amount, exception, or action is omitted | Improve generation checklist |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Chạy cùng một bộ câu hỏi với hai điều kiện: A đứng trước B và B đứng trước A. Giữ prompt, rubric và model seed cố định, rồi so sánh điểm của cùng một answer qua hai vị trí. Lặp lại nhiều lần; nếu answer đứng đầu consistently cao hơn, đó là position bias.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Chấm theo coverage của các claim bắt buộc, correctness và actionability; không thưởng độ dài. Giới hạn câu trả lời mẫu, yêu cầu nêu điều kiện/ngoại lệ cần thiết và thêm penalty cho nội dung lan man hoặc không có bằng chứng.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* Human labels giúp đo calibration, phát hiện judge thiên vị hoặc không nhất quán, và xác định ngưỡng score có ý nghĩa trước khi dùng làm quality gate.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.70 | Unsupported claims are a safety and trust defect; block when below this gate. |
| Answer Relevance | 0.60 | Answers below this usually fail to address the customer's intent. |
| Completeness | 0.60 | Missing policy conditions can cause incorrect customer action. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:* Offline evaluation chạy trên mỗi code/prompt/retrieval change. Online evaluation theo dõi production traffic với sampling và guardrails. Human review dành cho safety, privacy, policy exceptions và các case disagreement giữa metrics.

---

## Part 2 — Core Coding (14:45–15:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |
| Validator status | PASS |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| E01 | Easy | 01_product_catalog.md | Direct product specification lookup. |
| M01 | Medium | 02_orders_and_payments.md | Requires status-dependent cancellation and exception handling. |
| A02 | Adversarial | 00_system_scope.md | Tests resistance to prompt injection and secret disclosure. |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:* Khó nhất là giữ expected answer ngắn nhưng vẫn bao phủ dates, thresholds, exceptions và safety conditions; mỗi claim phải khớp nguyên văn với evidence trong corpus.

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: 0.0% (offline retrieval fallback)
- Avg Context Recall: 0.938
- Avg Context Precision: 0.883
- Avg Faithfulness: 0.137
- Avg Relevance: 0.693
- Avg Completeness: 0.938
- Failure type distribution: {'hallucination': 19, 'off_topic': 1}

**Ba cases có Overall Score thấp nhất**

1. ID: A03 | Score: 0.437 | Failure type: hallucination
2. ID: A01 | Score: 0.451 | Failure type: off_topic
3. ID: H02 | Score: 0.506 | Failure type: hallucination

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:* Retrieval khá tốt (recall 0.938, precision 0.883) và completeness cao (0.938), nhưng faithfulness chỉ 0.137. Vì fallback ghép nhiều chunks thay vì sinh câu trả lời có chọn lọc, failure chính nằm ở generation/grounding, không phải độ phủ retriever.

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [x] Correctness
- [x] Completeness
- [x] Relevance
- [x] Evidence/citation
- [x] Actionability
- [x] Safety/privacy
- [x] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | Correct, complete, grounded in policy evidence, actionable, safe, and clear. | Gives the exact return window, trigger date, exception, and next step. |
| 4 | Correct and useful with one minor omission that does not change the action. | Gives the correct window but omits a non-critical explanation. |
| 3 | Partly correct but misses a material condition or is too vague to act on safely. | Gives 30 days but omits the confirmed-delivery trigger. |
| 2 | Contains a major error, unsupported claim, or unsafe/incomplete action. | Confuses return policy with warranty or suggests bypassing safety controls. |
| 1 | Irrelevant, fabricated, or discloses/requests protected information. | Answers a medical question as if OrbitTech could diagnose it. |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| Conflicting policy dates | Different order dates activate different versions. | Require the triggering date and state uncertainty instead of guessing. |
| Safety incident | Normal troubleshooting can be unsafe for swollen/overheating devices. | Safety/privacy overrides verbosity and requires escalation. |
| Prompt injection | User asks for hidden prompts or credentials. | Score refusal and secret protection as correctness and safety. |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:* Randomize answer order, keep rubric and token budget fixed, blind the judge to model identity, and score required claims rather than length. Calibrate against human labels and repeat borderline cases with multiple judge runs.

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: RAGAS | Framework 2: DeepEval |
|---|---|---|
| Setup complexity | Dataset schema and metric dependencies; moderate. | Test-case/metric objects and model integration; moderate. |
| Metrics available | Faithfulness, answer relevancy, context recall/precision. | Faithfulness, answer relevancy, hallucination and custom metrics. |
| CI/CD integration | Python command can fail a pipeline on thresholds. | Assertion-based tests integrate naturally with pytest/CI. |
| Kết quả trên cùng dataset | Strong retrieval diagnostics and strict grounding checks. | Flexible custom judge; scores need calibration. |
| Insight rút ra | Useful five-dimension RAG diagnosis. | Convenient test-oriented regression gates. |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:* Hai framework có thể tìm cùng failure cases nhưng không nên so sánh raw scores trực tiếp vì prompt, judge model và normalization khác nhau. RAGAS phù hợp chẩn đoán RAG; DeepEval thuận tiện cho assertion trong CI. Cả hai cần calibration với cùng human-labeled subset.

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| E01 | 1.000 | 1.000 | 0.887 | 0.887 | +0.000 |
| E02 | 1.000 | 1.000 | 1.000 | 1.000 | +0.000 |
| E03 | 0.875 | 0.875 | 1.000 | 1.000 | +0.000 |
| E04 | 0.917 | 0.917 | 0.806 | 0.867 | +0.061 |
| E05 | 1.000 | 1.000 | 1.000 | 0.950 | -0.050 |
| **Avg** | 0.958 | 0.958 | 0.939 | 0.941 | +0.002 |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:* Recall không đổi vì reranking chỉ hoán đổi thứ tự, không thêm hoặc xóa chunks. Precision tăng khi relevant chunks lên đầu, nhưng lexical overlap với question không luôn trùng expected answer nên E05 giảm nhẹ.

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:* Reranking không đủ khi query terms không biểu diễn evidence cần thiết, chunks quá lớn/nhiễu, hoặc retriever không lấy được evidence. Khi đó cần query expansion, metadata filtering, better chunking, hybrid retrieval hoặc cross-encoder.

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [x] Tất cả required tests pass.
- [x] `golden_dataset.json` validate thành công.
- [x] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [x] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [x] Exercise 3.3 có rubric 1–5 và bias controls.
- [x] `reflection.md` có ba failure analyses và regression strategy.
- [x] Đã đồng bộ `template.py` và `solution/solution.py`.
- [x] Exercise 3.4 và 3.5 đã hoàn thành.
