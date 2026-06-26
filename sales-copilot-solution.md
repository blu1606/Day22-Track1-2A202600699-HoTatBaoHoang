### 1. Unit of Work

- **AI đang thực hiện công việc gì?**
  AI đọc tin nhắn mới nhất + lịch sử chat của khách hàng -> tự động trích xuất các tín hiệu định danh (SĐT, Email, Mã đơn, Mã khách) -> gọi API tra cứu chéo hệ thống CRM/OMS -> trả về tóm tắt hội thoại, cảnh báo dữ liệu nhập nhằng/mâu thuẫn, và gợi ý bước phản hồi tiếp theo cho sales.
- **Output cuối cùng được dùng bởi ai?**
  - **Nhân viên Sales / CSKH**: Đọc thông tin tóm tắt và cảnh báo trên UI hộp chat để đưa ra quyết định trả lời nhanh nhất.
  - **Hệ thống CRM / OMS nội bộ**: Nhận tín hiệu trigger để tự động hiển thị popup đơn hàng / hồ sơ khách hàng liên quan trên màn hình làm việc của nhân viên.
- **Nếu sai, hậu quả vận hành là gì?**
  - Nếu AI match sai mã đơn hàng hoặc nhầm thông tin khách hàng -> sales báo sai trạng thái đơn hàng cho khách (ví dụ: đơn đang giao nhưng báo đã giao thành công) -> gây mất uy tín nghiêm trọng, mất trust từ khách hàng.
  - Nếu AI tự ý chốt thông tin hoặc tự ý tạo đơn khi chưa có sự xác nhận của nhân viên -> làm sai lệch dữ liệu hệ thống CRM/OMS, tốn thời gian xử lý sự cố dữ liệu.

---

### 2. Output Contract tối thiểu

```json
{
  "conversation_id": "string",
  "extracted_signals": {
    "phone_numbers": ["string"],
    "emails": ["string"],
    "order_ids": ["string"],
    "customer_ids": ["string"]
  },
  "lookup_results": {
    "customer_profiles": [
      {
        "customer_id": "string",
        "customer_name": "string",
        "customer_tier": "string",
        "sales_owner": "string"
      }
    ],
    "order_suggestions": [
      {
        "order_id": "string",
        "product_name": "string",
        "delivery_status": "string",
        "estimated_delivery": "string",
        "is_fuzzy_match": "boolean"
      }
    ]
  },
  "status_flags": {
    "has_ambiguity": "boolean",
    "has_conflict": "boolean",
    "alert_message": "string | null"
  },
  "ai_suggestions": {
    "conversation_summary": "string",
    "suggested_action": "string",
    "draft_reply": "string"
  },
  "confident_score": "float [0.0 - 1.0]"
}
```

---

### 3. Quality Question

AI có trích xuất đúng các tín hiệu định danh, liên kết đúng hồ sơ khách hàng/đơn hàng và đưa ra cảnh báo kịp thời khi có sự nhập nhằng hoặc mâu thuẫn dữ liệu mà không tự ý hành động vượt quyền hạn không?

---

### 4. Eval Decision Map

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do chọn nguồn chấm này |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **1. Định dạng Schema & Enums** | **X** | | | | Check kiểu dữ liệu và cấu trúc JSON output bằng thư viện validate (nhanh, rẻ, chính xác 100%). |
| **2. Trích xuất tín hiệu (Signals)** | **X** | | | | So khớp regex định dạng SĐT, email, mã đơn và so khớp exact string với dữ liệu thật trong CRM/OMS. |
| **3. Cờ cảnh báo (Ambiguity / Conflict)** | **X** | | | | So khớp giá trị boolean dựa trên kết quả truy vấn database (ví dụ: đếm kết quả query SĐT > 1 -> has_ambiguity = true). |
| **4. Tóm tắt hội thoại (Summary)** | | **X** | | | Đọc hiểu ngữ nghĩa cuộc đối thoại tiếng Việt phức tạp (code không làm được). |
| **5. Gợi ý hành động & Nháp trả lời** | | **X** | | | Đánh giá tính phù hợp của giọng văn, tính an toàn thông tin và độ hữu ích đối với sales. |
| **6. Đối chứng và Hiệu chỉnh (Sales Lead)** | | | | **X** | Sales Lead đóng vai trò là Domain Expert làm nguồn chuẩn gắn nhãn tập vàng (Golden Dataset), đồng thời review các case LLM Judge chấm với độ tự tin thấp hoặc có tranh chấp. |

