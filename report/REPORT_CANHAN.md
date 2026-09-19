# Báo Cáo Cá Nhân — Lab 7: Embedding & Vector Store

**Họ tên:** Ngô Gia Quốc
**Nhóm:** VSF
**Ngày:** 19/9/2026

> **Nộp 1 bản / sinh viên.** Phần nhóm (lựa chọn tài liệu, thiết kế chiến lược, bộ câu hỏi đánh giá, demo) nộp chung 1 bản trong `REPORT_NHOM.md`. Chi tiết thang điểm: `docs/SCORING.md`.

**Tổng điểm phần cá nhân: 60** = Khởi động (5) + Hướng tiếp cận (10) + Hoàn thiện code (30) + Dự đoán độ tương tự (5) + Kết quả truy xuất của tôi (10).

---

## 1. Khởi động (Warm-up) — Cá nhân (5 điểm)

### Độ tương tự Cosine (Cosine Similarity) (Bài tập 1.1)

**Độ tương tự cosine cao (High cosine similarity) nghĩa là gì?**
> Độ tương tự cosine cao nghĩa là hai vector biểu diễn văn bản có hướng gần nhau trong không gian embedding. Điều này thường cho thấy hai đoạn văn có nội dung hoặc ý nghĩa ngữ nghĩa gần nhau, dù chúng không nhất thiết sử dụng cùng từ vựng.

**Ví dụ có độ tương tự CAO:**
- Câu A: “Sinh viên cần đăng nhập SIS để đăng ký học phần.”
- Câu B: “Người học phải truy cập hệ thống quản lý sinh viên để ghi danh môn học.”
- Tại sao tương đồng: Hai câu dùng từ khác nhau nhưng đều mô tả cùng hành động đăng nhập hệ thống để đăng ký môn học.

**Ví dụ có độ tương tự THẤP:**
- Câu A: “Sinh viên cần đăng nhập SIS để đăng ký học phần.”
- Câu B: “Thời tiết hôm nay có nhiều mây và khả năng mưa lớn.”
- Tại sao khác: Hai câu thuộc hai chủ đề và mục đích hoàn toàn khác nhau: đăng ký học tập và dự báo thời tiết.

**Tại sao độ tương tự cosine (cosine similarity) được ưu tiên hơn khoảng cách Euclid (Euclidean distance) cho text embeddings?**
> Cosine similarity so sánh hướng của hai vector nên tập trung vào quan hệ ngữ nghĩa, ít bị ảnh hưởng bởi độ lớn vector hoặc độ dài văn bản. Khoảng cách Euclid phụ thuộc cả hướng lẫn độ lớn, vì vậy hai văn bản cùng nghĩa vẫn có thể bị đánh giá xa nhau nếu chuẩn vector khác nhau.

### Bài toán tính toán Chunking (Bài tập 1.2)

**Tài liệu 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> *Trình bày phép tính:* Bước dịch giữa hai chunk là `500 - 50 = 450` ký tự. Theo công thức của bài: `ceil((10.000 - 50) / (500 - 50)) = ceil(9.950 / 450) = ceil(22,11) = 23`.
> *Đáp án:* 23 chunks.

**Nếu độ chồng chéo (overlap) tăng lên 100, số lượng chunk thay đổi thế nào? Tại sao muốn độ chồng chéo nhiều hơn?**
> Khi `overlap=100`, bước dịch còn `500 - 100 = 400`, nên số chunk là `ceil((10.000 - 100) / 400) = ceil(24,75) = 25`. Overlap lớn hơn giúp giữ ngữ cảnh và dữ kiện nằm gần ranh giới chunk, nhưng làm tăng số chunk, dung lượng lưu trữ và chi phí embedding/search.

---

## 2. Hướng tiếp cận của tôi (My Approach) — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi lập trình (implement) các phần chính trong gói `src`.

### Các hàm chia nhỏ (Chunking Functions)

