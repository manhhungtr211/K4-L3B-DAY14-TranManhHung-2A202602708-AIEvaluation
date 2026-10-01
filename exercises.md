# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 9:15–12:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 9:15–9:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (9:30–9:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric            | Acceptable Low Score Scenario                                                                                                                       | Critical Low Score Scenario                                                                                                                                  | Action Required                                                                                                                                           |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Faithfulness      | Câu trả lời xã giao, chào hỏi (chitchat) hoặc giải thích định nghĩa thuật ngữ chung không nằm trong context nhưng vô hại.        | Bịa đặt (Hallucination) về chính sách bảo hành, hoàn tiền, giá bán hoặc thông số kỹ thuật của OrbitTech gây tranh chấp/thiệt hại.      | Thắt chặt system prompt ("chỉ trả lời dựa trên context"), hạ temperature = 0, thêm Hallucination Guardrail hoặc fallback khi thiếu dữ liệu.  |
| Answer Relevance  | Bổ sung thêm các lưu ý/hướng dẫn hữu ích liên quan (proactive tips) khiến tỷ lệ từ khóa trùng khớp với câu hỏi bị pha loãng. | Trả lời hoàn toàn lạc đề (off-topic), nhầm lẫn sang sản phẩm/dịch vụ khác, không giải quyết vấn đề khách đang hỏi.                    | Cải thiện Intent Detection/Query Router, yêu cầu LLM trả lời trực diện câu hỏi trọng tâm trước khi đưa thêm thông tin phụ.             |
| Context Recall    | Câu hỏi đơn giản, câu trả lời chỉ cần một phần nhỏ context là đủ (không cần retrieve toàn bộ tài liệu về chủ đề đó).    | Câu hỏi tổng hợp/so sánh nhiều chính sách nhưng retriever bỏ sót tài liệu mấu chốt dẫn đến câu trả lời thiếu điều kiện quan trọng. | Tăng top_k retrieval, áp dụng Hybrid Search (Dense Semantic + BM25 Lexical), tối ưu chunking hoặc mở rộng query (HyDE/Query Expansion).           |
| Context Precision | Hệ thống lấy nhiều chunks dự phòng (high recall focus) và Generator vẫn lọc đúng thông tin mà không bị nhiễu.                       | Các chunk chứa thông tin đúng nằm ở cuối bảng xếp hạng hoặc có quá nhiều chunk rác/mâu thuẫn làm LLM bị phân tâm/ảo giác.            | Tích hợp Reranker (Cross-encoder / Rerank by overlap), tinh chỉnh similarity threshold, chia nhỏ chunk size có tính tập trung cao hơn.            |
| Completeness      | Câu hỏi mở và bot tóm tắt ngắn gọn các ý chính kèm lời gợi ý khách hỏi chi tiết.                                                  | Bỏ sót các bước bắt buộc trong quy trình đổi trả, thiếu điều kiện bảo hành, hotline hỗ trợ khiến khách hàng thực hiện sai.           | Bổ sung hướng dẫn Step-by-Step vào prompt, thêm Few-shot examples chuẩn mực cho các câu hỏi quy trình, kiểm tra coverage qua LLM-as-a-Judge. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
>
> - **Condition 1 (Thứ tự gốc):** Đưa prompt đánh giá dạng Pairwise Comparison với thứ tự `[Candidate A, Candidate B]` vào LLM Judge.
> - **Condition 2 (Đảo vị trí):** Hoán đổi vị trí thành `[Candidate B, Candidate A]` và đưa vào cùng một LLM Judge với cùng tham số (temperature=0).
> - **Thực thi & Đánh giá:** Chạy trên tập benchmark gồm N cặp câu trả lời (N<50). Tính tỷ lệ nhất quán P(WinA|Pos1) so với P(WinA|Pos2). Nếu vị trí thứ nhất (hoặc thứ hai) có tỷ lệ thắng áp đảo bất kể chất lượng nội dung (> 55-60%), hệ thống đã bị Position Bias. Giải pháp khắc phục là chấm điểm 2 chiều (swapped pair scoring) và lấy điểm trung bình hoặc chỉ công nhận kết quả khi cả hai lượt đều đồng nhất.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
>
> - **Tách biệt tiêu chí:** Tách rõ ràng tiêu chí "Đầy đủ thông tin (Completeness/Coverage)" và "Độ súc tích (Conciseness)".
> - **Định lượng dựa trên Fact/Claim:** Đánh giá dựa trên số lượng luận điểm/thông tin cốt lõi (key factual points) được giải quyết, không chấm dựa trên độ dài hay sự mượt mà của câu chữ.
> - **Phạt thông tin thừa:** Thêm quy định rõ trong Rubric: "Không cộng điểm cho câu trả lời dài nếu chứa nội dung lặp lại, sáo rỗng hoặc ngoài lề. Trừ điểm nếu câu trả lời lan man làm che khuất ý chính".
> - **Đặt giới hạn độ dài lý tưởng:** Cung cấp độ dài mong muốn trong prompt (ví dụ: "câu trả lời chuẩn từ 2-4 câu / dưới 100 từ").
> - **Mô tả ví dụ mẫu:** Đưa mẫu cho Judge thấy: một câu trả lời chỉ 2 dòng nhưng đầy đủ thông tin vẫn đạt 5/5 điểm.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
>
> - LLM Judge có các thiên kiến tiềm ẩn (self-preference, leniency bias khi luôn cho điểm cao, severity bias khi quá khắt khe) và không nắm bắt được toàn bộ ngữ cảnh đặc thù của doanh nghiệp.
> - Calibrate với nhãn của con người (Human Ground Truth) thông qua các hệ số tương quan (Cohen's Kappa, Pearson/Spearman correlation):
>   1. Đảm bảo tiêu chuẩn chấm của LLM Judge tiệm cận với kỳ vọng của chuyên gia nghiệp vụ thực tế (Human Alignment).
>   2. Giúp tinh chỉnh ngưỡng threshold chấm điểm và hoàn thiện prompt rubric, hạn chế tối đa False Positives (cho qua các lỗi nghiêm trọng) và False Negatives (bắt lỗi oan các câu trả lời đạt chuẩn).

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric           | Threshold | Lý do                                                                                                                                                                                                                  |
| ---------------- | --------: | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Faithfulness     |      0.85 | Trong hệ thống CSKH, hallucination về chính sách bảo hành, hoàn tiền hoặc giá bán gây rủi ro pháp lý và thiệt hại tài chính trực tiếp cho OrbitTech. Đây là quality gate nghiêm ngặt nhất. |
| Answer Relevance |      0.75 | Đảm bảo câu trả lời trực diện vào thắc mắc của khách hàng, tránh trả lời vòng vo, sai ngữ cảnh làm giảm chất lượng trải nghiệm (bad UX).                                                     |
| Completeness     |      0.70 | Cần đủ các bước chính và thông tin bắt buộc; có thể chấp nhận threshold mềm dẻo hơn một chút vì khách hàng có thể hỏi tiếp (multi-turn) nếu cần thêm chi tiết phụ.                      |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
>
> - **Offline Evaluation (CI/CD Quality Gate):** Dùng trong môi trường Dev/Staging, mỗi khi tạo Pull Request, cập nhật prompt, đổi LLM/embedding model, hoặc sửa đổi retriever. Thực hiện đánh giá tự động trên Golden Dataset cố định để phát hiện regression nhanh với chi phí thấp trước khi release.
> - **Online Evaluation (Production Monitoring):** Dùng khi hệ thống đang phục vụ người dùng thật. Đánh giá liên tục qua tín hiệu tương tác thực tế (tỷ lệ Thumbs Up/Down, CSAT, tỷ lệ escalate cho nhân viên), lấy mẫu ngẫu nhiên production logs để LLM-as-a-Judge chấm điểm, và theo dõi latency, token cost.
> - **Human Review (Expert Audit):** Dùng định kỳ (hàng tuần/tháng) hoặc can thiệp khi: (1) Kiểm tra các case người dùng phản hồi tiêu cực (negative feedback / escalations); (2) Rà soát và cập nhật mở rộng Golden Dataset; (3) Calibrate lại LLM Judge; (4) Đánh giá các rủi ro an toàn và các cuộc tấn công đối kháng (adversarial test cases).

---

## Part 2 — Core Coding (9:45–10:40)

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

## Part 3 — Golden Dataset & Real Benchmark (10:40–11:35)

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

### Exercise 3.4 — Framework Comparison (Bonus +5)

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

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