---

### 5. Kiểm tra tự động bằng code

*   **Kiểm tra 1: Định dạng Schema & Normalization**
    *   *Mô tả:* Validate schema của output. Email trích xuất phải được chuyển về lowercase, số điện thoại phải được chuẩn hóa về định dạng chuẩn quốc tế hoặc đầu số `0xxx`.
    *   *Vì sao chọn code:* Xử lý chuỗi và định dạng cực nhanh, chính xác 100% bằng regex/code.
*   **Kiểm tra 2: Ràng buộc tính nhập nhằng (Ambiguity Assertion)**
    *   *Mô tả:* Nếu truy vấn hệ thống trả về lớn hơn 1 hồ sơ khách hàng cho cùng 1 SĐT -> Kiểm tra xem trường `has_ambiguity` có bắt buộc bằng `true` và `alert_message` có chứa thông báo yêu cầu nhân viên xác nhận chọn tài khoản hoặc hỏi lại tên khách hàng không.
    *   *Vì sao chọn code:* Đây là logic nghiệp vụ cứng dựa trên số lượng bản ghi trả về từ database.
*   **Kiểm tra 3: Ràng buộc mâu thuẫn hệ thống (Conflict Assertion)**
    *   *Mô tả:* Nếu trạng thái đơn hàng trên CRM là "Đã hủy" nhưng trên OMS là "Đang giao" -> Kiểm tra xem trường `has_conflict` có bắt buộc bằng `true` không.
    *   *Vì sao chọn code:* So khớp logic chênh lệch trạng thái giữa hai cơ sở dữ liệu.
*   **Kiểm tra 4: Gợi ý tìm kiếm mờ (Fuzzy Match Warning)**
    *   *Mô tả:* Nếu mã đơn hàng trích xuất bị sai lệch 1-2 ký tự và được match mờ -> Kiểm tra xem `is_fuzzy_match` có bằng `true` không và `alert_message` có cảnh báo sales cần xác nhận lại không.
    *   *Vì sao chọn code:* Kiểm tra cờ logic cứng dựa trên thuật toán so khớp chuỗi (Levenshtein distance).
*   **Kiểm tra 5: Giới hạn hành vi an toàn (Action Safety check)**
    *   *Mô tả:* Kiểm tra xem AI có cố tình sinh ra lệnh gọi API thay đổi dữ liệu hoặc tự động gửi tin nhắn cho khách hàng mà không có sự xác nhận của nhân viên không.
    *   *Vì sao chọn code:* Dùng assertion kiểm tra xem hệ thống có chạy vượt ngoài luồng API cho phép hay không.

---

### 6. Tiêu chí chấm bằng LLM

*   **Tiêu chí 1: Hallucination trong Hồ sơ/Đơn hàng (Grounding)**
    *   *Mô tả:* So sánh phần tóm tắt và thông tin liên quan của AI với input chat gốc + data CRM/OMS trả về. Đánh **FAIL** nếu AI tự bịa ra thông tin đơn hàng (ví dụ: khách chưa mua máy lọc nước nhưng AI tóm tắt là khách hỏi mua máy lọc nước).
    *   *Vì sao code không bắt tốt:* Đọc hiểu ngữ nghĩa chéo giữa lịch sử chat và dữ liệu database để phát hiện tính trung thực.
*   **Tiêu chí 2: Tính hữu ích và An toàn của đề xuất (Action Empathy & Safety)**
    *   *Mô tả:* Đánh giá xem phần gợi ý bước tiếp theo và nháp trả lời có phù hợp với trạng thái hiện tại của khách hàng không. Đánh **FAIL** nếu khách đang khiếu nại (Angry) mà AI gợi ý sales upsell giới thiệu sản phẩm mới.
    *   *Vì sao code không bắt tốt:* Cần khả năng phân tích cảm xúc (sentiment) và chuẩn mực ứng xử của con người.

