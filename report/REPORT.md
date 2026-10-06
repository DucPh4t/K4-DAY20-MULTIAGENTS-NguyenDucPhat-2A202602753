# Báo cáo Lab: Self evolving Agentic

> Sao chép tệp này thành `report/REPORT.md` (đã làm ở Phần 0) và điền dần qua các Phần của lab. Xóa các dòng hướng dẫn dạng trích dẫn (bắt đầu bằng `>`). Văn phong kỹ thuật, ngắn gọn, mọi nhận định đi kèm số liệu hoặc bằng chứng. Trong buổi học: điền mục 1 đến 7 (bản nháp). Sau buổi học: hoàn thiện mục 8 đến 10.

## 1. Thông tin nhóm và cấu hình

| Họ tên | Mã sinh viên | Phần đóng góp |
|---|---|---|
| Nguyễn Đức Phát | 2A202602753 | 100% |

- Nhà cung cấp và mô hình (`LAB_MODEL`, không ghi khóa API), nhiệt độ (`LAB_TEMPERATURE`), `recursion_limit`: `google_genai:gemini-3.5-flash-lite`, nhiệt độ `0`, `recursion_limit`: `50` (mặc định của harness runner).
- Phiên bản Deep Agents (`pip show deepagents`), hệ điều hành, chạy trực tiếp hay trong Docker: `deepagents==0.7.21`, macOS (Darwin 24.1.0 arm64), chạy trực tiếp trong môi trường ảo `.venv` (không dùng Docker).
- Số lần chạy tác vụ đã dùng / ngân sách: 9 lần chạy phát triển (3 baseline learn + 3 subagents learn + 3 skills-auto learn) / ngân sách đánh giá chính thức (9 lần chạy đánh giá sau freeze).
- Commit của tag `freeze`: Sẽ được cập nhật sau khi tạo tag `freeze`.

## 2. Giả thuyết (commit TRƯỚC tag `freeze`, Phần 4.0)

- H1 (subagents so với baseline): Dự đoán điều kiện `subagents` sẽ có điểm số tương đương hoặc giảm nhẹ so với `baseline` (đặc biệt ở các check quy ước hình thức ngầm), trong khi chi phí token tăng mạnh từ 3x đến 5x và thời gian thực thi tăng từ 3x đến 6x. Căn cứ từ lý thuyết và thực nghiệm ở tập học: cơ chế chia tách tác vụ cho các subagent phi trạng thái (`explorer`, `implementer`, `reviewer`) dẫn đến phân mảnh ngữ cảnh (context fragmentation); các subagent chỉ nhận task prompt mà không kế thừa đầy đủ lịch sử hội thoại và các chỉ dẫn quy ước chi tiết của tác tử chính, dẫn đến việc giải quyết tốt logic lõi nhưng bỏ sót định dạng quy ước tổ chức.
- H2 (skills-auto so với baseline): Dự đoán điều kiện `skills-auto` sẽ đạt điểm số cao hơn rõ rệt so với `baseline` trên cả tập học và tập đánh giá, với chi phí token chỉ tăng nhẹ (~10-25% do đưa chỉ dẫn tóm tắt vào context). Căn cứ từ phân loại lỗi ở mục 4: 100% lỗi thất bại của `baseline` thuộc Nhóm E (Vi phạm quy ước tổ chức) - là những tri thức quy ước nội bộ không thể suy diễn từ logic nghiệp vụ chung. Bộ kỹ năng tự tiến hóa được curator cô đọng ngắn gọn và kích hoạt đúng ngữ cảnh sẽ bổ sung trực tiếp các tri thức quy ước này vào bộ nhớ làm việc của tác tử.
- H3 (tác vụ học so với tác vụ đánh giá): Dự đoán điểm số của `skills-auto` trên tác vụ học sẽ cải thiện vượt bậc so với baseline (đạt điểm cao ở các check quy ước đã học), trong khi trên tác vụ đánh giá (eval), điểm của `skills-auto` sẽ cao hơn baseline ở các quy ước dùng chung có tính khái quát (như tính toán tiền tệ integer cents, schema metadata, chuẩn format test/changelog), nhưng sẽ không đạt điểm tuyệt đối ở những quy ước mới (novel conventions) chỉ xuất hiện riêng ở tác vụ đánh giá. Căn cứ lý thuyết: kỹ năng học từ phản hồi quá khứ có khả năng chuyển giao (transferability) đối với các quy ước lặp lại, nhưng không thể đoán trước các quy tắc tổ chức chưa từng xuất hiện trong tập huấn luyện (bảo đảm không rò rỉ dữ liệu).

