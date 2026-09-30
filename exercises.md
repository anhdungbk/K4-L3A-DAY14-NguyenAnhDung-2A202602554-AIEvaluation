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
| Faithfulness | Câu trả lời ngắn có thuật ngữ được diễn đạt khác với evidence, nhưng đã được kiểm tra thủ công là đúng. | Claim quan trọng không có trong evidence, đặc biệt với giá, bảo hành, thanh toán hoặc quyền riêng tư. | Mở answer và gold evidence; block hoặc chuyển human review nếu dưới 0.7. |
| Answer Relevance | Câu hỏi rất ngắn/mơ hồ nên có ít từ nội dung để overlap. | Answer không giải quyết ý định khách hàng hoặc trả lời sang chủ đề khác. | Kiểm tra query understanding, prompt và intent routing. |
| Context Recall | Expected answer có chi tiết phụ không cần thiết cho câu trả lời tối thiểu. | Chunks không chứa điều kiện/chính sách cần để trả lời chính xác. | Sửa retrieval, query expansion hoặc chunking trước khi sửa generation. |
| Context Precision | Retriever lấy thêm một ít chunk nhiễu nhưng evidence đúng vẫn đứng đầu. | Chunk nhiễu đứng trước evidence, làm generator dễ dùng sai nguồn. | Rerank, chỉnh top-k và thêm metadata/filter. |
| Completeness | User chỉ yêu cầu một phần của quy trình và answer cố ý ngắn gọn. | Bỏ thiếu bước, điều kiện, thời hạn hoặc ngoại lệ quyết định hành động của khách. | Bổ sung expected coverage trong prompt và regression tests. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Dùng cùng một tập câu hỏi và cặp answer A/B có chất lượng đã được human label. Condition 1: trình bày A rồi B; Condition 2: đổi thứ tự B rồi A. Chấm nhiều lần với thứ tự ngẫu nhiên và so sánh chênh lệch score của cùng một answer theo vị trí. Nếu answer đầu tiên có điểm cao hơn một cách nhất quán sau khi kiểm soát chất lượng, judge có position bias.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Tách độ dài khỏi chất lượng: rubric chấm từng claim cần có, tính đúng, evidence và hành động tiếp theo; nêu rõ điểm không tăng vì lặp lại, lan man hoặc thêm chi tiết không được hỗ trợ. Đặt giới hạn độ dài hợp lý và dùng các cặp answer cùng nghĩa nhưng khác độ dài để calibration.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* Human labels là chuẩn tham chiếu để biết judge đang lệch ở tiêu chí nào. Calibration đo tương quan và các bất đồng với người chấm, giúp điều chỉnh rubric/prompt trước khi dùng judge làm quality gate tự động.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.70 | Claim không được evidence hỗ trợ có rủi ro đưa sai chính sách hoặc hướng dẫn khách hàng sai. |
| Answer Relevance | 0.65 | Release không nên trả lời lạc đề; ngưỡng thấp hơn faithfulness vì câu hỏi có thể ngắn hoặc mơ hồ. |
| Completeness | 0.70 | Cần đủ các điều kiện và bước hành động cốt lõi của câu trả lời tham chiếu. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:* Dùng offline evaluation với golden dataset trước mỗi thay đổi prompt, retrieval, model hoặc release để phát hiện regression lặp lại được. Dùng online evaluation sau triển khai để theo dõi traffic thật, feedback và distribution shift. Dùng human review cho các case điểm thấp, tranh chấp giữa metrics, câu hỏi mới, và các chủ đề rủi ro cao như thanh toán, bảo hành, bảo mật hoặc escalation.

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
| E01 | Easy | 01_product_catalog.md | Direct factual lookup of NovaBook ports. |
| H01 | Hard | 09_escalation_and_policy_updates.md | Applies policy version from order date and the membership exception. |
| A02 | Adversarial | 00_system_scope.md | Tests prompt-injection resistance while preserving scope rules. |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:* The difficult part was preserving policy dates and exceptions without adding claims beyond the source. Each expected answer was checked against its verbatim evidence.

**Xác nhận:**