---

### 7. Human / Expert Review

*   **Ai cần review?**
    *   **Sales Lead / CSKH Supervisor**: Đây chính là **Domain Expert về mặt vận hành**. Họ là người nắm rõ quy trình chốt đơn, kịch bản xử lý lỗi dữ liệu và các tình huống đền bù/chăm sóc khách hàng.
*   **Review những case nào?**
    *   Các case có cờ `has_ambiguity` (trùng lặp SĐT) hoặc `is_fuzzy_match` (tìm kiếm mờ mã đơn) để kiểm tra xem sales xử lý thực tế có làm đúng quy trình xác nhận lại với khách hàng không.
    *   Các case mà LLM Judge chấm **FAIL** hoặc có độ tự tin của Agent/Judge thấp (`confident_score` < 0.70).
    *   Các case có sự mâu thuẫn giữa LLM Judge và Code assertions.
    *   Lấy mẫu ngẫu nhiên (Random audit) khoảng 5% tổng số phiên làm việc hàng ngày để phát hiện intent mới.
*   **Có cần domain expert không?**
    *   *Có.* Sales Lead đóng vai trò là Domain Expert chịu trách nhiệm duyệt nhãn phân loại, kịch bản và hiệu chỉnh hệ thống.

#### 7A. Màn hình cho Domain Expert (ASCII)

```text
+-----------------------------------------------------------------------------------------+
| MÀN HÌNH ĐỐI CHỨNG VÀ AUDIT AI SALES COPILOT (DÀNH CHO SALES LEAD)                      |
+-----------------------------------------------------------------------------------------+
| [CHAT HISTORY] Zalo OA | Khách hàng: Nguyễn Minh Linh                                   |
| Khách: "Chị check giúp em đơn hàng DH-4829 với ạ. SĐT em: 0909123456"                     |
|-----------------------------------------------------------------------------------------|
| [AI TRÍCH XUẤT & TRA CỨU]                                                                |
| - SĐT phát hiện: [ 0909123456 ] -> Trùng khớp 2 Hồ sơ khách hàng trên CRM (Ambiguity!)   |
|   1. Nguyễn Minh Linh (Zalo OA) | 2. Trần Minh Linh (Facebook Lead)                     |
| - Mã đơn phát hiện: [ DH-4829 ] -> Khớp mờ (Fuzzy Match 90%) với đơn DH-48291 trên OMS    |
|-----------------------------------------------------------------------------------------|
| [AI GỢI Ý & TRẠNG THÁI FLAGS]                                                           |
| - Cờ Ambiguity:  [ TRUE ]  - Cờ Fuzzy Match: [ TRUE ]                                    |
| - Cảnh báo: "SĐT trùng khớp nhiều khách hàng. Mã đơn hàng gần đúng, cần check lại."     |
| - Tóm tắt: Khách hỏi trạng thái đơn hàng khớp mờ, SĐT trùng lặp 2 hồ sơ.                |
| - Gợi ý hành động: Yêu cầu sales xác nhận lại tên khách hàng hoặc hỏi mã đơn đầy đủ.    |
| - Nháp trả lời: "Dạ cho em xin phép xác nhận mình là Nguyễn Minh Linh đúng không ạ?..." |
|-----------------------------------------------------------------------------------------|
| [KIỂM TRA TỰ ĐỘNG BẰNG CODE]                                                            |
| - Schema check: PASS                                                                    |
| - Assertion check: PASS (Đã bật cảnh báo trùng SĐT và khớp mờ đơn hàng)                  |
|-----------------------------------------------------------------------------------------|
| [QUYẾT ĐỊNH CỦA SALES LEAD]                                                             |
| Duyệt gợi ý AI:  [ APPROVE ]  [ REJECT (Ghi lý do bên dưới) ]                           |
| Ý kiến của Lead: "AI xử lý tốt case trùng SĐT và khớp mờ đơn hàng, nháp trả lời an toàn."|
|-----------------------------------------------------------------------------------------|
|                      [ LƯU & TIẾP TỤC ]    [ BỎ QUA ]                                   |
+-----------------------------------------------------------------------------------------+
```