## 3. Làm quen Deep Agents (Phần 0.3)

1. Tác tử mặc định có 9 công cụ: `ls`, `read_file`, `write_file`, `edit_file`, `delete`, `glob`, `grep`, `execute`, `task`. Trong đó, công cụ duy nhất cho phép chạy lệnh shell trong sandbox là `execute`.
2. Mô tả của công cụ `task` nêu rõ: subagent `general-purpose` là tác tử đa năng dùng để nghiên cứu các câu hỏi phức tạp, tìm kiếm tệp/nội dung và thực hiện các tác vụ nhiều bước (có đầy đủ mọi công cụ như tác tử chính). Về ngữ cảnh: mỗi lần gọi subagent là độc lập và phi trạng thái theo mặc định (stateless by default); subagent chỉ nhìn thấy prompt mà tác tử chính truyền vào trực tiếp và trả về một báo cáo duy nhất, hoàn toàn không nhìn thấy lịch sử hội thoại trước đó của tác tử chính trừ khi kiểu subagent đó được cấu hình tường minh để kế thừa hội thoại.
3. Câu hướng dẫn hành vi trích từ mô tả công cụ:
   - Từ mô tả công cụ `task`: *"Launch multiple agents concurrently when their tasks are independent, using a single message with multiple tool calls."* (hoặc *"Put full detail in the prompt and state exactly what it should return — unless an agent type below says it inherits your conversation instead."*).
   - Từ mô tả công cụ `execute`: *"You MUST avoid using search commands like find and grep. Instead use the grep, glob tools to search. Use read_file rather than cat/head/tail."*

## 4. Đường cơ sở và phân loại lỗi (Phần 2.2)

| Tác vụ | Check thất bại | Nhóm lỗi (A-G) | Bằng chứng (trích ngắn từ `detail` hoặc vết) |
|---|---|---|---|
| data-learn | `rule_money_in_cents` | E | `RULE: money values in answer.json are integer cents (1606.67 USD is written 160667).` |
| data-learn | `rule_meta_block` | E | `RULE: answer.json has an object 'meta' = {"source": <input file name>, "rows_in": <number of data rows in the input file, duplicates included>, "rows_used": <number of distinct orders with a known amount>}.` |
| data-learn | `rule_clean_csv` | E | `RULE: write workspace/clean.csv with the header order_id,timestamp_utc,region,amount_cents; one row per distinct order with a known amount...` |
| code-learn | `rule_type_hints` | E | `RULE: every public function and method (name not starting with '_') has type annotations on all parameters and the return value.` |
| code-learn | `rule_regression_tests` | E | `RULE: write tests in tests/test_regressions.py covering every bug you fixed, with at least one test function per bug.` |
| code-learn | `rule_changelog` | E | `RULE: update CHANGELOG.md under '## Unreleased' with a bullet for every bug fixed, formatted as '- fix(<func>): <short description>'.` |
| logs-learn | `rule_service_names` | E | `RULE: service names in the output are lower-case with '-' replaced by '_' (payment-service -> payment_service).` |
| logs-learn | `rule_sorted_errors` | E | `RULE: top_errors is sorted by count descending, with ties broken by error_code ascending.` |
| logs-learn | `rule_schema_header` | E | `RULE: summary.json has schema_version = '2.1' and generated_by = 'log-analyzer'.` |

