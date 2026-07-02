# TÀI LIỆU PHÂN TÍCH YÊU CẦU NGHIỆP VỤ (BRD) - HỆ THỐNG eKYC ABC BANK

---

## PART 1: PHÂN TÍCH HỆ THỐNG

### 1. Actors (Tác nhân hệ thống)
* **Khách hàng vãng lai (Prospective Customer):** Người dùng chưa có tài khoản, thực hiện tải app và thực hiện eKYC để mở tài khoản.
* **Hệ thống Core Banking (Core Banking System):** Hệ thống lõi của ngân hàng, tiếp nhận thông tin đã phê duyệt để khởi tạo số tài khoản tự động.
* **Hệ thống OCR & Face Matching (Dịch vụ eKYC bên thứ 3/Nội bộ):** Trích xuất dữ liệu từ CCCD và đối chiếu khuôn mặt.
* **Hệ thống Dữ liệu Dân cư Quốc gia (Cơ sở dữ liệu RAR - Bộ Công an):** Xác thực tính hợp pháp của thẻ CCCD chip (nếu có tích hợp NFC/API).
* **Hệ thống Quản trị rủi ro & Blacklist (Risk & Compliance System):** Kiểm tra xem khách hàng có nằm trong danh sách cấm, danh sách rửa tiền (AML/PEP) hay không.
* **Kiểm soát viên/Giao dịch viên (Back-office Auditor - Hạn chế tối đa):** Chỉ tham gia xử lý các ca lỗi rủi ro cao (Exception cases) được hệ thống cảnh báo.

### 2. Business Flow (Luồng nghiệp vụ)

#### A. Luồng chính (Happy Path - Zero Manual)
1.  Khách hàng tải ứng dụng ABC Bank, chọn "Mở tài khoản trực tuyến".
2.  Khách hàng nhập Số điện thoại, Email và xác thực OTP.
3.  Khách hàng chụp ảnh mặt trước và mặt sau của CCCD (Còn hạn sử dụng, không bị mờ/loá).
4.  Hệ thống chạy **OCR** để trích xuất thông tin (Họ tên, Số CCCD, Ngày sinh, Quê quán, Ngày hết hạn...).
5.  Khách hàng thực hiện **Liveness Check** (Quét khuôn mặt: quay trái, quay phải, nhìn thẳng, mỉm cười).
6.  Hệ thống đối chiếu ảnh chụp selfie với ảnh trên CCCD và kiểm tra chống giả mạo khuôn mặt (Anti-spoofing).
7.  Hệ thống check **Blacklist / AML / PEP**.
8.  Nếu tất cả các bước trên đều đạt điểm tin cậy (Confidence Score) $\ge 95\%$, hệ thống tự động đẩy dữ liệu sang Core Banking.
9.  Core Banking khởi tạo số tài khoản, gửi SMS/Email thông báo thành công cho khách hàng kèm thông tin đăng nhập.

#### B. Luồng ngoại lệ (Exception Path)
* **Lỗi OCR:** Giấy tờ bị mờ, rách hoặc hết hạn $\rightarrow$ Hệ thống hiển thị thông báo lỗi cụ thể và yêu cầu chụp lại tối đa 3 lần.
* **Lỗi Liveness Check:** Phát hiện giả mạo khuôn mặt (sử dụng ảnh chụp lại từ màn hình, mặt nạ...) $\rightarrow$ Từ chối ngay lập tức và khóa thiết bị/SĐT đó trong 24h.
* **Ngưỡng xám (Gray Zone):** Điểm đối chiếu khuôn mặt nằm trong khoảng $75\% - 94\%$ $\rightarrow$ Chuyển hồ sơ về hàng đợi (Queue) của Back-office để Giao dịch viên phê duyệt thủ công (Manual review).

### 3. Functional Requirements (Yêu cầu chức năng)