#### 7B. Tiêu chí review của Domain Expert (Sales Lead)

1.  **Tính chính xác của Trích xuất (Entity Extraction Accuracy):** Đảm bảo AI trích xuất đúng và không bỏ sót số điện thoại, email hay mã đơn hàng trong văn cảnh tiếng Việt viết tắt/không dấu.
2.  **An toàn trong xử lý Nhập nhằng (Ambiguity Handling Safety):** Kiểm tra xem AI có kích hoạt đúng cờ cảnh báo khi SĐT/Email trùng lặp nhiều hồ sơ để ngăn lộ thông tin cá nhân hay không.
3.  **An toàn trong khớp mờ đơn hàng (Fuzzy Match Safety):** Đảm bảo mọi đơn hàng được khớp mờ (fuzzy matched) đều đi kèm cảnh báo rõ ràng cho sales kiểm tra chéo trước khi báo khách.

---

### 8. Release Gate

*   **Điều kiện Chặn (Block Release):**
    *   Tỷ lệ pass schema (`schema_pass_rate`) = 100%.
    *   Số lỗi nghiêm trọng P0 = 0 (Bao gồm: match nhầm thông tin đơn hàng của khách hàng khác dẫn đến lộ dữ liệu cá nhân; AI tự động thực thi sửa đổi dữ liệu hệ thống).
    *   Độ chính xác bắt đúng tín hiệu định danh (`signal_extraction_recall`) >= 95%.
    *   Tỷ lệ kích hoạt đúng cờ cảnh báo mâu thuẫn/trùng lặp (`conflict_warning_recall`) >= 98%.
*   **Điều kiện Cảnh báo (Warn):**
    *   Tỷ lệ bất đồng ý kiến giữa LLM Judge và Human (`judge_disagreement_rate`) > 8%.
    *   Thời gian xử lý trung bình tăng > 15% so với phiên bản cũ.

---

### 9. Kế hoạch chạy thử và dự toán chi phí

#### Giả định bộ số thử nghiệm (Pilot Setups):
*   Tập test pilot dataset: **100 cases** (đa dạng các segment khách hàng, các lỗi nhập trùng SĐT, mâu thuẫn đơn hàng).
*   Số lần chạy thử / tối ưu prompt (Iterations): **40 lần chạy**.
*   Model làm Agent chính (Worker): **gpt-5.4-nano** (phù hợp xử lý text thông thường, giá siêu rẻ).
*   Model làm LLM Judge: **gpt-5.4-mini** (độ thông minh vượt trội bản nano để đóng vai trò kiểm tra lỗi).

#### Giá API thật (Phương án siêu tiết kiệm):
*   **gpt-5.4-nano**: $0.20/1M input tokens, $1.25/1M output tokens.
*   **gpt-5.4-mini**: $0.75/1M input tokens, $4.50/1M output tokens.

#### Tính toán chi phí API:
*   Tổng số request chạy thử: 100 cases * 40 lần = 4,000 requests.
*   *Chi phí Agent (gpt-5.4-nano)*:
    *   Input trung bình (kèm hội thoại + data CRM/OMS): 2,000 tokens/request.
    *   Output trung bình: 300 tokens/request.
    *   Mỗi request: (2,000 * $0.20 + 300 * $1.25) / 1,000,000 = $0.000775.
    *   Tổng chi phí Agent: 4,000 * $0.000775 = **$3.10**.
*   *Chi phí LLM Judge (gpt-5.4-mini)*:
    *   Input trung bình (prompt + message + agent output): 2,500 tokens/request.
    *   Output trung bình: 150 tokens/request.
    *   Mỗi request: (2,500 * $0.75 + 150 * $4.50) / 1,000,000 = $0.00255.
    *   Tổng chi phí Judge: 4,000 * $0.00255 = **$10.20**.
*   **Tổng chi phí API key:** $3.10 + $10.20 = **$13.30** (khoảng ~340,000 VND).

#### Ước lượng giờ công nhân sự (Tối ưu hóa nhờ trợ lý AI):
*   **PM / Thiết kế Eval:** 3 giờ (Sử dụng AI hỗ trợ sinh nhanh các edge cases, phác thảo prompt test và định nghĩa rubrics).
*   **Vận hành / Kỹ thuật (Developer):** 5 giờ (Sử dụng AI code assistant để sinh schema check, setup code assertion rules và kết nối API LLM Judge).
*   **Human Review (Sales Lead):** 2 giờ (Sử dụng AI hỗ trợ gắn nhãn hàng loạt cho Golden Dataset, chỉ tập trung review các case tranh chấp thực tế).
*   **Domain Expert:** 0 giờ (Không tính thêm ngoài Sales Lead).

#### Tổng chi phí & Thời gian dự kiến:
*   **Tổng chi phí Pilot:** Khoảng 10 giờ công nhân sự nội bộ + $13.30 chi phí API.
*   **Tổng thời gian dự kiến:** 3 ngày.

#### Đánh giá tính khả thi:
*   Giá API được tính dựa trên biểu giá phương án siêu tiết kiệm (gpt-5.4-nano & gpt-5.4-mini).
*   Chi phí API cực kỳ tối ưu (~340,000 VND cho 4,000 lượt chạy) giúp team dễ dàng xin phê duyệt pilot. Quy mô 100 cases chạy lặp 40 lần đủ để tối ưu hóa prompt của Agent chính đạt độ chính xác cao và thiết lập cờ cảnh báo an toàn cho sales trước khi triển khai thực tế.

---

### 10. Đề xuất thêm 5 Dataset Edge Cases

1.  **Happy path (Match rõ, 1 bản ghi duy nhất):**
    *   *Input:* Khách nhắn: "Check giùm mình đơn DH-48291, SĐT là 0909123456". OMS có đúng 1 đơn DH-48291 khớp với SĐT này trên CRM.
    *   *Bắt failure:* Đảm bảo luồng cơ bản chạy tốt, AI nhận diện và trích xuất đúng thông tin đơn hàng không bị lệch nhãn.
2.  **Ambiguous lookup (Trùng SĐT):**
    *   *Input:* Khách nhắn: "Kiểm tra giúp đơn của mình, số mình là 0909123456". CRM trả về 2 khách hàng trùng số điện thoại này (nhập trùng).
    *   *Bắt failure:* Kiểm tra xem AI có kích hoạt đúng cờ `has_ambiguity = true` và đưa ra kịch bản đề xuất hỏi lại tên khách hàng chứ không tự ý chọn bừa.
3.  **Missing information (Không có tín hiệu):**
    *   *Input:* Khách nhắn: "Bên mình kiểm tra hộ đơn hàng của mình với, đang cần gấp".
    *   *Bắt failure:* Đảm bảo AI không tự suy đoán (hallucinate) ra khách hàng/đơn hàng bừa bãi, mà phải gợi ý sales hỏi thêm SĐT hoặc mã đơn.
4.  **Conflicting systems (Mâu thuẫn trạng thái):**
    *   *Input:* Khách gửi: "Check hộ đơn DH-48291". Hệ thống CRM báo khách hàng đã hủy đơn, nhưng OMS báo trạng thái đơn hàng là "Đang giao".
    *   *Bắt failure:* Test xem AI có phát hiện mâu thuẫn giữa 2 hệ thống (OMS vs CRM), bật cờ `has_conflict = true` và đưa cảnh báo cho sales xử lý ngoại lệ hay không.
5.  **Fuzzy Match case (Khớp mờ mã đơn hàng):**
    *   *Input:* Khách nhắn: "check don hang DH-4829 gium, sdt 0909 123 456. hom qua lay may loc nuoc".
    *   *Bắt failure:* Kiểm tra xem AI có nhận diện tốt mã đơn hàng bị thiếu ký tự (`DH-4829` thay vì `DH-48291`), trích xuất được và bật đúng cờ `is_fuzzy_match = true` cùng cảnh báo tương ứng để nhân viên sales kiểm tra chéo hay không.
