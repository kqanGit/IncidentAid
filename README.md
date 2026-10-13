# IncidentAid

## 1. Vấn đề và người dùng

**Người dùng và tình huống.** Minh là kỹ sư backend junior, lần đầu trực sự cố (on-call) cho một nền tảng thanh toán. Lúc 2 giờ sáng, ba service cùng báo động: API gateway trả `504 Gateway Timeout`, payment service báo `PoolTimeout`, database có truy vấn chậm.

**Vấn đề.** Minh không phân biệt được service nào là nguyên nhân gốc và service nào chỉ là triệu chứng, nên không biết phải theo runbook nào và chạy lệnh gì.

**Hiện tại Minh làm gì.** Minh dán stack trace vào ô tìm kiếm của Slack và Confluence, dò theo chuỗi lỗi bề mặt, restart service trông tệ nhất, và nếu không được thì gọi dậy một kỹ sư senior.

## 2. Tính năng LLM và cái giá khi nó sai

**Gợi ý triage.** Minh mở một sự cố và dán log thô của nhiều service. LLM trả về một kết quả có cấu trúc: (a) service gây lỗi gốc, kèm các dòng log làm bằng chứng; (b) đúng một runbook phù hợp trong 15 runbook của nhóm, kèm đoạn trích dẫn; (c) mẫu lệnh của runbook đó với tham số đã điền từ log, ví dụ `docker restart <container_id>`. Màn hình hiển thị log, đoạn runbook và lệnh cạnh nhau. Minh kiểm tra, có thể sửa tham số, rồi copy lệnh sang terminal. **App không bao giờ tự chạy lệnh.** Khi đóng sự cố, app soạn sẵn một bản nháp post-mortem từ audit log để Minh chỉnh sửa; phần này không phải tính năng được đánh giá.

**Cái giá khi sai.** Nếu LLM chỉ nhầm service hoặc điền nhầm tham số (ví dụ database primary thay vì read replica), Minh sẽ restart nhầm: lỗi thật vẫn tiếp diễn, còn lần restart gây thêm một đợt gián đoạn. *Ai bị ảnh hưởng:* khách hàng đang thanh toán trên nền tảng, và Minh. *Bao nhiêu:* giả định nền tảng xử lý 200 giao dịch/phút, thêm 15 phút gián đoạn là khoảng 3.000 giao dịch thất bại và vi phạm SLA. *Có hoàn tác được không:* không; giao dịch thất bại và thời gian mất đi không lấy lại được.

**Cơ chế an toàn.** LLM chỉ được chọn trong các mẫu lệnh đã duyệt, không bao giờ tự viết lệnh tự do. Mỗi mẫu khai báo sẵn các giá trị tham số hợp lệ (danh sách service và container của hệ thống demo); giá trị nằm ngoài danh sách, hoặc có toán tử shell (`;`, `&&`, `|`, `$()`), sẽ khóa nút Copy. Với mẫu rủi ro cao (database, lõi thanh toán), Minh phải gõ lại tên đối tượng thì mới copy được.

**Làm sao biết nó sai.** Offline: 30 kịch bản gắn nhãn thủ công, mỗi kịch bản có sẵn service, runbook và tham số đúng; nhóm báo cáo độ chính xác nguyên nhân gốc, độ chính xác runbook và tỉ lệ sai tham số. Khi sử dụng: mỗi tham số Minh sửa trước khi copy đều được ghi lại (tỉ lệ sửa tham số), và khi đóng sự cố Minh đánh dấu nguyên nhân gốc được gợi ý có đúng không.

## 3. Phạm vi

**Làm trong học kỳ:** ticket sự cố (Investigating → Triaging → Resolved) có audit log; gợi ý triage stream qua SSE lên màn hình 3 cột; bộ kiểm tra mẫu lệnh, danh sách tham số hợp lệ và bước xác nhận cho lệnh rủi ro cao; tiền xử lý log (gộp dòng trùng, bỏ `INFO`/`DEBUG`, che secret và email); 15 runbook và 30 kịch bản gắn nhãn cho một hệ thống demo (gateway, payment service, Postgres, Redis); script đánh giá; bản nháp post-mortem chỉnh sửa được.

**Cố ý không làm:** chạy lệnh, SSH, hay gọi API cloud/Kubernetes; tích hợp Datadog, PagerDuty, New Relic, Slack (log được dán vào hoặc gửi qua webhook giả lập); trace và metric; vector database (15 runbook đủ nhỏ để đưa thẳng vào prompt); tài khoản, phân quyền và SSO (dùng một tài khoản demo); giao diện mobile.

## 4. Kế hoạch và phân công

Thời gian tính theo tuần kể từ khi bắt đầu dự án; mỗi checkpoint kéo dài 2 tuần.

Ba giai đoạn: **GĐ1 – Nền tảng và dữ liệu** (CP1–2), **GĐ2 – Xây dựng tính năng** (CP3–4), **GĐ3 – Đánh giá và hoàn thiện** (CP5–6).