| ID | Nhóm chức năng | Mô tả yêu cầu |
| :--- | :--- | :--- |
| **FR-01** | Xác thực ban đầu | Hệ thống phải gửi được mã SMS OTP/Firebase OTP về SĐT của khách hàng trong vòng 5 giây và có hiệu lực trong 2 phút. |
| **FR-02** | Chụp & Trích xuất dữ liệu (OCR) | Tự động nhận diện loại giấy tờ (CCCD 12 số, CCCD gắn chip). Tự động cắt khung ảnh (Auto-cropping) và trích xuất text với độ chính xác cao. |
| **FR-03** | Nhận diện thực thể sống (Liveness) | Yêu cầu người dùng thực hiện các hành động ngẫu nhiên để đảm bảo là người thật. Phải chặn được deepfake và hình ảnh tĩnh. |
| **FR-04** | Điểm tin cậy (Face Matching) | So sánh ảnh chân dung eKYC và ảnh trên giấy tờ để trả về điểm số tương đồng (Matching Score) theo tỷ lệ phần trăm. |
| **FR-05** | Kiểm tra điều kiện (Sanity Check) | Tự động đối chiếu thông tin OCR xem khách hàng đã đủ 18 tuổi hay chưa. Check trùng SĐT/Số CCCD đã tồn tại trong hệ thống chưa. |
| **FR-06** | Cấp tài khoản tự động | Gọi API sang Core Banking tạo số tài khoản dựa trên cấu trúc định sẵn của ABC Bank khi có flag "APPROVED". |

### 4. Non-Functional Requirements (Yêu cầu phi chức năng)
* **Bảo mật (Security):** * Toàn bộ dữ liệu truyền tải qua API phải được mã hóa bằng giao thức HTTPS/TLS 1.3.
    * Thông tin nhạy cảm của khách hàng (Số CCCD, Số điện thoại) lưu trong DB phải được mã hóa (AES-256).
    * Hệ thống tuân thủ tiêu chuẩn PCI-DSS và các quy định về an toàn bảo mật thông tin ngân hàng.
* **Hiệu năng (Performance):** * Thời gian phản hồi (Response time) cho bước OCR và Face Matching không được quá 3 giây/giao dịch.
    * Hệ thống chịu tải (Throughput) tối thiểu 500 CCU (Concurrent Users) tại cùng một thời điểm.
* **Độ chính xác (Accuracy):**
    * Tỷ lệ nhận diện sai giấy tờ hợp lệ (FRR - False Rejection Rate) $< 2\%$.
    * Tỷ lệ chấp nhận giấy tờ/khuôn mặt giả mạo (FAR - False Acceptance Rate) $< 0.001\%$.
* **Tính sẵn sàng (Availability):** Cam kết SLA hệ thống đạt $99.9\%$, hoạt động 24/7 kể cả ngày lễ.

### 5. Assumptions & Business Rules (Giả định & Quy tắc nghiệp vụ)

> **BR-01 (Độ tuổi hợp lệ):** Khách hàng đăng ký mở tài khoản trực tuyến phải từ đủ 18 tuổi trở lên (tính theo ngày tháng năm sinh trên OCR).
> 
> **BR-02 (Quy tắc Zero-Manual):** Hồ sơ được duyệt tự động 100% nếu `OCR_Confidence_Score` $\ge 98\%$ VÀ `Face_Matching_Score` $\ge 95\%$ VÀ `Blacklist_Status` = Clean.
> 
> **BR-03 (Giới hạn thử lại):** Khách hàng chỉ được phép thực hiện eKYC lỗi tối đa 3 lần/ngày trên cùng 1 thiết bị/SĐT. Vượt quá giới hạn, hệ thống sẽ khóa tính năng eKYC của tài khoản đó trong 24 giờ.
> 
> **BR-04 (Hạn mức tài khoản eKYC):** Tài khoản mở qua eKYC tự động sẽ có hạn mức giao dịch tối đa (Transaction Limit) là 100 triệu VND/tháng theo quy định của Ngân hàng Nhà nước, trừ khi khách hàng ra quầy định danh lại.

---

## PART 2: CHUYỂN ĐỔI ARTIFACTS

### 1. Danh sách User Story

#### **User Story 1 (Cốt lõi - Đăng ký thông tin và OCR)**
* **As a** Khách hàng mới của ABC Bank,
* **I want to** chụp ảnh CCCD của mình để hệ thống tự động điền các thông tin cá nhân,
* **So that** tôi không phải nhập tay thủ công các trường thông tin dài, giúp tiết kiệm thời gian và tránh sai sót.
* **Acceptance Criteria (Tiêu chí nghiệm thu):**
    * *Scenario 1:* Chụp ảnh thành công trong điều kiện ánh sáng tốt. 
        * Given: Tôi ở màn hình chụp CCCD.
        * When: Tôi đưa mặt trước CCCD vào khung hình và bấm chụp.
        * Then: Ứng dụng tự động căn chỉnh, cắt góc và hiển thị thông tin trích xuất (Họ tên, số CCCD...) chính xác lên màn hình tiếp theo để tôi kiểm tra lại.
    * *Scenario 2:* Ảnh chụp bị mờ/loá/mất góc.
        * Given: Tôi chụp ảnh CCCD bị chói đèn.
        * When: Hệ thống quét ảnh.
        * Then: Hệ thống hiển thị thông báo lỗi "Ảnh bị loá sáng, vui lòng chụp lại" và không cho phép đi tiếp.

#### **User Story 2 (Cốt lõi - Face Matching & Phê duyệt)**
* **As a** Khách hàng mới của ABC Bank,
* **I want to** thực hiện quét khuôn mặt theo hướng dẫn,
* **So that** ngân hàng xác thực tôi chính là chủ nhân của giấy tờ và tự động kích hoạt tài khoản cho tôi ngay lập tức.
* **Acceptance Criteria (Tiêu chí nghiệm thu):**
    * *Scenario 1:* Xác thực trùng khớp.
        * Given: Khuôn mặt thật của tôi trùng khớp với ảnh trên CCCD đã chụp.
        * When: Tôi hoàn thành luồng quét Liveness Check.
        * Then: Hệ thống thông báo thành công và chuyển trạng thái hồ sơ thành "Approved", hiển thị số tài khoản mới trong vòng dưới 5 giây.

#### **Các User Story bổ sung khác:**
* **User Story 3:** *As a* Khách hàng, *I want to* nhận được mã định danh OTP qua SMS trước khi thực hiện eKYC *So that* đảm bảo số điện thoại đăng ký thuộc quyền sở hữu của tôi.
* **User Story 4:** *As a* Kiểm soát viên rủi ro, *I want to* nhận được các hồ sơ rơi vào "vùng xám" rủi ro *So that* tôi có thể thẩm định thủ công một cách nhanh chóng mà không làm gián đoạn trải nghiệm của khách hàng quá lâu.

### 2. Danh sách Use Case

| Use Case ID | Use Case Name | Primary Actor | Description |
| :--- | :--- | :--- | :--- |
| **UC-01** | Đăng ký & Xác thực SĐT | Khách hàng vãng lai | Khách hàng nhập SĐT, nhận mã OTP để kích hoạt phiên đăng ký eKYC. |
| **UC-02** | Quét & Trích xuất CCCD (OCR) | Khách hàng vãng lai | Khách hàng chụp 2 mặt giấy tờ, hệ thống OCR phân tích, điền data. |
| **UC-03** | Xác thực khuôn mặt (Liveness) | Khách hàng vãng lai | Hệ thống ghi hình khuôn mặt, kiểm tra chuyển động và so sánh đối chiếu với ảnh CCCD. |
| **UC-04** | Kiểm tra điều kiện & Rủi ro | Hệ thống (System) | Tự động kiểm tra tuổi, check trùng dữ liệu và quét danh sách Blacklist. |
| **UC-05** | Cấp số tài khoản tự động | Hệ thống (System) | Hệ thống tự động gọi API sang Core Banking mở tài khoản khi hồ sơ đạt chuẩn "Zero manual". |
| **UC-06** | Phê duyệt thủ công ca nghi vấn | Kiểm soát viên | Xử lý các hồ sơ eKYC bị chấm điểm thấp hoặc lỗi logic nghiệp vụ nhẹ. |
