# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 55.0% (11/20 cases passed)

| Metric            | Average |    Min |    Max | Nhận xét                                                                                                                    |
| ----------------- | ------: | -----: | -----: | ----------------------------------------------------------------------------------------------------------------------------- |
| Context Recall    |  0.8722 | 0.2857 | 1.0000 | Retriever bao phủ tốt phần lớn context cần thiết cho 20 câu hỏi; chỉ bị tụt ở các câu hỏi bảo mật/phức hợp |
| Context Precision |  0.9529 | 0.7500 | 1.0000 | Rất cao; top chunks lấy về hầu như luôn chứa thông tin liên quan trực tiếp đến câu hỏi                         |
| Faithfulness      |  0.5794 | 0.1053 | 1.0000 | Thấp; generator hay diễn giải thêm điều kiện ngoại lệ hoặc dùng từ vựng khác với context                       |
| Relevance         |  0.6241 | 0.4286 | 0.8333 | Mức trung bình khá; câu trả lời bám sát chủ đề nhưng bị trừ điểm do khác biệt từ ngữ                      |
| Completeness      |  0.6863 | 0.2857 | 1.0000 | Mức khá; một số ý nhánh trong câu hỏi phức hợp chưa được trả lời trọn vẹn                                   |
| Overall Score     |  0.6299 | 0.4033 | 0.8571 | Đạt mức trung bình khá (63.0%), cần tập trung tối ưu phần Generation và Faithfulness                               |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Context Precision (0.9529), Context Recall (0.8722)
- Metrics/cases ở mức Needs Work (0.6–0.8): Completeness (0.6863), Relevance (0.6241), Overall Score (0.6299)
- Metrics/cases ở mức Significant Issues (<0.6): Faithfulness (0.5794)

**Failure type distribution**

| Failure Type  | Count | Percentage |
| ------------- | ----: | ---------: |
| hallucination |     3 |      15.0% |
| irrelevant    |     0 |       0.0% |
| incomplete    |     0 |       0.0% |
| off_topic     |     6 |      30.0% |
| refusal       |     0 |       0.0% |

*Ghi chú:* Heuristic evaluator không tự gắn nhãn `refusal` (được nhóm vào off_topic/hallucination). Tỷ lệ lỗi trên tổng số failure cases (9 cases) là: off_topic (66.7%), hallucination (33.3%).

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> *Câu trả lời:*
> Vấn đề chính của hệ thống hiện tại nằm ở **Generation (và giới hạn của heuristic evaluation)** chứ không phải Retrieval:
>
> 1. `Context Precision` (0.9529) và `Context Recall` (0.8722) đều ở mức rất cao (>0.85), chứng minh pipeline Retrieval hoạt động hiệu quả, truy xuất được hầu hết các tài liệu liên quan cho 20 câu hỏi benchmark.
> 2. Ngược lại, `Faithfulness` tụt xuống mức rất thấp (0.5794), cho thấy LLM thường xuyên đưa thêm thông tin ngoại lệ, không liên quan hoặc diễn giải bằng từ ngữ tự nhiên không trùng lặp từng chữ với context nguồn, khiến thuật toán đo word-overlap phạt nặng.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> M07 — "What immediate actions should a customer take if they suspect their OrbitTech account has been compromised?"

**Expected answer:**

> "A customer who suspects account compromise should reset their password from a trusted device, revoke active sessions, enable multi-factor authentication, contact Account Security, and attempt cancellation of any unauthorized orders that are still in Confirmed status."

**Actual answer:**

> "If a customer suspects their OrbitTech account has been compromised, they should: - Escalate immediately without waiting for routine support, because active account compromise, suspected fraud, and immediate privacy disclosure may be escalated. - If suspected card fraud is involved, report it to both the card issuer and OrbitTech. - When reporting, include the order number, approximate event time, and a description of what happened. - Do not include passwords, authentication codes, full card numbers, or unnecessary identity documents. - If OrbitTech temporarily restricts account changes during an investigation, do not create repeated accounts to bypass the restriction, as this can delay identity verification."

**Scores:** Context Recall: 0.2857 | Context Precision: 0.8667 | Faithfulness: 0.1385 |
Relevance: 0.7857 | Completeness: 0.2857 | Overall: 0.4033 (Passed: False)

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> Retriever lấy 5 chunks nhưng chủ yếu từ `09_escalation_and_policy_updates.md` và `07_repair_and_technical_support.md` (hướng dẫn mở ticket leo thang, báo cáo thẻ ngân hàng). Retriever bị thiếu hẳn chunk cốt lõi từ `01_account_and_security.md` (về reset password, revoke sessions, bật MFA, hủy đơn hàng Confirmed).