| GĐ · CP | Thời gian | Phụ trách | Sản phẩm bàn giao | Hỗ trợ |
|---|---|---|---|---|
| 1 · CP1 | Tuần 1–2 | Trần Trọng Quân | Repo, Docker Compose, schema Postgres; viết và bấm giờ 3 kịch bản đầu | Bùi Minh Quân: kịch bản |
| 1 · CP2 | Tuần 3–4 | Bùi Minh Quân | Gọi LLM có output cấu trúc, prompt triage; 15 runbook; kịch bản 4–15 | Nguyễn Quốc Lộc: wireframe |
| 2 · CP3 | Tuần 5–6 | Trần Trọng Quân | API sự cố, SSE, bộ kiểm tra lệnh và danh sách tham số, tiền xử lý và che log | Bùi Minh Quân: kịch bản 16–30 |
| 2 · CP4 | Tuần 7–8 | Nguyễn Quốc Lộc | Màn hình 3 cột, khóa Copy, xác nhận lệnh rủi ro cao, ghi lại tham số bị sửa | Trần Trọng Quân: sửa API |
| 3 · CP5 | Tuần 9–10 | Bùi Minh Quân | Chạy đánh giá trên 30 kịch bản, chỉnh prompt, bản nháp post-mortem | Nguyễn Quốc Lộc: trang kết quả |
| 3 · CP6 | Tuần 11–12 | Nguyễn Quốc Lộc | Xử lý lỗi trên giao diện, khởi chạy bằng một lệnh, demo (một triage đúng, một lệnh bị chặn) | Cả nhóm |

## 5. Rủi ro

**1. Không kịp xây dựng kịch bản gắn nhãn đủ thật.** Không có bộ dữ liệu công khai nào ghép log nhiều service với nguyên nhân gốc và runbook đã gắn nhãn (`danluu/post-mortems` là văn xuôi; log của LogHub không gắn nhãn và chỉ từ một hệ thống). Kịch bản đơn giản thì demo vô nghĩa; kịch bản thực tế thì viết rất lâu. *Tuần này:* viết và bấm giờ 3 kịch bản. Nếu một kịch bản mất hơn 1 giờ, giảm mục tiêu xuống 20 và dùng script sinh log nhiễu xung quanh các dòng lỗi gốc viết tay.

**2. Log dán vào quá dài khiến triage quá chậm hoặc quá tốn tiền.** Một bản log thật có thể dài hàng chục nghìn dòng, và một gợi ý mất hơn 30 giây là vô dụng lúc 2 giờ sáng. *Tuần này:* làm bộ tiền xử lý trước, đo kích thước prompt trên 3 kịch bản, và giới hạn phần log khoảng 4.000 token, giữ các dòng `ERROR`/`WARN` và lần xuất hiện đầu tiên của mỗi dòng lặp.

## 6. Lựa chọn công nghệ

- **Next.js + Tailwind:** một framework cho cả màn hình xử lý sự cố và giao diện streaming; nhóm đã quen React.
- **FastAPI:** endpoint bất đồng bộ và SSE; giữ code LLM, bộ kiểm tra lệnh và script đánh giá cùng bằng Python.
- **PostgreSQL:** sự cố, audit log và mẫu lệnh là dữ liệu quan hệ và cần lưu lịch sử.
- **Pydantic:** một schema dùng chung cho output có cấu trúc của LLM, API và script đánh giá.
- **Gemini API, `gemini-3.6-flash`:** hỗ trợ output JSON có cấu trúc với chi phí thấp: $0,75 / 1 triệu token đầu vào và $3,75 / 1 triệu token đầu ra (gói trả phí, đến 31/12/2026). Một lần triage ≈ 8 nghìn token vào + 1 nghìn token ra ≈ **$0,01**; năm lần chạy đánh giá (150 lượt gọi) ≈ $1,50. Nhóm phát triển trên gói miễn phí và chỉ dùng log tự tạo, vì dữ liệu ở gói miễn phí có thể được Google sử dụng.
- **Docker Compose:** một lệnh khởi động web, API và database để chấm bài và demo.

## Thuật ngữ

| Thuật ngữ | Nghĩa |
|---|---|
| On-call (trực sự cố) | Kỹ sư được phân công xử lý sự cố của hệ thống ngoài giờ làm, kể cả ban đêm. |
| Service | Một phần riêng của hệ thống chạy độc lập, ví dụ cổng API, xử lý thanh toán, database. |
| Log | Dòng ghi lại sự kiện và lỗi mà mỗi service tự in ra trong lúc chạy. |
| Nguyên nhân gốc (root cause) | Lỗi đầu tiên gây ra sự cố; các service khác báo lỗi theo chỉ là triệu chứng. |
| Triage | Bước phân loại nhanh khi có sự cố: lỗi ở đâu, nghiêm trọng thế nào, làm gì trước. |
| Runbook | Tài liệu hướng dẫn từng bước xử lý một loại sự cố, thường kèm lệnh cần chạy. |
| Post-mortem | Báo cáo viết sau sự cố: chuyện gì xảy ra, vì sao, đã xử lý thế nào, rút kinh nghiệm gì. |
| Restart | Khởi động lại một service; thường làm service đó ngừng phục vụ trong chốc lát. |
| Primary / read replica | Database chính nhận ghi dữ liệu / bản sao chỉ dùng để đọc. Restart nhầm primary làm ngừng ghi giao dịch. |
| Container | Gói chạy một service (qua Docker); mỗi container có tên hoặc ID riêng. |
| SLA | Cam kết với khách hàng về thời gian hệ thống hoạt động; vi phạm có thể bị phạt. |
| SSE | Cách server đẩy dữ liệu dần dần về trình duyệt, để kết quả hiện ra từng phần thay vì chờ xong hết. |
| Token | Đơn vị đo độ dài văn bản mà LLM xử lý và tính tiền (khoảng 3–4 ký tự tiếng Anh). |
| Shell injection | Chèn ký tự đặc biệt (`;`, `&&`, `|`) vào lệnh để chạy thêm lệnh khác ngoài ý muốn. |