**`SentenceChunker.chunk`** — hướng tiếp cận:
> Tôi dùng regex `(?<=[.!?])(?:[ \t]+|\n+)` để tách sau dấu `.`, `!`, `?` khi theo sau là khoảng trắng hoặc xuống dòng, nhờ đó giữ dấu câu trong câu đứng trước. Các câu được `strip`, bỏ phần rỗng rồi nhóm theo `max_sentences_per_chunk`; văn bản rỗng hoặc chỉ có khoảng trắng trả về danh sách rỗng.

**`RecursiveChunker.chunk` / `_split`** — hướng tiếp cận:
> Thuật toán thử các separator theo thứ tự `\n\n`, `\n`, `. `, khoảng trắng và cuối cùng là chuỗi rỗng; các phần quá dài được đệ quy với separator tiếp theo rồi ghép lại đến gần `chunk_size`. Base case là text rỗng trả `[]`, text không vượt giới hạn trả một chunk; nếu hết separator thì cắt cứng theo `chunk_size` để bảo đảm kết thúc đệ quy.

### Lớp EmbeddingStore

**`add_documents` + `search`** — hướng tiếp cận:
> `add_documents` tạo một record cho mỗi `Document`, sao chép metadata, sinh embedding từ content và lưu record vào danh sách in-memory cùng chỉ số chèn. `search` embed query, tính dot product với mọi record, sắp xếp score giảm dần và trả tối đa `top_k`; với embedding đã chuẩn hóa, dot product tương đương cosine similarity.

**`search_with_filter` + `delete_document`** — hướng tiếp cận:
> `search_with_filter` lọc metadata trên toàn bộ tập record trước rồi mới similarity search, tránh để tài liệu sai metadata chiếm các vị trí top-k. `delete_document` loại mọi record có `metadata["doc_id"]` bằng ID tài liệu cần xóa và trả `True` khi số record thực sự giảm.

### Tác tử KnowledgeBaseAgent

**`answer`** — hướng tiếp cận:
> `answer` lấy top-k chunk, đánh số từng nguồn và ghép `source_url`, `source`, `doc_id` hoặc chunk ID cùng nội dung thành context. Prompt yêu cầu LLM chỉ trả lời từ context, trích dẫn `[1]`, `[2]`, ... và nói rõ không tìm thấy nếu thiếu dữ kiện; nếu retrieval không có kết quả, hàm trả thông báo không tìm thấy mà không gọi LLM.

---

## 3. Hoàn thiện code (Core Implementation) — Cá nhân (30 điểm)

Vượt qua bộ kiểm thử là điều kiện tính điểm phần này.

### Kết Quả Kiểm Thử (Test Results)

```text
============================= test session starts =============================
collected 42 items

tests/test_solution.py ..........................................          [100%]

============================== 42 passed ==============================
```

**Số lượng bài test vượt qua (pass):** 42 / 42

---

## 4. Dự đoán độ tương tự (Similarity Predictions) — Cá nhân (5 điểm)

Các dự đoán dưới đây được ghi theo ngưỡng quy ước `0,60`: từ `0,60` trở lên là cao, dưới `0,60` là thấp. Điểm thực tế lấy từ OpenAI `text-embedding-3-small` trong lượt benchmark đã lưu; Câu B là phần tóm tắt nội dung đầy đủ của top-1 chunk tương ứng.

| Cặp | Câu A | Câu B | Dự đoán | Điểm thực tế | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | “Tối đa bao nhiêu tín chỉ được phép chuyển vào chương trình đại học tại VinUni?” | “Tối đa 60 tín chỉ được phép chuyển vào chương trình đại học tại VinUni.” | Cao | 0.7397 | Đúng |
| 2 | “Học phần cần đáp ứng mức tương đương nội dung và điểm tối thiểu nào để được xem xét chuyển đổi tín chỉ?” | “Có nội dung tương đương ít nhất 70% và điểm số tối thiểu là C hoặc tương đương.” | Cao | 0.6350 | Đúng |
| 3 | “Trên SIS cần thao tác thế nào để hoàn tất đăng ký môn và trạng thái nào xác nhận đăng ký thành công?” | “Nhấn Add rồi Register; trạng thái Registered xác nhận đăng ký thành công.” | Cao | 0.6863 | Đúng |
| 4 | “Yêu cầu bảng điểm hoặc thư xác nhận thường mất bao lâu và phí mỗi bản là bao nhiêu?” | “Yêu cầu cấp Bảng điểm và Chứng nhận - Registrar.” | Thấp | 0.5350 | Đúng |
| 5 | “Cần thực hiện những bước nào để đăng ký học phần?” | “Các bước quan trọng để chuẩn bị cho quá trình đăng ký học phần.” | Cao | 0.7222 | Đúng về similarity chủ đề, nhưng chưa đủ để trả lời |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn ý nghĩa?**
> Cặp 5 đáng chú ý nhất: score 0.7222 rất cao vì hai câu cùng chủ đề đăng ký học phần, nhưng chunk không chứa các thao tác SIS cần để trả lời gold answer. Điều này cho thấy embedding biểu diễn độ gần về ngữ nghĩa/chủ đề, không trực tiếp đo việc một chunk có đủ dữ kiện trả lời câu hỏi hay không; vì vậy benchmark vẫn phải kiểm nội dung answer chunk thay vì chỉ dựa vào score.

---

## 5. Kết quả truy xuất của tôi (Competition Results) — Cá nhân (10 điểm)

Chạy **5 câu hỏi đánh giá của nhóm** trên mã nguồn cá nhân của bạn trong gói `src`. **5 câu hỏi này phải trùng với các thành viên cùng nhóm** (xem `REPORT_NHOM.md`).

**Chiến lược cá nhân:** Custom `HeadingChunker`, `chunk_size=500`; section dài được chia tiếp bằng `RecursiveChunker` và tiêu đề được gắn lại vào từng chunk con.

**Điều kiện chạy:** 6 tài liệu, 90 chunk, `top_k=3`, OpenAI `text-embedding-3-small`. Kết quả được tạo bằng cùng corpus, năm query, gold answer và `HeadingChunker`; chỉ embedding backend được đổi từ mock sang embedding thật.

| # | Câu hỏi (Query) | Top-1 Chunk truy xuất được (tóm tắt) | Điểm Score | Có liên quan không? (Relevant) | Câu trả lời của Agent (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | Tối đa bao nhiêu tín chỉ được phép chuyển vào chương trình đại học tại VinUni? | Chunk `chuyen-doi-tin-chi#6`, chứa trực tiếp quy định tối đa 60 tín chỉ | 0.7397 | Có; answer chunk ở top-1 | Chưa chạy LLM; context top-1 đủ để trả lời “Tối đa 60 tín chỉ” |
| 2 | Học phần cần đáp ứng mức tương đương nội dung và điểm tối thiểu nào để được xem xét chuyển đổi tín chỉ? | Chunk `chuyen-doi-tin-chi#4`, chứa điều kiện tương đương ít nhất 70% và điểm tối thiểu C | 0.6350 | Có; answer chunk ở top-1 | Chưa chạy LLM; context top-1 đủ để trả lời đúng hai điều kiện |
| 3 | Trên SIS cần thao tác thế nào để hoàn tất đăng ký môn và trạng thái nào xác nhận đăng ký thành công? | Chunk `thoi-khoa-bieu-dang-ky-hoc-phan#5`, chứa Add, Register và Registered | 0.6863 | Có; answer chunk ở top-1 | Chưa chạy LLM; context top-1 đủ để trả lời đúng quy trình và trạng thái |
| 4 | Yêu cầu bảng điểm hoặc thư xác nhận thường mất bao lâu và phí mỗi bản là bao nhiêu? | Chunk `yeu-cau-cap-bang-diem-va-chung-nhan#0`, đúng tài liệu nhưng không chứa số liệu; chunk `#9` ở top-2 chứa phí 50.000 VNĐ/bản | 0.5350 | Liên quan một phần; top-3 có phí nhưng thiếu thời gian 2–3 ngày và tối đa 5 ngày | Chưa chạy LLM; context chỉ đủ trả lời phần phí, không đủ gold answer hoàn chỉnh |
| 5 | Cần thực hiện những bước nào để đăng ký học phần? (`audience=student`) | Chunk `thoi-khoa-bieu-dang-ky-hoc-phan#2`, nói về các bước chuẩn bị đăng ký | 0.7222 | Không đủ; top-3 đúng tài liệu nhưng không có chunk chứa trọn quy trình SIS theo gold answer | Chưa chạy LLM; context thiếu Course Registration, Add/Register và Your Class Schedule nên không thể trả lời đầy đủ |