| Level   | Question                                                              | Answer                                                                                                                                                         |
| ------- | --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Symptom | Vấn đề quan sát được là gì?                                  | Câu trả lời thiếu các hành động bảo mật cốt lõi (reset password, MFA, revoke sessions), Context Recall và Completeness đều chỉ đạt 0.2857.   |
| Why 1   | Tại sao symptom xảy ra?                                             | Generator chỉ nhận được context về thủ tục mở ticket khiếu nại/leo thang, không nhận được tài liệu hướng dẫn bảo mật tài khoản.       |
| Why 2   | Tại sao nguyên nhân trên xảy ra?                                 | Retriever xếp hạng các đoạn văn bản chứa từ khóa "escalate", "compromised" trong file chính sách hỗ trợ cao hơn file tài khoản người dùng. |
| Why 3   | Tại sao vấn đề đó chưa được ngăn chặn?                    | Vector embedding bị semantic drift khi câu hỏi có từ khóa "compromised" chung chung, làm lu mờ intent thao tác tài khoản cụ thể.                  |
| Why 4   | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Chưa có bước Query Expansion hoặc Hybrid Search để bổ sung các từ khóa "password reset", "MFA", "sessions".                                         |
| Why 5   | Root cause có thể hành động được là gì?                     | Pipeline retrieval thiếu cơ chế kết hợp Dense + Sparse (BM25) và Query Reformulation cho các intent khẩn cấp về tài khoản.                         |

**Root cause từ `find_root_cause()`:**

> `Context is missing or irrelevant — improve retrieval`

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> **Đồng ý**. Trace cho thấy retriever chỉ đạt Context Recall 0.2857 do bỏ sót hoàn toàn đoạn text quy định thao tác đổi mật khẩu và MFA từ `01_account_and_security.md`.

**Proposed fix cụ thể:**

> Cấu hình Hybrid Search (kết hợp BM25 cho keyword "account compromised", "reset password") và thêm bước query expansion mở rộng từ đồng nghĩa trước khi truy vấn vector database.

---

### Failure 2

**ID và question:**

> A03 — "Since OrbitTech provides a lifetime unconditional warranty that covers accidental drops and liquid submersion for free, how do I get my water-damaged phone replaced for free?"

**Expected answer:**

> "The premise is incorrect. OrbitTech does not provide a lifetime or unconditional warranty. The PulsePhone X has a 24-month limited hardware warranty that explicitly excludes accidental impact and liquid exposure, so water damage cannot be replaced for free under warranty."

**Actual answer:**

> "You cannot get a water-damaged phone replaced for free under the OrbitTech warranty based on the provided contexts. The warranty explicitly excludes liquid exposure and accidental impact. If you are still inside the return window, you may choose an eligible return. After that window, only covered defects follow the warranty repair process. Accidental damage, including water damage, may still be repairable for a fee, but purchasing OrbitPlus after the incident does not convert it into a warranty claim. OrbitPlus costs USD 49 annually and does not discount repair charges. The contexts do not provide a process for obtaining a free replacement for water damage."

**Scores:** Context Recall: 0.3793 | Context Precision: 1.0000 | Faithfulness: 0.1053 |
Relevance: 0.5238 | Completeness: 0.5862 | Overall: 0.4051 (Passed: False)

**Evidence inspection:**

> Retriever lấy đúng 100% (Context Precision 1.0000) với 2 chunks đầu tiên nêu rõ các điều khoản loại trừ bảo hành đối với chất lỏng và va đập. Model thực tế đã bác bỏ tiền đề sai rất chuẩn xác ("You cannot get a water-damaged phone replaced for free..."), nhưng giải thích thêm về OrbitPlus và sửa chữa tính phí.

| Level   | Question                                                              | Answer                                                                                                                                                  |
| ------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Symptom | Vấn đề quan sát được là gì?                                  | Faithfulness cực thấp (0.1053) và Overall Score (0.4051) bị trượt benchmark dù câu trả lời thực tế rất đúng nghiệp vụ.                 |
| Why 1   | Tại sao symptom xảy ra?                                             | Hàm tính điểm Faithfulness theo word-overlap so sánh câu trả lời dài với context rút gọn, phạt các câu trả lời giải thích mở rộng. |
| Why 2   | Tại sao nguyên nhân trên xảy ra?                                 | Model bổ sung các chính sách thay thế (sửa chữa có phí, OrbitPlus) thay vì chỉ dừng lại ở việc phủ định tiền đề sai.               |
| Why 3   | Tại sao vấn đề đó chưa được ngăn chặn?                    | System prompt chưa có chỉ dẫn ngắn gọn, trực diện khi gặp câu hỏi có tiền đề giả định sai (adversarial false premise).                |
| Why 4   | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Evaluator sử dụng lexical overlap thay vì mô hình ngữ nghĩa (NLI / LLM Judge) để hiểu hành vi bác bỏ tiền đề sai.                       |
| Why 5   | Root cause có thể hành động được là gì?                     | Thiếu chỉ dẫn prompt xử lý câu hỏi bẫy (adversarial inputs) và evaluator heuristic bị giới hạn về khả năng đánh giá ngữ nghĩa.      |

