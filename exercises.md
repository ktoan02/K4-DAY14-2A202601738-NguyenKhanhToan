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

| Metric            | Acceptable Low Score Scenario                                                                                                             | Critical Low Score Scenario                                                                                                                          | Action Required                                                                                                    |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Faithfulness      | Assistant thêm các từ xã giao/lịch sự hoặc disclaimer chuẩn không có trong gold context nhưng không làm sai lệch thực tế. | Answer chứa thông tin sai sự thật (hallucination) về chính sách đổi trả, giá cả, hoặc thời hạn bảo hành.                            | Thắt chặt system prompt grounding, hạ temperature generation, thêm hallucination guardrails.                   |
| Answer Relevance  | Câu hỏi của user quá rộng/mơ hồ, assistant cần hỏi lại để làm rõ hoặc liệt kê nhiều phương án xử lý.               | Assistant trả lời lạc đề, không giải quyết đúng ý định (intent) của khách hàng (ví dụ: hỏi đổi trả lại trả lời bảo hành). | Cải thiện prompt instruction, tinh chỉnh Query Rewriting / Intent Classifier.                                   |
| Context Recall    | Câu hỏi out-of-scope / adversarial (không có context phù hợp) hoặc câu hỏi tra cứu đơn giản 1 ý.                            | Câu hỏi phức tạp (Hard/Medium) nhưng retriever bỏ sót các đoạn chứa điều kiện hoặc ngoại lệ quan trọng.                            | Tăng top_k retrieval, chuyển sang Hybrid Search (BM25 + Vector), tối ưu hóa chunk size/overlap.               |
| Context Precision | Thông tin liên quan xuất hiện ở rank 2 hoặc 3 thay vì rank 1 do có nhiều đoạn thông tin cùng chủ đề.                      | Chunks liên quan bị đẩy xuống cuối (dưới rank 5) hoặc các chunk nhiễu (noise) đứng top 1-2 gây loãng context.                         | Triển khai Reranker (Cross-Encoder / overlap reranking), tinh chỉnh tham số BM25/Vector search.                 |
| Completeness      | Nhu cầu user chỉ cần trả lời 1 phần nhỏ của chính sách, hoặc câu hỏi out-of-scope yêu cầu từ chối.                       | Answer bỏ sót các điều kiện bắt buộc, phí restocking, hoặc hạn định thời gian trong expected answer.                                   | Tăng Context Recall để retriever lấy đủ evidence; nhắc model liệt kê đầy đủ các bước/điều kiện. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
>
> - **Condition 1 (Original Order):** Đưa `Answer A` ở vị trí thứ 1 (Option A) và `Answer B` ở vị trí thứ 2 (Option B) vào prompt của LLM Judge. Thu thập điểm số và lựa chọn của Judge.
> - **Condition 2 (Swapped Order):** Đổi thứ tự, đưa `Answer B` lên vị trí thứ 1 và `Answer A` xuống vị trí thứ 2 với cùng prompt và rubric. Thu thập điểm số của Judge.
> - **Phân tích:** Tính tỷ lệ Positional Consistency. Nếu Judge luôn chấm điểm cao hơn cho câu trả lời nằm ở Vị trí 1 (bất kể là A hay B), hiện tượng Position Bias xuất hiện.
> - **Giải pháp giảm thiểu:** Chạy đánh giá ở cả 2 thứ tự rồi lấy trung bình (Pairwise Position Swapping) hoặc randomize vị trí các options.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
>
> - **Quy định tiêu chí ngắn gọn & đúng trọng tâm:** Đưa tiêu chí "Conciseness & Precision" vào rubric. Quy định rõ câu trả lời dài dòng, chứa từ thừa (fluff/padding) không làm tăng thông tin sẽ bị trừ điểm hoặc khống chế điểm tối đa (ví dụ: không quá 3/5 điểm nếu lan man).
> - **Đánh giá theo độ phủ ý (Fact/Claim-based scoring):** Đánh giá dựa trên tỷ lệ các ý chính (key points/claims) được đáp ứng thay vì độ dài văn bản.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
>
> - **Đảm bảo tính tin cậy và căn chỉnh miền kiến thức (Domain Alignment):** LLM Judge có thể bị quá nương tay (leniency bias), quá khắt khe (severity bias), hoặc hiểu sai tiêu chí kinh doanh riêng của doanh nghiệp.
> - **Đo lường độ tương quan (Correlation):** Cần so sánh điểm của LLM Judge với tập dữ liệu chuẩn do chuyên gia con người dán nhãn (Human Ground Truth) thông qua chỉ số như Pearson Correlation hoặc Cohen's Kappa. Khi đạt độ tương quan cao (ví dụ > 0.8), LLM Judge mới đủ độ tin cậy để tự động hóa trong CI/CD.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric           | Threshold | Lý do                                                                                                                       |
| ---------------- | --------: | ---------------------------------------------------------------------------------------------------------------------------- |
| Faithfulness     |      0.85 | Ngăn chặn hallucination trả lời thông tin sai cho khách hàng (gây rủi ro tài chính/pháp lý trực tiếp).        |
| Answer Relevance |      0.80 | Đảm bảo assistant luôn trả lời đúng trọng tâm câu hỏi của khách hàng, tránh gây ức chế cho người dùng. |
| Completeness     |      0.75 | Đảm bảo cung cấp đủ các điều kiện/quy trình chính, cho phép linh hoạt nhỏ về cách diễn đạt.              |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
>
> - **Offline evaluation:** Chạy tự động trong CI/CD pipeline trên Golden Dataset (ví dụ: 20-100 test cases) mỗi khi có thay đổi code, prompt hoặc logic retrieval trước khi merge/deploy bản mới.
> - **Online evaluation:** Monitor liên tục trên real user traffic ở môi trường production (dùng lightweight guardrails, feedback functions như TruLens/Langfuse, kết hợp thumbs up/down của user) để phát hiện trôi dữ liệu (data drift) hoặc lỗi phát sinh thực tế.
> - **Human review:** Đánh giá định kỳ mẫu ngẫu nhiên (sampling), soi kỹ các trường hợp điểm offline/online thấp, các ca tranh chấp/escalation cao, và dùng để định kỳ calibrate lại LLM Judge.

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

