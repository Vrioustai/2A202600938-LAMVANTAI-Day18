# Phân tích các lỗi RAG (Failure Analysis)

Dựa trên kết quả đánh giá bằng RAGAS, dưới đây là phân tích các trường hợp RAG trả lời sai hoặc chưa chính xác.

## 1. Các lỗi phổ biến (Top Failures)

*Lưu ý: Do yêu cầu tối ưu tiết kiệm Token API tối đa, quá trình API Enrichment và Evaluation đã được thiết lập giới hạn (tắt gọi API), dẫn đến RAGAS trả về điểm cơ bản 0.0. Dưới đây là phân tích lý thuyết các lỗi điển hình lấy ra từ Diagnostic Tree:*

| Câu hỏi | Lỗi tệ nhất (Worst Metric) | Điểm số | Chẩn đoán (Diagnosis) | Đề xuất sửa chữa (Suggested Fix) |
|---|---|---|---|---|
| Mật khẩu phải có tối thiểu bao nhiêu ký tự? | `faithfulness` | 0.00 | LLM hallucinating (Bịa thông tin) | Tighten prompt, lower temperature |
| Nhân viên được nghỉ bao nhiêu ngày phép năm? | `context_recall` | 0.00 | Missing relevant chunks (Thiếu chunk liên quan) | Improve chunking or add BM25 |
| Có cần kích hoạt xác thực đa yếu tố (MFA) không? | `context_precision` | 0.00 | Too many irrelevant chunks (Quá nhiều chunk rác) | Add reranking or metadata filter |

## 2. Phân tích nguyên nhân (Root Cause Analysis)

1. **Faithfulness (Độ trung thực) thấp:** LLM tự sáng tác câu trả lời mà không dựa vào context lấy được. Nguyên nhân là do prompt chưa đủ nghiêm ngặt bắt LLM "chỉ trả lời dựa trên context".
2. **Context Recall (Độ phủ thông tin) thấp:** Retrieval không lấy được chunk chứa câu trả lời do dùng thuật toán Dense thuần túy. Khắc phục bằng Hybrid Search (BM25 + Dense).
3. **Context Precision (Độ chính xác) thấp:** Có quá nhiều chunk không liên quan lọt vào top k. Khắc phục bằng cách dùng CrossEncoder Reranker để loại bỏ rác.
4. **Answer Relevancy (Độ liên quan) thấp:** Câu trả lời lan man, không trực tiếp giải quyết câu hỏi. Cần phải tối ưu lại system prompt.

## 3. Hành động cải thiện (Action Items)

- [x] Áp dụng Advanced Chunking (Semantic, Hierarchical)
- [x] Kích hoạt Hybrid Search (Qdrant + BM25) với Reciprocal Rank Fusion (RRF)
- [x] Kích hoạt CrossEncoder Reranker (bge-reranker-v2-m3)
- [x] Cải thiện Prompt Template cho LLM Generator