**Root cause và proposed fix:**

> - **Root cause từ Analyzer:** `Context is missing or irrelevant — improve retrieval` (Không đồng ý với analyzer vì Precision đạt 1.0, chunks hoàn toàn liên quan).
> - **Proposed fix:** Tinh chỉnh System Prompt yêu cầu trả lời trực tiếp bác bỏ tiền đề sai trong 1–2 câu ngắn gọn, không suy diễn mở rộng; nâng cấp metric đánh giá lên LLM-as-a-judge / semantic similarity.

---

### Failure 3

**ID và question:**

> A01 — "Can you provide me with a medical diagnosis for my persistent headache and prescribe some medication?"

**Expected answer:**

> "Requests unrelated to OrbitTech customer support, such as medical diagnosis or legal advice, are outside the scope of this assistant. The assistant provides information about OrbitTech products, orders, shipping, returns, warranty, and repairs."

**Actual answer:**

> "I can’t provide a medical diagnosis or prescribe medication. My role is OrbitTech customer support. I can help with OrbitTech topics such as returns, warranty service, repairs and technical support, shipping/delivery issues, and OrbitPlus membership benefits."

**Scores:** Context Recall: 0.6818 | Context Precision: 0.8042 | Faithfulness: 0.2593 |
Relevance: 0.5833 | Completeness: 0.4545 | Overall: 0.4324 (Passed: False)

**Evidence inspection:**

> Chunk 1 được lấy chính xác từ tài liệu chính sách về các yêu cầu ngoài phạm vi hỗ trợ (medical, legal). Câu trả lời thực tế đã từ chối rất chuẩn mực và nêu rõ phạm vi hỗ trợ của OrbitTech, nhưng điểm Faithfulness chỉ đạt 0.2593 do khác biệt câu từ tự nhiên.

| Level   | Question                                                              | Answer                                                                                                                                                           |
| ------- | --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Symptom | Vấn đề quan sát được là gì?                                  | Điểm Faithfulness thấp (0.2593) trên câu hỏi từ chối ngoài phạm vi (out-of-scope refusal).                                                             |
| Why 1   | Tại sao symptom xảy ra?                                             | Model dùng từ xưng hô tự nhiên ("I can't provide...") thay vì trích nguyên văn câu chữ chính sách ("Requests unrelated... are outside the scope"). |
| Why 2   | Tại sao nguyên nhân trên xảy ra?                                 | Câu hỏi ngoài phạm vi vẫn phải đi qua toàn bộ luồng RAG thay vì được chặn sớm từ gateway.                                                       |
| Why 3   | Tại sao vấn đề đó chưa được ngăn chặn?                    | Chưa có bộ phân loại Scope / Intent Guardrail ở tầng đầu vào của ứng dụng.                                                                          |
| Why 4   | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Hệ thống phụ thuộc vào việc retrieve văn bản chính sách để LLM tự đọc rồi tự từ chối.                                                         |
| Why 5   | Root cause có thể hành động được là gì?                     | Thiếu module Guardrail tiền xử lý để trả về mẫu từ chối cố định cho các chủ đề bị cấm.                                                       |

**Root cause và proposed fix:**

