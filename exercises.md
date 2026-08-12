# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 09:15–12:00

**Domain:** Northstar University Student Services

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 09:15–09:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (09:30–09:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Khi câu hỏi mang tính ngoại suy rộng hơn context hoặc out-of-scope mà agent trả lời bằng tri thức chung không gây hại. | Agent bịa đặt (hallucinate) sai thông tin chính sách, học phí, ngày hạn nộp đơn trái với context. | Thắt chặt prompt grounding, yêu cầu từ chối khi context không có thông tin, tăng penalty với hallucination. |
| Answer Relevance | Câu trả lời quá ngắn gọn (đúng trọng tâm nhưng ít từ trùng khớp với question) hoặc dùng từ đồng nghĩa khác. | Answer đi lạc đề hoàn toàn, trả lời sang quy định/chính sách khác không liên quan đến thắc mắc. | Tối ưu hóa System Prompt, cải thiện Intent Classification, yêu cầu trả lời trực tiếp câu hỏi trước. |
| Context Recall | `expected_answer` chứa thông tin bổ sung quá chi tiết mà context không nhất thiết phải phủ hết. | Retriever bỏ sót tài liệu chứa thông tin cốt lõi (ví dụ bỏ sót điều kiện miễn giảm học phí). | Cải tiến Retriever: tăng Top-K, kết hợp Hybrid Search (BM25 + Dense Retrieval) hoặc Reranking. |
| Context Precision | Các chunk liên quan đứng ở vị trí k=2 hoặc k=3 (thay vì k=1) nhưng LLM vẫn đọc và trích xuất đúng. | Các chunk chứa thông tin đúng bị đẩy xuống cuối (k > 5) hoặc danh sách chứa quá nhiều chunk nhiễu. | Thêm bước Reranking bằng Cross-Encoder để đẩy chunk quan trọng lên vị trí đầu; tinh chỉnh chunk size. |
| Completeness | Trả lời đúng các ý chính cốt lõi nhưng bỏ qua các ví dụ minh họa phụ hoặc các chi tiết không bắt buộc. | Bỏ sót điều kiện tiên quyết hoặc các bước bắt buộc trong quy trình (ví dụ bỏ sót hạn nộp đơn). | Cải thiện Generation Prompt yêu cầu liệt kê đầy đủ tất cả điều kiện/bước dưới dạng danh sách bullet points. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Condition 1 (Thứ tự gốc):** Gửi cho LLM Judge cặp câu trả lời theo thứ tự `[Prompt + Context + Answer A (Vị trí 1) + Answer B (Vị trí 2)]`.
> - **Condition 2 (Đảo vị trí):** Gửi cùng prompt/context nhưng đảo ngược vị trí `[Prompt + Context + Answer B (Vị trí 1) + Answer A (Vị trí 2)]`.
> - **Đánh giá & Kết luận:** Nếu Judge chọn Answer A ở Condition 1 nhưng chuyển sang chọn Answer B ở Condition 2 (luôn ưu tiên Vị trí 1), ta kết luận Judge bị Position Bias. Giải pháp khắc phục là chạy cả 2 conditions và lấy điểm trung bình (Pairwise Swap / Position Swapping).

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> 1. Định nghĩa rõ tiêu chí chấm điểm dựa trên **mật độ thông tin chính xác (information density)** và tính đầy đủ của bằng chứng, không chấm dựa trên độ dài từ/câu.
> 2. Đưa vào Rubric tiêu chí phạt (Penalty Rule) đối với câu trả lời dài dòng, chứa từ ngữ hoa mỹ hoặc lặp từ nhưng không bổ sung giá trị thông tin mới.
> 3. Yêu cầu Judge trích xuất các ý chính (Key facts extraction) trước khi đưa ra điểm số tổng kết.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> - LLM Judge vẫn có các điểm mù (bias nội tại, trôi chảy nhưng sai sự thật, hiểu sai nuance miền chuyên môn sâu).
> - Human Labels (được dán nhãn bởi chuyên gia miền Student Services) đóng vai trò là "Ground Truth" để kiểm chứng.
> - Calibration giúp tính toán độ tương quan (như Cohen's Kappa hoặc Spearman Rank Correlation) giữa LLM Judge và Human Judge. Qua đó tinh chỉnh Prompt / Rubric cho tới khi LLM Judge đạt độ đồng thuận cao với con người (> 80–85%), giúp pipeline đánh giá tự động vừa nhanh vừa tin cậy.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | **0.85** | Tránh bịa đặt thông tin chính sách/quy định. Bịa đặt thông tin gây hậu quả nghiêm trọng về niềm tin sinh viên và pháp lý. |
| Answer Relevance | **0.80** | Đảm bảo câu trả lời giải quyết đúng thắc mắc của sinh viên, không trả lời lan man hoặc nhầm chủ đề. |
| Completeness | **0.75** | Cần đảm bảo đủ các bước quan trọng. Chấp nhận 0.75 nếu đã nắm trọn các ý cốt lõi mà chỉ thiếu ví dụ hoặc chi tiết phụ. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation:** Dùng trong quá trình phát triển (Dev/Staging) trước khi release phiên bản prompt/model/retriever mới. Chạy tự động trên Golden Dataset 20 câu để làm Quality Gate cho CI/CD pipeline (nếu score < threshold ➔ block deploy).
> - **Online Evaluation:** Dùng trên môi trường Production với real user traffic. Theo dõi liên tục các chỉ số như user feedback (thumbs up/down), latency, cost, hoặc dùng Evaluator/Langfuse/TruLens chấm điểm ngẫu nhiên trên sample traffic thực tế.
> - **Human Review:** Dùng định kỳ (hàng tuần/tháng) hoặc đối với các edge cases, các câu hỏi bị user phản hồi tiêu cực (thumbs down), các câu hỏi có điểm offline nằm ở vùng nghi vấn (0.5 – 0.7). Human Review dùng để cập nhật Golden Dataset và calibrate LLM Judge.

---

## Part 2 — Core Coding (09:45–10:40)

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

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

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

Thiết kế rubric domain-specific cho Student Services. Mỗi mức phải đủ cụ thể để
hai người chấm độc lập có thể hiểu giống nhau.

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
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +10)

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