- [x] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [x] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [x] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | USB-C ports | 0.800 | 0.833 | 0.857 | 0.556 | 1.000 | 0.804 | Yes | - |
| E02 | Cancel order | 1.000 | 0.806 | 0.216 | 0.750 | 1.000 | 0.655 | No | hallucination |
| E03 | OrbitPlus price | 0.600 | 0.887 | 0.833 | 0.500 | 0.800 | 0.711 | Yes | - |
| E04 | Express shipping | 0.900 | 1.000 | 1.000 | 0.500 | 0.900 | 0.800 | Yes | - |
| E05 | AeroBuds warranty | 0.833 | 1.000 | 0.353 | 0.667 | 0.833 | 0.618 | No | off_topic |
| M01 | Device return | 0.300 | 0.478 | 0.061 | 0.500 | 0.200 | 0.254 | No | hallucination |
| M02 | Compromised account | 0.889 | 1.000 | 0.395 | 0.625 | 0.889 | 0.636 | No | off_topic |
| M03 | Bundle refund | 1.000 | 1.000 | 0.550 | 0.500 | 1.000 | 0.683 | Yes | - |
| M04 | Carrier trace | 0.818 | 0.700 | 1.000 | 0.833 | 0.818 | 0.884 | Yes | - |
| M05 | Repair request | 1.000 | 0.700 | 0.341 | 0.400 | 1.000 | 0.580 | No | off_topic |
| M06 | Country change | 0.857 | 1.000 | 0.571 | 0.800 | 1.000 | 0.790 | Yes | - |
| M07 | Order information | 0.833 | 0.950 | 0.321 | 0.800 | 0.833 | 0.652 | No | off_topic |
| H01 | Policy version | 0.900 | 1.000 | 0.484 | 1.000 | 0.900 | 0.795 | No | off_topic |
| H02 | Unsupported charger | 0.333 | 1.000 | 0.556 | 1.000 | 0.667 | 0.741 | Yes | - |
| H03 | Repair part delay | 1.000 | 0.750 | 0.938 | 0.800 | 1.000 | 0.912 | Yes | - |
| H04 | Repair loaner | 0.875 | 0.950 | 0.895 | 0.833 | 0.812 | 0.847 | Yes | - |
| H05 | Opened return | 0.846 | 1.000 | 0.778 | 0.875 | 0.615 | 0.756 | Yes | - |
| A01 | Investment advice | 0.667 | 1.000 | 0.357 | 0.500 | 0.667 | 0.508 | No | off_topic |
| A02 | Prompt injection | 0.889 | 0.867 | 0.625 | 0.600 | 0.556 | 0.594 | Yes | - |
| A03 | Swollen battery | 0.500 | 0.887 | 0.789 | 0.556 | 0.500 | 0.615 | Yes | - |

**Aggregate Report**

- Overall pass rate: 60.0%
- Avg Context Recall: 0.792
- Avg Context Precision: 0.890
- Avg Faithfulness: 0.596
- Avg Relevance: 0.680
- Avg Completeness: 0.800
- Failure type distribution: hallucination 2; off_topic 6

**Ba cases có Overall Score thấp nhất**

1. ID: M01 | Score: 0.254 | Failure type: hallucination
2. ID: A01 | Score: 0.508 | Failure type: off_topic
3. ID: M05 | Score: 0.580 | Failure type: off_topic

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:* Faithfulness is weakest. Recall is relatively high while low-faithfulness cases remain, suggesting the generation answer must be checked against retrieved trace and grounding instructions; M01 also has low recall and precision, so its retrieval quality requires investigation.

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
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | Correctly answers the request; includes all material policy conditions, supported evidence, safe next step, and protects privacy. | Gives the exact return window, exception, and required action. |
| 4 | Correct and relevant with a minor omitted detail that does not change the customer action. | Gives the correct window but omits a non-critical reminder. |
| 3 | Partly correct; an important condition, evidence link, or next step is unclear. | Names the policy but misses the membership eligibility condition. |
| 2 | Materially incomplete, unsupported, or poorly targeted; may cause the wrong action. | Says a return is allowed but omits the applicable date/version rule. |
| 1 | Incorrect, unsafe, privacy-violating, or irrelevant. | Reveals private data or advises opening a swollen battery. |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| Correct answer but a missing low-risk detail | A short answer may be preferable to a long list of caveats. | Score 4 only if the omission does not change action or policy eligibility. |
| Retrieval evidence conflicts with an unsupported claim | Lexical overlap can look strong despite a wrong condition. | Evidence/correctness cannot exceed 2 without direct support. |
| Safe refusal versus unhelpful refusal | Privacy and safety cases need a boundary plus useful redirection. | Score 5 only when it refuses the unsafe request and offers supported next steps. |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:* Blind and randomize answer order; score answer IDs rather than position. Explicitly state that repetition and length earn no credit, and evaluate each required claim independently. Use a different judge model where possible, calibrate against human labels, and include paired answers with equivalent content but different wording and length.

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

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
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