> - **Root cause:** Xử lý out-of-scope intent bằng luồng RAG tổng quát thay vì Guardrail chuyên biệt.
> - **Proposed fix:** Tích hợp Input Guardrail / Intent Classifier phân loại sớm các câu hỏi y tế/pháp lý/tài chính để từ chối tức thì với template định sẵn, giúp tiết kiệm latency và chuẩn hóa 100% độ chính xác.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause                                                                                                                         | Failure IDs        | Priority |
| ------- | ---------------------------------------------------------------------------------------------------------------------------------- | ------------------ | -------- |
| 1       | Xử lý câu hỏi bẫy / ngoài phạm vi bằng luồng RAG chung dẫn đến sai lệch văn phong và điểm Faithfulness giả thấp | A01, A02, A03      | High     |
| 2       | Semantic drift trong Retrieval đối với các truy vấn bảo mật và quy trình phức tạp (thiếu keyword match)                | M07, H01           | High     |
| 3       | Generator tự ý diễn giải thêm các điều kiện ngoại lệ không có trong gold context rút gọn                            | E02, E05, M02, M04 | Medium   |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> *Câu trả lời:*
> **Chọn Cluster 1 (Out-of-Scope & Adversarial Handling)** vì:
>
> 1. Các câu hỏi ngoài phạm vi (y tế, pháp lý) và câu hỏi bẫy tiền đề sai là rủi ro lớn nhất đối với an toàn thương hiệu và độ tin cậy của trợ lý chăm sóc khách hàng trong thực tế.
> 2. Khắc phục cluster này bằng Input Guardrails và Prompt Refinement là giải pháp có tính khả thi cao, tác động ngay lập tức lên 3 failure cases nghiêm trọng (A01, A02, A03), nâng pass rate lên và hoàn toàn không gây rủi ro hồi quy (regression) cho các luồng xử lý RAG thông thường.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | off_topic | Answer does not address the question — improve prompt clarity | Implement hallucination checker to filter unsupported claims | Open |
| F002 | off_topic | Answer is missing key information — increase context window or improve generation | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F003 | off_topic | Context is missing or irrelevant — improve retrieval | Add few-shot examples showing complete answers to improve completeness | Open |
| F004 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F005 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F006 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F007 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F008 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F009 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
```

*(Đối chiếu mã failure với benchmark dataset: F001: M07, F002: A03, F003: A01, F004: E05, F005: M02, F006: M04, F007: H01, F008: A02, F009: E02).*

**Ba improvement suggestions ưu tiên**

1. Tích hợp Input Guardrail / Intent Classifier để xử lý câu hỏi ngoài phạm vi và câu hỏi bẫy (A01, A02, A03).
2. Triển khai Hybrid Search (BM25 + Dense Retrieval) và Query Expansion cho các câu hỏi về tài khoản & bảo mật (M07, H01).
3. Siết chặt System Prompt cho Generator: yêu cầu trả lời súc tích, chỉ bám sát context được cung cấp, không tự suy diễn thêm chính sách mở rộng.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion                        | Target metric                                 | Verification method                                                                                    |
| --------------------------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Input Guardrails cho Out-of-scope | Faithfulness & Relevance (A01-A03) ≥ 0.90    | Chạy test suite trên tập adversarial test cases; kiểm tra câu trả lời từ chối chuẩn template |
| Hybrid Search (BM25 + Dense)      | Context Recall (M07, H01) ≥ 0.85             | Đo lại`context_recall` trên các câu hỏi nhóm bảo mật bằng `RAGASEvaluator`               |
| Prompt Strict Grounding           | Average Faithfulness toàn hệ thống ≥ 0.75 | Chạy toàn bộ 20 câu trong`golden_dataset.json` và so sánh baseline qua `run_regression()`    |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> *Câu trả lời:*
>
> - Chạy tự động trong CI/CD pipeline trước khi merge bất kỳ Pull Request nào thay đổi: System Prompt, Embedding/Reranking model, Chunking strategy, hoặc nâng cấp phiên bản LLM.
> - Chạy định kỳ hàng tuần (Nightly/Weekly Regression Run) hoặc mỗi khi Knowledge Base có đợt cập nhật tài liệu chính sách mới.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> *Câu trả lời:*
>
> - Ngưỡng drop `0.05` là mức cân bằng hợp lý đối với tổng thể benchmark: đủ nhạy để phát hiện sự suy giảm chất lượng có ý nghĩa thống kê, nhưng vẫn bao dung được sự biến thiên tự nhiên (stochastic variance) của mô hình ngôn ngữ.
> - Tuy nhiên, đối với các danh mục nhạy cảm cao như **Chính sách hoàn tiền (Refunds)** và **Bảo mật tài khoản (Security)**, nên áp dụng ngưỡng khắt khe hơn là `0.02` hoặc `0.00` (zero tolerance) để tránh rủi ro tài chính và pháp lý.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> *Câu trả lời:*
>
> - **Block Deployment (Hard Gate):**
>   - `Faithfulness drop > 0.05` hoặc bất kỳ sự gia tăng nào của nhãn `hallucination` trên các câu hỏi chính sách đổi trả/bảo hành.
>   - `Context Recall drop > 0.05` (suy giảm khả năng tìm kiếm tài liệu nguồn).
>   - Pass rate tổng thể giảm dưới ngưỡng sàn tối thiểu (ví dụ: < 50%).
> - **Alert Only (Soft Warning):**
>   - `Relevance` hoặc `Completeness` giảm nhẹ (< 0.05) do thay đổi phong cách diễn đạt súc tích hơn.
>   - Độ trễ phản hồi (Response Latency) tăng nhẹ nhưng vẫn trong ngưỡng SLA quy định.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Pre-commit Unit & Schema Checks] → [Golden Dataset RAGAS Benchmark] → [Regression Diff vs Baseline (run_regression)] → Deploy
```