Nhận xét: Toàn bộ 9/9 (100%) check thất bại trên tập học của baseline đều thuộc Nhóm E (Vi phạm quy ước tổ chức). Ngược lại, 18/18 (100%) các check chức năng kỹ thuật (thuộc các nhóm A-D như logic xử lý ngày giờ, lọc duplicate, sửa bug mã nguồn, phân tích cú pháp log) đều đã đạt điểm tối đa. Bộ kỹ năng (Skill) hoàn toàn có thể phòng ngừa nhóm lỗi E này một cách hữu hiệu, vì các lỗi này xuất phát từ việc thiếu các quy ước tổ chức nội bộ mà tác tử không thể tự đoán trước nếu không được cung cấp hướng dẫn rõ ràng trong bộ nhớ ngữ cảnh.

## 5. Điều kiện `subagents` (Phần 2.3)

- Các subagent đã định nghĩa:
  - `explorer`: Khảo sát cấu trúc thư mục, tệp mã nguồn và tệp dữ liệu; tóm tắt đặc điểm mà không can thiệp tệp, giúp giảm bớt ngữ cảnh thô ban đầu cho tác tử chính.
  - `implementer`: Chuyên thực thi việc chỉnh sửa mã nguồn, viết code xử lý dữ liệu và tạo tệp kết quả theo thiết kế; được cấp đầy đủ công cụ tệp và shell execute.
  - `reviewer`: Kiểm tra chéo kết quả, chạy test tự động, rà soát tính hợp lệ của schema đầu ra và đối chiếu với các quy ước để phát hiện lỗi trước khi hoàn tất.
- `subagent_calls` ở từng tác vụ và nhận xét:
  - `data-learn`: 2 lần gọi (khảo sát dữ liệu và phân tích làm sạch).
  - `code-learn`: 5 lần gọi (khám phá bug -> sửa code -> chạy test -> kiểm tra chéo).
  - `logs-learn`: 2 lần gọi (đọc tệp log và định dạng JSON tổng hợp).
- Thông tin thiếu hoặc thừa khi giao việc: Tác tử chính phải mô tả lặp lại ngữ cảnh bài toán do subagent không thừa hưởng bộ nhớ đối thoại, gây tiêu hao token dư thừa. Đồng thời, tác tử chính thường chỉ tóm tắt mục tiêu nghiệp vụ lõi mà vô tình bỏ sót các yêu cầu quy ước ngầm, khiến subagent dù hoàn thành xuất sắc chức năng kỹ thuật vẫn vi phạm quy ước tổ chức.
- Ảnh hưởng đến token và thời gian: Chi phí token tăng vọt gấp 3-5 lần (từ 109k-156k token ở baseline lên 479k-733k token ở subagents), thời gian chạy tăng gấp 3-6 lần (từ 27-72s lên 164-295s). Điểm số không hề cải thiện (data-learn: 4/8 vs 5/8; code-learn: 6/10 vs 7/10; logs-learn: 3/9 vs 6/9), chứng minh kiến trúc đa tác tử phân cấp không mang lại hiệu quả chi phí cho các bài toán đơn lẻ có quy ước ngầm.

## 6. Self-evolving: skill do curator sinh (Phần 3)

- Số lần chạy curator: 1 lần. Số skill bị xóa: 0 (cả 3 skill đều tuân thủ nghiêm ngặt chuẩn YAML frontmatter, tên kebab-case, mô tả trigger rõ ràng, nội dung súc tích < 10 dòng, vượt qua hoàn toàn hàm `validate_skill`).