**Bao nhiêu câu hỏi trả về chunk có liên quan trong top-3?** 4 / 5 nếu tính Q4 là liên quan một phần; trong đó 3 / 5 câu có chunk chứa đầy đủ các marker của gold answer.

**Điểm truy xuất theo chất lượng context:** 7/10 tạm tính: Q1–Q3 mỗi câu 2 điểm, Q4 có chunk liên quan nhưng thiếu chi tiết nên 1 điểm, Q5 không có answer chunk nên 0 điểm. Đây chưa phải điểm rubric chính thức vì benchmark chưa gọi LLM để xác minh câu trả lời của agent.

**Phân tích failure case:** Q4 lấy đúng tài liệu ở cả ba vị trí và lấy được chunk chứa phí ở top-2, nhưng không lấy được section “Cách Yêu Cầu Tài Liệu” chứa thời gian xử lý. Q5 cũng lấy đúng tài liệu ở cả ba vị trí nhưng xếp các section giới thiệu/chuẩn bị cao hơn hai chunk chứa quy trình SIS. Điều này cho thấy embedding thật đã nhận diện đúng chủ đề và tài liệu, nhưng `HeadingChunker` không có overlap nên mỗi section chứa dữ kiện chỉ có một cơ hội lọt top-3; có thể cải thiện bằng overlap hoặc tăng `top_k` sau khi giữ nguyên cấu hình benchmark để so sánh công bằng.

**Kết quả A/B metadata filter:** Ở Q5, top-3 có filter `audience=student` và không filter giống hệt nhau, cùng score và thứ hạng. Vì cả ba kết quả cao nhất vốn đã có `audience=student`, filter không thay đổi retrieval trong lần chạy OpenAI này; theo tiêu chí của lab, câu hỏi hiện tại chưa chứng minh được lợi ích thực tế của metadata filter.

**Điều hay nhất tôi học được từ thành viên khác / nhóm khác (qua demo):**
> Tại thời điểm hoàn thiện báo cáo, tôi chưa nhận được kết quả hoặc demo của thành viên/nhóm khác nên không có cơ sở để gán một bài học cho họ. Bài học tạm thời từ quy trình so sánh là phải giữ nguyên corpus, năm query, embedding backend và `top_k`, chỉ thay chunker; nếu thay nhiều biến cùng lúc thì không thể xác định chiến lược nào tạo ra khác biệt.

---

## Tự Đánh Giá (Phần Cá Nhân)

| Tiêu chí | Điểm tự đánh giá |
|----------|-------------------|
| Khởi động (Warm-up) | 5 / 5 |
| Hướng tiếp cận của tôi (My Approach) | 10 / 10 |
| Hoàn thiện code (Core Implementation — tests) | 30 / 30 |
| Dự đoán độ tương tự (Similarity Predictions) | 5 / 5 |
| Kết quả truy xuất của tôi (Competition Results) | 7 / 10 (tạm tính theo context, chưa chạy agent) |
| **Tổng phần cá nhân** | **57 / 60 (tạm tính)** |
