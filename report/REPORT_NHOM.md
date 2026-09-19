# Báo Cáo Nhóm — Lab 7: Embedding & Vector Store

**Nhóm:** VSF
**Thành viên:** [Họ tên từng thành viên]
**Ngày:** 19/9/2026

> **Nộp 1 bản / nhóm.** Phần cá nhân (hướng tiếp cận, kết quả riêng, dự đoán…) mỗi thành viên nộp riêng trong `REPORT_CANHAN.md`. Chi tiết thang điểm: `docs/SCORING.md`.

**Tổng điểm phần nhóm: 40** = Lựa chọn tài liệu (10) + Thiết kế chiến lược (15) + Chất lượng truy xuất (10) + Thuyết trình (5).

---

## 1. Lựa chọn tài liệu (Document Set Quality) — Nhóm (10 điểm)

### Chủ đề (Domain) & Lý Do Chọn

**Chủ đề:** [ví dụ: Customer support FAQ, Luật Việt Nam, công thức nấu ăn, ...]

**Tại sao nhóm chọn chủ đề này?**
> *Viết 2-3 câu:*

### Danh sách tài liệu (Data Inventory)

| # | Tên tài liệu | Nguồn (Source URL) | Ngày lấy / Phiên bản | Số ký tự | Metadata đã gán |
|---|--------------|------------|--------------------|----------|-----------------|
| 1 | | | | | |
| 2 | | | | | |
| 3 | | | | | |
| 4 | | | | | |
| 5 | | | | | |

**Danh sách kiểm tra quản trị dữ liệu (Data governance checklist):**
- [ ] Tập tài liệu (Corpus) chỉ chứa nguồn công khai/được phép dùng và không chứa dữ liệu cá nhân, thông tin đăng nhập hoặc tài liệu nội bộ.
- [ ] Mỗi tài liệu có `source_url`, `retrieved_at`, `document_version` (hoặc ngày hiệu lực) trong metadata.

### Cấu trúc Metadata (Metadata Schema)

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho truy xuất (retrieval)? |
|----------------|------|---------------|-------------------------------|
| | | | |
| | | | |

---

## 2. Thiết kế chiến lược (Strategy Design) — Nhóm (15 điểm)

> Mỗi thành viên thử **một chiến lược khác nhau** trên cùng bộ tài liệu; nhóm tổng hợp và so sánh ở đây.

### Phân tích đường cơ sở (Baseline Analysis)

Chạy `ChunkingStrategyComparator().compare()` trên 2-3 tài liệu:

| Tài liệu | Chiến lược (Strategy) | Số lượng Chunk | Độ dài trung bình | Giữ được ngữ cảnh không? |
|-----------|----------|-------------|------------|-------------------|
| Thời khóa biểu & Đăng ký học phần | FixedSizeChunker (`fixed_size`) | 8 | 481.25 | Trung bình; có thể cắt giữa câu hoặc giữa một bước đăng ký |
| Thời khóa biểu & Đăng ký học phần | SentenceChunker (`by_sentences`) | 16 | 238.50 | Khá tốt ở câu văn, nhưng các bước dạng dòng rời dễ bị tách khỏi tiêu đề |
| Thời khóa biểu & Đăng ký học phần | RecursiveChunker (`recursive`) | 8 | 469.25 | Tốt hơn fixed-size vì ưu tiên ranh giới đoạn và dòng |
| Chuyển đổi tín chỉ | FixedSizeChunker (`fixed_size`) | 7 | 459.29 | Trung bình; điều kiện 70% và điểm C có thể nằm gần ranh giới chunk |
| Chuyển đổi tín chỉ | SentenceChunker (`by_sentences`) | 3 | 1068.33 | Kém phù hợp; dữ liệu crawl có nhiều dòng không kết thúc bằng dấu câu nên chunk quá dài |
| Chuyển đổi tín chỉ | RecursiveChunker (`recursive`) | 7 | 445.86 | Khá tốt; các nhóm điều kiện theo dòng được giữ gần nhau |
| Bảng điểm & Chứng nhận | FixedSizeChunker (`fixed_size`) | 11 | 459.27 | Trung bình; bảng loại tài liệu và phí có thể bị cắt cơ học |
| Bảng điểm & Chứng nhận | SentenceChunker (`by_sentences`) | 10 | 502.30 | Trung bình; cấu trúc bảng dạng dòng làm ranh giới câu kém ổn định |
| Bảng điểm & Chứng nhận | RecursiveChunker (`recursive`) | 11 | 450.00 | Khá tốt; ưu tiên đoạn/dòng trước khi cắt nhỏ |

Số liệu trên được tạo bởi `python bench.py` sau khi tách bỏ frontmatter. Với corpus crawl theo nhiều dòng ngắn, `SentenceChunker` có thể tạo chunk rất dài vì nhiều mục không có dấu kết câu.