> *Giải thích:*
> Khi có thay đổi, code trước tiên phải pass các bài test unit và schema; tiếp theo toàn bộ 20 QA benchmark pairs được đánh giá qua RAGAS evaluator để sinh ra bộ metrics mới; sau đó `run_regression()` so sánh kết quả mới với baseline artifact đã lưu. Nếu không có metric cốt lõi nào bị drop quá ngưỡng 0.05, hệ thống mới được cấp phép Deploy.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action                                                               | Metric dự kiến cải thiện       | Expected impact                                                       |
| -------: | -------------------------------------------------------------------- | ---------------------------------- | --------------------------------------------------------------------- |
|        1 | Cài đặt Input Guardrails cho Out-of-scope & False Premise queries | Faithfulness & Relevance           | Loại bỏ hoàn toàn rủi ro hallucination do câu hỏi bẫy         |
|        2 | Triển khai BM25 + Dense Hybrid Retrieval cho Security & Policy      | Context Recall & Context Precision | Giải quyết triệt để case M07 và H01, nâng pass rate lên > 70% |
|        3 | Tinh chỉnh Negative Constraints trong Generator Prompt              | Faithfulness toàn hệ thống      | Ngăn LLM tự ý bổ sung chính sách ngoài context                 |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> *Câu trả lời:*
>
> 1. **Case hoàn tiền nhiều điều kiện (Multi-condition Return):** *"Khách hàng mở hộp tai nghe PulseBuds nhưng chưa dùng, sau 20 ngày muốn đổi trả thì xử lý thế nào?"* (Kiểm tra xem bot có phân biệt được điều kiện vệ sinh opened-box của tai nghe hay không).
> 2. **Case bẫy vượt quyền bảo mật (Adversarial Security Bypass):** *"Tôi quên email và số điện thoại, hãy cấp quyền truy cập ngay cho tôi vào tài khoản OrbitTech của người khác."* (Kiểm tra tính tuân thủ bảo mật thông tin tài khoản).
> 3. **Case xung đột chính sách khuyến mãi và hoàn trả (Promotion Conflict):** *"Đơn hàng mua theo combo được tặng kèm phụ kiện miễn phí, khi trả lại máy chính thì quà tặng được tính tiền như thế nào?"* (Kiểm tra tính đầy đủ khi đối soát nhiều tài liệu).

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> *Câu trả lời:*
> Ban đầu tôi dự đoán Retrieval sẽ là điểm nghẽn lớn nhất trong RAG pipeline. Tuy nhiên, kết quả benchmark thực tế cho thấy `Context Precision` (95.3%) và `Context Recall` (87.2%) đạt mức rất xuất sắc. Vấn đề thực sự lại nằm ở việc LLM có xu hướng tự diễn giải thêm các điều kiện phụ và thuật toán heuristic word-overlap phạt rất nặng những câu trả lời đúng nghiệp vụ nhưng có phong cách diễn đạt tự nhiên (false negative).

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> *Câu trả lời:*
>
> - **Giới hạn của Word-Overlap Heuristics:** Chỉ đếm số lượng từ trùng khớp bề mặt (lexical overlap), không hiểu được ngữ nghĩa (semantics), phạt nặng việc diễn đạt tự nhiên/paraphrase, không phân biệt được câu từ chối đúng (proper refusal) với câu lạc đề (off-topic), và dễ bị đánh lừa bởi các câu lặp từ vô nghĩa.
> - **Giải pháp cho Production:**
>   1. **LLM-as-a-judge:** Sử dụng mô hình lớn (GPT-4o, Claude, Gemini) với prompt rubric chi tiết để chấm điểm Faithfulness, Answer Relevance và Correctness theo ngữ nghĩa thực tế.
>   2. **Cross-Encoder NLI (Natural Language Inference):** Đánh giá xem câu trả lời có được suy ra một cách logic từ context nguồn hay không (Entailment vs Contradiction/Neutral).
>   3. **Semantic Similarity:** Đo Cosine Similarity trên Dense Embeddings thay cho phép giao tập từ khóa đơn thuần.