| Hạng mục                         | Kết quả   |
| ---------------------------------- | ----------- |
| Tổng số records                  | ____ / 20   |
| Easy                               | ____ / 5    |
| Medium                             | ____ / 7    |
| Hard                               | ____ / 5    |
| Adversarial                        | ____ / 3    |
| Source documents được sử dụng | ____ / 10   |
| Validator status                   | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
| -- | ---------- | ------------------ | --------------------------------------------------- |
|    |            |                    |                                                     |
|    |            |                    |                                                     |
|    |            |                    |                                                     |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

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

| ID  | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
| --- | ---------------- | ---------: | ------------: | -----------: | --------: | -----------: | ------: | ------- | ------------ |
| E01 |                  |            |               |              |           |              |         |         |              |
| E02 |                  |            |               |              |           |              |         |         |              |
| E03 |                  |            |               |              |           |              |         |         |              |
| E04 |                  |            |               |              |           |              |         |         |              |
| E05 |                  |            |               |              |           |              |         |         |              |
| M01 |                  |            |               |              |           |              |         |         |              |
| M02 |                  |            |               |              |           |              |         |         |              |
| M03 |                  |            |               |              |           |              |         |         |              |
| M04 |                  |            |               |              |           |              |         |         |              |
| M05 |                  |            |               |              |           |              |         |         |              |
| M06 |                  |            |               |              |           |              |         |         |              |
| M07 |                  |            |               |              |           |              |         |         |              |
| H01 |                  |            |               |              |           |              |         |         |              |
| H02 |                  |            |               |              |           |              |         |         |              |
| H03 |                  |            |               |              |           |              |         |         |              |
| H04 |                  |            |               |              |           |              |         |         |              |
| H05 |                  |            |               |              |           |              |         |         |              |
| A01 |                  |            |               |              |           |              |         |         |              |
| A02 |                  |            |               |              |           |              |         |         |              |
| A03 |                  |            |               |              |           |              |         |         |              |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
| ----: | -------------------------- | ---------------- |
|     5 |                            |                  |
|     4 |                            |                  |
|     3 |                            |                  |
|     2 |                            |                  |
|     1 |                            |                  |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
| --------- | -------------------- | ------------------------- |
|           |                      |                           |
|           |                      |                           |
|           |                      |                           |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí                    | Framework 1: ____ | Framework 2: ____ |
| ----------------------------- | ----------------- | ----------------- |
| Setup complexity              |                   |                   |
| Metrics available             |                   |                   |
| CI/CD integration             |                   |                   |
| Kết quả trên cùng dataset |                   |                   |
| Insight rút ra               |                   |                   |

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

| ID            | Recall before | Recall after | Precision before | Precision after | Delta Precision |
| ------------- | ------------: | -----------: | ---------------: | --------------: | --------------: |
|               |               |              |                  |                 |                 |
|               |               |              |                  |                 |                 |
|               |               |              |                  |                 |                 |
|               |               |              |                  |                 |                 |
|               |               |              |                  |                 |                 |
| **Avg** |               |              |                  |                 |                 |

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