| Skill | Tổng quát hay riêng cho tác vụ học? | Đúng hay sai (nêu chỗ sai nếu có) | Độ dài, `description` và `skills_read` ở Phần 3.4 |
|---|---|---|---|
| `enforce-type-hints-and-tests` | Tổng quát cho các tác vụ bảo trì và phát triển mã nguồn Python. | Đúng hoàn toàn: hướng dẫn thêm type annotations cho mọi public function và tạo regression tests trong `tests/test_regressions.py`. | 8 dòng (484 bytes); description chỉ định rõ khi viết/sửa code Python; `skills_read` = 3 (được nạp và đọc ở các tác vụ). |
| `verify-output-schema-and-units` | Bán tổng quát: áp dụng cho các bài toán xử lý dữ liệu và log xuất ra JSON/CSV. | Đúng hoàn toàn: quy định chuẩn hóa tiền tệ dạng integer cents, thêm metadata top-level và chuẩn hóa tên service. | 9 dòng (657 bytes); description chỉ định khi sinh tệp đầu ra có cấu trúc; `skills_read` = 3 (được nạp và đọc ở các tác vụ). |
| `maintain-changelog` | Tổng quát: quy chuẩn ghi nhận lịch sử thay đổi mã nguồn trong các dự án phần mềm. | Đúng hoàn toàn: cập nhật `CHANGELOG.md` dưới mục `## Unreleased` theo định dạng `- fix(<func>): <mô tả>`. | 8 dòng (368 bytes); description chỉ định khi sửa bug hoặc chỉnh sửa dự án; `skills_read` = 3 (được nạp và đọc ở các tác vụ). |

## 7. Kết quả so sánh (Phần 4.3, 4.4)

> Dán nội dung `report/table.md` và kết quả `python scripts/check_breakdown.py`. Nêu các lần chạy có `error` hoặc `skills_modified = true` (nếu có) và cách xử lý.

```text
(dán bảng ở đây)
```

## 8. Phân tích

> Trả lời từng câu bằng số liệu từ mục 7 và bằng chứng từ vết. Kết quả âm hoặc không có khác biệt vẫn hợp lệ nếu được phân tích tốt.

1. So với `baseline`, điều kiện nào cải thiện điểm tác vụ **học**? Điều kiện nào cải thiện điểm tác vụ **đánh giá**? Có điều kiện nào cải thiện tác vụ học nhưng không cải thiện tác vụ đánh giá? Nếu có, đó là dấu hiệu gì?
2. Tách điểm thành check kỹ thuật và check quy ước (`rule_`). Skill do curator sinh giúp nhóm check nào? Check quy ước **mới** của tác vụ đánh giá có được skill giúp không, và vì sao?
3. Dựa vào vết và `skills_read`, giải thích một check mà skill giúp đạt và một check mà skill không giúp (skill chưa được đọc, đọc nhưng không làm theo, skill thiếu hoặc sai).
4. Chi phí: so sánh số token trung bình giữa các điều kiện. Điều kiện nào có hiệu quả tốt nhất theo điểm trên mỗi token? Đa tác tử có đáng chi phí trong thí nghiệm này không?
5. Có dấu hiệu rò rỉ dữ liệu hoặc quá khớp nào trong skill sinh ra không? Nhóm đã phòng tránh như thế nào?
6. Nhiễu: so sánh điểm tác vụ học của cùng bộ skill ở Phần 3.4 (đã sao lưu) và sau đóng băng. Chênh lệch bao nhiêu? Nó cho biết điều gì về độ tin cậy của các chênh lệch trong bảng ở mục 7?

## 9. Hạn chế và tính hợp lệ

> Nêu ít nhất 3 hạn chế và ảnh hưởng của từng hạn chế đến kết luận (ví dụ: chỉ 3 tác vụ mỗi vai trò, mỗi cấu hình chạy một lần, nhiễu của mô hình, tác vụ do giảng viên thiết kế sẵn quy ước, chỉ một mô hình).

1.
2.
3.

## 10. Kết luận

> Tối đa 5 câu. Chỉ khẳng định điều số liệu hỗ trợ. Nêu một đề xuất cải tiến tiếp theo.

## Phụ lục

- Lệnh đã chạy (theo thứ tự):
- Thử thách mở rộng (nếu có): hướng chọn, kết quả, nhận xét.
- Ghi chú khác:
