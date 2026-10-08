# Example research run

A real run of the application on the national-core library (laptop CPU, local qwen2.5:3b through Ollama),
exported from its result. The legal texts are public Vietnamese law.

**Question.** Người lao động làm việc đủ 12 tháng được nghỉ hằng năm bao nhiêu ngày?

**Status.** grounded — Mọi câu trả lời đều có căn cứ được trích dẫn.

**Answer.**

> Người lao động làm việc đủ 12 tháng được nghỉ hằng năm 12 ngày làm việc. [E2]

**Verification.**

| Sentence | Cites | Support | Result |
|---|---|---|---|
| Người lao động làm việc đủ 12 tháng được nghỉ hằng năm 12 ngày làm việc. | E2 | 0.86 | supported |

**Evidence** (items marked ● were given to the model; the others are shown to the reader):

| Key | Provision | Document | Effect status | How it was found |
|---|---|---|---|---|
| ● E1 | Điều 66 | Nghị định 145/2020/NĐ-CP | Hết hiệu lực một phần | retrieval (keyword rank 2, dense rank 2) |
| ● E2 | khoản 1–4 Điều 113 | Bộ luật 45/2019/QH14 | Hết hiệu lực một phần | retrieval (keyword rank 1, dense rank 4) |
| ● E3 | Điều 65 | Nghị định 145/2020/NĐ-CP | Hết hiệu lực một phần | retrieval (keyword rank 26, dense rank 9) |
| ● E4 | khoản 3 Điều 89 | Nghị định 145/2020/NĐ-CP | Hết hiệu lực một phần | retrieval (keyword rank 28, dense rank 12) |
| ● E5 | Điều 114 | Bộ luật 45/2019/QH14 | Hết hiệu lực một phần | retrieval (keyword rank 13, dense rank 28) |
| E6 | khoản 1–3 Điều 18 | Nghị định 12/2022/NĐ-CP | Còn hiệu lực | retrieval (keyword rank 22, dense rank 25) |
| E7 | khoản 3 Điều 8 | Nghị định 145/2020/NĐ-CP | Hết hiệu lực một phần | retrieval (keyword rank 30, dense rank 22) |
| E8 | Điều 7 | Nghị định 121/2014/NĐ-CP | Còn hiệu lực | retrieval (keyword rank 17, dense rank 46) |
| ● E9 | Điều 61 | Bộ luật 45/2019/QH14 | Hết hiệu lực một phần | graph: được dẫn chiếu trong E3 |
| ● E10 | Điều 115 | Bộ luật 45/2019/QH14 | Hết hiệu lực một phần | graph: được dẫn chiếu trong E3 |
| E11 | Điều 112 | Nghị định 145/2020/NĐ-CP | Hết hiệu lực một phần | graph: được dẫn chiếu trong E1 |
| E12 | Điều 113 | Nghị định 145/2020/NĐ-CP | Hết hiệu lực một phần | graph: được dẫn chiếu trong E1 |
| E13 | Điều 114 | Nghị định 145/2020/NĐ-CP | Hết hiệu lực một phần | graph: được dẫn chiếu trong E1 |
| E14 | Điều 14 | Nghị định 145/2020/NĐ-CP | Hết hiệu lực một phần | graph: được dẫn chiếu trong E4 |

**Cited provision** (khoản 1–4 Điều 113, Bộ luật 45/2019/QH14):

> Điều 113. Nghỉ hằng năm
> 1. Người lao động làm việc đủ 12 tháng cho một người sử dụng lao động thì được nghỉ hằng năm, hưởng nguyên lương theo hợp đồng lao động như sau:
> a) 12 ngày làm việc đối với người làm công việc trong điều kiện bình thường;
> b) 14 ngày làm việc đối với người lao động chưa thành niên, lao động là người khuyết tật, người làm nghề, công việc nặng nhọc, độc hại, nguy hiểm;
> c) 16 ngày làm việc đối với người làm nghề, công việc đặc biệt nặng nhọc, độc hại, nguy hiểm.
> 2. Người lao động làm việc chưa đủ 12 tháng cho một người sử dụng lao động thì số ngày nghỉ hằng năm theo tỷ lệ tương ứng với số tháng làm việc.
> 3. Trường hợp do thôi việc, bị mất việc làm mà chưa nghỉ hằng năm hoặc chưa nghỉ hết số ngày nghỉ hằng năm thì được người sử dụng lao động thanh toán tiền lương cho những ngày chưa nghỉ.
> 4. Người sử dụng lao động có trách nhiệm quy định lịch nghỉ hằng năm sau khi tham khảo ý kiến của người lao động và phải thông báo trước cho người lao động biết. Người lao động có thể thỏa thuận với người sử dụng lao động để nghỉ hằng năm thành nhiều lần hoặc nghỉ gộp tối đa 03 năm một lần.

**Research steps** (as shown in the interface; no model reasoning is exposed):

| Step | Detail | Time |
|---|---|---|
| Phân tích câu hỏi | không nêu văn bản cụ thể | 0 ms |
| Tìm theo từ khóa | BM25 trên âm tiết và cụm hai âm tiết (FTS5) | 132 ms |
| Tìm theo ngữ nghĩa | multilingual-E5, cosine | 31 ms |
| Trọng số hiệu lực | giảm hạng văn bản hết hiệu lực hoặc đã bị thay thế (15 văn bản theo quan hệ); ưu tiên văn bản được nêu tên và văn bản hướng dẫn | 1 ms |
| Mở rộng qua đồ thị | 18 liên kết (sqlite); thêm 6 căn cứ liên quan | 8 ms |
| Kiểm tra hiệu lực | tình trạng hiệu lực và văn bản thay thế | 0 ms |
| Tổng hợp câu trả lời | qwen2.5:3b, 29 token | 3,441 ms |
| Kiểm tra trích dẫn | 2/2 câu được căn cứ hỗ trợ | 1 ms |

The generation time above is from a repeated run (the model server reuses its prompt cache); the median over the
evaluation questions is 65 seconds per question on this CPU, retrieval included.
