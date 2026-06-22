# Kế hoạch Thực thi Lab 18: Production RAG Pipeline

Tài liệu này mô tả chi tiết spec và luồng xử lý (flow) cho từng phần để hoàn thiện dự án Lab 18.

## User Review Required

> [!IMPORTANT]
> Cần xác nhận từ bạn trước khi bắt đầu viết code:
> 1. Bạn có đồng ý với kế hoạch chi tiết dưới đây không?
> 2. **API Key**: Module 4 (RAGAS Eval) và Module 5 (Enrichment) yêu cầu sử dụng `OPENAI_API_KEY`. Hãy chắc chắn bạn đã cấu hình key này trong file `.env` trước khi chúng ta chạy các phần đó.

## Mục tiêu Dự án
Chuyển đổi hệ thống từ **Naive RAG** (chỉ cắt bằng độ dài cố định, dùng BM25/Dense cơ bản) sang **Production RAG** với các cải tiến về Chunking, Search, Reranking, Evaluation, và Enrichment.

---

## Luồng công việc tổng thể (Flow)

1. **Chuẩn bị (Setup)**: Đảm bảo Docker (Qdrant) đang chạy, `requirements.txt` đã cài đặt, và chạy `naive_baseline.py` lấy điểm gốc.
2. **Implement M1 (Chunking)**: Thay đổi cách chia nhỏ văn bản.
3. **Implement M2 (Hybrid Search)**: Tìm kiếm đồng thời theo BM25 và Vector, kết hợp bằng RRF.
4. **Implement M3 (Reranking)**: Dùng mô hình chuyên biệt để đánh giá lại kết quả tìm kiếm.
5. **Implement M4 (RAGAS Eval)**: Tự động chấm điểm.
6. **Implement M5 (Enrichment)**: Dùng LLM (OpenAI) để thêm thông tin phụ trợ cho tài liệu trước khi index.
7. **Kiểm thử (Verify)**: Chạy `pytest` và file `main.py` để ra báo cáo, so sánh điểm. Cuối cùng, ghi nhận file `failure_analysis.md`.

---

## Chi tiết kỹ thuật từng Module (Spec)

### Module 1: Advanced Chunking (`src/m1_chunking.py`)
Mục tiêu: Không làm mất ngữ cảnh (context) khi cắt văn bản.
*   **`chunk_semantic`**:
    *   Chia text thành danh sách các câu.
    *   Dùng `SentenceTransformer('all-MiniLM-L6-v2')` để nhúng (embed) các câu.
    *   Tính Cosine Similarity giữa các câu liền kề. Nếu độ tương đồng `< 0.85` (ngưỡng), cắt thành chunk mới.
*   **`chunk_hierarchical`**:
    *   Tách nội dung thành các paragraph lớn (parent - kích thước <= 2048).
    *   Từ mỗi parent, tiếp tục chia thành các child chunks (kích thước <= 256).
    *   Gắn ID `parent_id` cho child để lúc tìm kiếm thấy child thì trả về nội dung của parent (giữ context rộng).
*   **`chunk_structure_aware`**:
    *   Sử dụng regex phân tích các Header Markdown (`#`, `##`, `###`).
    *   Tạo các đoạn nội dung nằm dưới từng Header thành một chunk độc lập, giữ nguyên cấu trúc List/Table.

### Module 2: Hybrid Search (`src/m2_search.py`)
Mục tiêu: Tối ưu khả năng tìm kiếm từ khóa tiếng Việt và ngữ nghĩa.
*   **`segment_vietnamese`**:
    *   Dùng thư viện `underthesea` để tokenize tiếng Việt (nối từ ghép). 
    *   Thay khoảng trắng `_` bằng ` ` (space) để BM25 hoạt động khớp.
*   **`BM25Search`**:
    *   `index()`: Khởi tạo `BM25Okapi` trên tập văn bản đã được segment tiếng Việt.
    *   `search()`: Trả về Top-K kết quả với method là "bm25". Lọc bỏ kết quả có score = 0.
*   **`DenseSearch`**:
    *   `index()`: Dùng `SentenceTransformer('BAAI/bge-m3')` để tạo vector, đẩy lên **Qdrant** qua `client.upsert`.
    *   `search()`: Gọi `client.query_points` để tìm kiếm vector gần nhất. Trả kết quả method "dense".
*   **`reciprocal_rank_fusion (RRF)`**:
    *   Hàm nhận vào danh sách kết quả từ BM25 và Dense.
    *   Tính điểm mới theo công thức: `Score = sum(1 / (k + rank + 1))` với k=60. 
    *   Sắp xếp giảm dần và lấy Top-K.

### Module 3: Reranking (`src/m3_rerank.py`)
Mục tiêu: Sắp xếp lại top 20 kết quả của Hybrid Search để đưa kết quả chính xác nhất lên đầu (top 3).
*   **`CrossEncoderReranker._load_model`**: Load mô hình `sentence_transformers.CrossEncoder('BAAI/bge-reranker-v2-m3')`.
*   **`CrossEncoderReranker.rerank`**:
    *   Tạo danh sách các cặp `(query, document_text)`.
    *   Đưa vào mô hình `predict` để lấy điểm.
    *   Sắp xếp lại kết quả giảm dần theo điểm rerank và trả về top K tài liệu.

### Module 4: RAGAS Evaluation (`src/m4_eval.py`)
Mục tiêu: Đánh giá bằng số liệu và đưa ra lời khuyên sửa lỗi.
*   **`evaluate_ragas`**:
    *   Sử dụng Ragas để chấm điểm tập `test_set.json`.
    *   Tính 4 metrics: `faithfulness`, `answer_relevancy`, `context_precision`, `context_recall`.
*   **`failure_analysis`**:
    *   Duyệt qua danh sách câu hỏi, lấy ra bottom N câu hỏi có điểm trung bình thấp nhất.
    *   Map chỉ số thấp nhất của câu đó vào một bảng *Diagnostic Tree* để chỉ định "Vấn đề (Diagnosis)" và "Cách sửa (Suggested fix)".

### Module 5: Enrichment (`src/m5_enrichment.py`)
Mục tiêu: Nhúng siêu dữ liệu (metadata) trực tiếp vào chunk trước khi sinh vector.
*   **`_enrich_single_call` (Combined Mode)**:
    *   Gửi 1 request duy nhất lên GPT-4o-mini thông qua thư viện `openai`.
    *   System Prompt ép mô hình trả về JSON gồm:
        1. Tóm tắt chunk.
        2. Sinh 3 câu hỏi giả định (HyQA).
        3. Context (Mô tả nội dung này nằm ở vị trí nào của tài liệu).
        4. Metadata (Topic, Category, Entities).
    *   Trả về text đã được làm giàu, ghép dòng "Context" vào phía trên nội dung gốc (`context_line + \n\n + original_text`).

## Verification Plan (Kế hoạch kiểm thử)

### Automated Tests
Sau khi code xong mỗi module, chạy unit test định sẵn để check:
- `pytest tests/test_m1.py -v`
- `pytest tests/test_m2.py -v`
- `pytest tests/test_m3.py -v`
- `pytest tests/test_m4.py -v`
- `pytest tests/test_m5.py -v`

### End-to-End
Chạy:
- `python src/pipeline.py`: Chạy chu trình toàn bộ xem có lỗi lầm nào không.
- `python main.py`: In bảng điểm báo cáo Baseline vs Production.
- `python check_lab.py`: Quét đảm bảo không còn dòng code `# TODO` nào.