### Chiến lược của từng thành viên

> Mỗi thành viên điền một khối dưới đây (copy thêm nếu nhóm có nhiều hơn 3 người).

**Thành viên 1 — NgoGiaQuoc**
- **Loại chiến lược:** Custom `HeadingChunker`, `chunk_size=500`
- **Mô tả & lý do chọn cho chủ đề này:** Tài liệu quy định và dịch vụ học vụ được tổ chức thành các mục như điều kiện, quy trình, chi phí và lưu ý. Chunker tách theo tiêu đề để giữ mỗi mục thành một đơn vị ngữ nghĩa; nếu mục vượt giới hạn thì dùng `RecursiveChunker` và gắn lại tiêu đề vào từng mảnh con.
- **Code snippet (nếu custom):**
```python
class HeadingChunker:
    def _split_section(self, heading: str, body: str) -> list[str]:
        prefix = f"{heading}\n" if heading else ""
        pieces = RecursiveChunker(
            chunk_size=max(1, self.chunk_size - len(prefix))
        ).chunk(body)
        return [f"{prefix}{piece}".strip() for piece in pieces]
```

Implementation đầy đủ nằm trong `bench.py`. Tên thành viên được lấy theo thông tin Git hiện tại; các thành viên còn lại cần tự điền tên và chiến lược riêng.

**Thành viên 2 — [Tên]**
- **Loại chiến lược:**
- **Mô tả & lý do chọn:**
- **Code snippet (nếu custom):**

**Thành viên 3 — [Tên]**
- **Loại chiến lược:**
- **Mô tả & lý do chọn:**
- **Code snippet (nếu custom):**

### So Sánh Giữa Các Thành Viên

| Thành viên | Chiến lược (Strategy) | Điểm truy xuất (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Ngô Gia Quốc | Custom `HeadingChunker`, `chunk_size=500`, section dài fallback sang `RecursiveChunker` | 7/10 tạm tính với OpenAI `text-embedding-3-small` (chưa chạy agent) | Q1–Q3 lấy answer chunk ở top-1; giữ tiêu đề cùng nội dung section và bảo toàn các nhóm điều kiện/quy trình | Q4 thiếu section chứa thời gian; Q5 đúng tài liệu nhưng sai section; không có overlap nên mỗi section chứa đáp án chỉ có một cơ hội lọt top-3 |
| [Thành viên 2] | [Chiến lược khác] | Chưa có kết quả | Chờ kết quả benchmark cùng cấu hình | Chờ kết quả benchmark cùng cấu hình |
| [Thành viên 3] | [Chiến lược khác] | Chưa có kết quả | Chờ kết quả benchmark cùng cấu hình | Chờ kết quả benchmark cùng cấu hình |

**Chiến lược nào tốt nhất cho chủ đề này? Tại sao?**
> *Viết 2-3 câu — đây là phần được đánh giá cao nhất (khả năng suy nghĩ & giải thích):*

---

## 3. Câu hỏi đánh giá & Chất lượng truy xuất (Retrieval Quality) — Nhóm (10 điểm)

### Câu hỏi đánh giá & Câu trả lời chuẩn (nhóm thống nhất)

> **Đúng 5 câu hỏi**, đa dạng, có thể kiểm chứng; **ít nhất 1 câu** cần lọc metadata mới trả lời tốt. Đây là bộ câu hỏi chung cho mọi thành viên chạy.

| # | Câu hỏi (Query) | Câu trả lời chuẩn (Gold Answer) | Chunk nào chứa thông tin? |
|---|-------|-------------------------------|--------------------------|
| 1 | Tối đa bao nhiêu tín chỉ được phép chuyển vào chương trình đại học tại VinUni? | Tối đa 60 tín chỉ. | `chuyen-doi-tin-chi.md` — mục “Các lưu ý quan trọng” |
| 2 | Học phần cần đáp ứng mức tương đương nội dung và điểm tối thiểu nào để được xem xét chuyển đổi tín chỉ? | Nội dung tương đương ít nhất 70% và điểm tối thiểu là C hoặc tương đương. | `chuyen-doi-tin-chi.md` — mục “Điều kiện được xem xét chuyển đổi tín chỉ” |
| 3 | Trên SIS cần thao tác thế nào để hoàn tất đăng ký môn và trạng thái nào xác nhận đăng ký thành công? | Nhấn “Add”, sau đó “Register”; trạng thái phải là “Registered”. | `thoi-khoa-bieu-dang-ky-hoc-phan.md` — mục “Cách sử dụng SIS để đăng ký môn học” |
| 4 | Yêu cầu bảng điểm hoặc thư xác nhận thường mất bao lâu và phí mỗi bản là bao nhiêu? | Thông thường 2–3 ngày làm việc, có thể đến 5 ngày vào mùa cao điểm; phí 50.000 VNĐ mỗi bản. | `yeu-cau-cap-bang-diem-va-chung-nhan.md` — các mục “Cách Yêu Cầu Tài Liệu” và “Chi Phí & Giao Nhận” |
| 5 | Cần thực hiện những bước nào để đăng ký học phần? | Đăng nhập SIS, mở Course Registration, tìm môn, nhấn Add rồi Register và kiểm tra Your Class Schedule. Dùng `metadata_filter={"audience": "student"}`. | `thoi-khoa-bieu-dang-ky-hoc-phan.md` — mục “Cách sử dụng SIS để đăng ký môn học” |

### Tổng hợp chất lượng truy xuất của nhóm

> Cách chấm (theo `docs/SCORING.md`): **2 điểm/câu** — top-3 chứa chunk liên quan + agent trả lời đúng (2), có liên quan nhưng thiếu/không ở top-1 (1), không có trong top-3 (0).

| # | Câu hỏi | Chiến lược tốt nhất cho câu này | Có chunk liên quan trong top-3? | Ghi chú |
|---|---------|-------------------------------|-------------------------------|---------|
| 1 | Tối đa bao nhiêu tín chỉ được phép chuyển vào chương trình đại học tại VinUni? | `HeadingChunker` (kết quả hiện có) | Có, đầy đủ ở top-1 | `chuyen-doi-tin-chi#6`, score 0.7397, chứa trực tiếp “Tối đa 60 tín chỉ” |
| 2 | Học phần cần đáp ứng mức tương đương nội dung và điểm tối thiểu nào? | `HeadingChunker` (kết quả hiện có) | Có, đầy đủ ở top-1 | `chuyen-doi-tin-chi#4`, score 0.6350, chứa điều kiện 70% và điểm C |
| 3 | Trên SIS cần thao tác thế nào và trạng thái nào xác nhận đăng ký thành công? | `HeadingChunker` (kết quả hiện có) | Có, đầy đủ ở top-1 | `thoi-khoa-bieu-dang-ky-hoc-phan#5`, score 0.6863, chứa Add, Register và Registered |
| 4 | Yêu cầu bảng điểm hoặc thư xác nhận mất bao lâu và phí mỗi bản bao nhiêu? | `HeadingChunker` (kết quả hiện có) | Có một phần | Top-2 `yeu-cau-cap-bang-diem-va-chung-nhan#9` chứa phí 50.000 VNĐ/bản, nhưng top-3 thiếu thời gian 2–3 ngày và tối đa 5 ngày |
| 5 | Cần thực hiện những bước nào để đăng ký học phần? | `HeadingChunker` (kết quả hiện có) | Không đủ | Top-3 đều đúng tài liệu nhưng không có các chunk chứa trọn quy trình SIS theo gold answer |

Kết quả hiện có dùng OpenAI `text-embedding-3-small`: Q1–Q3 có answer chunk ở top-1, Q4 liên quan một phần và Q5 thiếu answer chunk. Điểm chất lượng context tạm tính là 7/10; chưa phải điểm rubric chính thức vì benchmark chưa gọi agent. Chỉ có thể kết luận chiến lược tốt nhất của nhóm sau khi các thành viên còn lại chạy cùng embedding backend và cấu hình.

**Lọc bằng metadata có giúp ích không? Ở câu hỏi nào?**
> Câu 5 được chạy A/B với `metadata_filter={"audience": "student"}` bằng OpenAI `text-embedding-3-small`. Top-3 có filter và không filter giống hệt nhau về chunk, score và thứ hạng vì ba kết quả cao nhất vốn đã thuộc `audience: student`. Vì vậy filter không giúp ích trong lượt chạy này; theo tiêu chí của lab, câu hỏi hiện tại chưa thực sự chứng minh được tác động của metadata filter.

---

## 4. Thuyết trình (Demo) & Bài học nhóm — Nhóm (5 điểm)

**Những phân tích (insights) hay nhất nhóm sẽ trình bày:**
> *Liệt kê 2-3 ý:*

**Bài học rút ra khi so sánh trong nhóm:**
> *Viết 2-3 câu — cùng tài liệu nhưng chiến lược khác nhau dẫn tới khác biệt gì?*

**Nếu làm lại, nhóm sẽ thay đổi gì trong chiến lược dữ liệu (data strategy)?**
> *Viết 2-3 câu:*

---

## Tự Đánh Giá (Phần Nhóm)

| Tiêu chí | Điểm tự đánh giá |
|----------|-------------------|
| Lựa chọn tài liệu (Document Set Quality) | / 10 |
| Thiết kế chiến lược (Strategy Design) | / 15 |
| Chất lượng truy xuất (Retrieval Quality) | / 10 |
| Thuyết trình (Demo) | / 5 |
| **Tổng phần nhóm** | **/ 40** |
