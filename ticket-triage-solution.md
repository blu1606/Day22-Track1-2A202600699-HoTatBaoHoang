### 1. Unit of Work

- **AI đang thực hiện công việc gì?**
  AI thực hiện phân tích ticket -> gán nhãn phù hợp -> đề xuất routing đến đúng phòng ban. Nếu không xác định được ý định của người dùng thì đánh `unknown`, nếu cần review thì chuyển qua hàng đợi cho con người xử lý.
- **Output cuối cùng được dùng bởi ai?**
  - **Hệ thống Auto-Routing Engine**: Để tự động phân bổ ticket vào các hàng đợi của các bộ phận.
  - **Nhân viên hỗ trợ (Support Agent)**: Xem trên UI inbox nội bộ để có thông tin tóm tắt và xử lý nhanh nhất.
- **Nếu sai, hậu quả vận hành là gì?**
  - Nếu routing sai -> tốn thời gian chuyển qua lại giữa các phòng ban (ping-pong), kéo dài thời gian phản hồi đầu tiên (FRT) -> mất trust của khách hàng.
  - Nếu AI gán nhãn sai hoặc bỏ sót cờ khẩn cấp cho khách hàng VIP/Enterprise đang gặp lỗi nghiêm trọng (như lỗi thanh toán, khóa tài khoản) -> vi phạm cam kết SLA, có nguy cơ mất hợp đồng lớn và tổn hại uy tín thương hiệu.

---

### 2. Output Contract tối thiểu

```json
{
  "ticket_id": "string",
  "customer_id": "string",
  "customer_tier": "free | pro | enterprise",
  "title": "string",
  "ai_suggestion": {
    "category": "Technical | Billing | Feature Request | Clarification Needed",
    "urgent_level": "High | Critical | null",
    "route_to": "technical_support | billing_ops | product_team | customer_support",
    "requires_human": "boolean",
    "reason": "string",
    "confident_score": "float [0.0 - 1.0]"
  }
}
```
*Lưu ý:* Cần giữ trường `customer_tier` trong contract để viết các cross-field code assertions.

---

### 3. Quality Question

AI có label đúng category, routing đúng team và escalation kịp thời cho các yêu cầu nghiêm trọng của doanh nghiệp nhằm đảm bảo cam kết SLA không?

---

### 4. Eval Decision Map

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do chọn nguồn chấm này |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **1. Định dạng Schema & Enums** | **X** | | | | Dùng thư viện check schema (như Zod/Pydantic) là chuẩn nhất, nhanh, rẻ, chính xác 100%. |
| **2. Phân loại (Category)** | **X** | | | | So khớp trực tiếp dạng String với nhãn mong muốn trong Golden Dataset đã được gắn sẵn. |
| **3. Mức khẩn cấp (Urgent level)** | **X** | | | | So khớp trực tiếp dạng String với nhãn mong muốn trong Golden Dataset. |
| **4. Định tuyến (Route to)** | **X** | | | | So khớp trực tiếp dạng String với nhãn mong muốn trong Golden Dataset. |
| **5. Cờ escalate (Requires human)** | **X** | | | | Check logic cứng bằng Code dựa trên business rule (Tier + Urgency). |
| **6. Lý do tóm tắt (Reason)** | | **X** | | | Kiểm tra ngữ nghĩa (coi có bịa đặt thông tin hay tóm tắt quá sơ sài không), code thường không tự chấm được. |
| **7. Đối chứng và Hiệu chỉnh (Support Lead)** | | | | **X** | Support Lead đóng vai trò là Domain Expert làm nguồn chuẩn gắn nhãn tập vàng (Golden Dataset), đồng thời review các case LLM Judge chấm với độ tự tin thấp (< 0.75) hoặc có tranh chấp. |


---

### 5. Kiểm tra tự động bằng code

*   **Kiểm tra 1: Schema & Allowed Enums**
    *   *Mô tả:* Check xem output JSON của Agent có đúng định dạng không, các trường `category`, `urgent_level`, `route_to` có nằm trong whitelist enums không.
    *   *Vì sao giao cho code:* Code check nhanh, chính xác 100%, chạy mượt trong CI.
*   **Kiểm tra 2: So khớp nhãn với Golden Dataset**
    *   *Mô tả:* So khớp trực tiếp `output.category == expected.category`, `output.urgent_level == expected.urgent_level`, và `output.route_to == expected.route_to`.
    *   *Vì sao giao cho code:* Đo đếm được chính xác độ lệch nhãn mà không cần gọi mô hình.
*   **Kiểm tra 3: Logic nghiệp vụ (Cross-field Validation)**
    *   *Mô tả:*
        *   Nếu `customer_tier == "enterprise"` và `urgent_level` là `"High"` hoặc `"Critical"` -> `requires_human` bắt buộc phải là `true`.
        *   Nếu `category == "Billing"` -> `route_to` không được phép là `"product_team"`.
        *   Nếu `ticket_content` có các từ khóa: `blocking work`, `locked out`, `account disabled` -> `urgent_level` không được phép là `low` hoặc `null`.
    *   *Vì sao giao cho code:* Đây là các rule nghiệp vụ cứng, biểu diễn bằng code logic `IF-THEN` rất đơn giản và chuẩn xác.
*   **Kiểm tra 4: Giới hạn độ tin cậy và latency**
    *   *Mô tả:* Check `confident_score` phải nằm trong range `[0.0, 1.0]`. Latency phản hồi phải dưới 2.0 giây.
    *   *Vì sao giao cho code:* Chỉ số đo lường dạng số, code bắt chuẩn nhất.

---

### 6. Tiêu chí chấm bằng LLM

*   **Tiêu chí 1: Groundedness (Tính trung thực / Không bịa đặt)**
    *   *Mô tả:* Đọc message gốc và đối chiếu với `reason`. Đánh **FAIL** nếu lý do tóm tắt của Agent tự bịa thêm các chi tiết không có trong ticket (như mã lỗi khác, lịch sử tài khoản, hoặc lời hứa hẹn xử lý).
    *   *Vì sao code không bắt tốt:* Code không thể hiểu ngữ nghĩa để phát hiện thông tin tự suy diễn của mô hình.
*   **Tiêu chí 2: Completeness (Tính đầy đủ)**
    *   *Mô tả:* Đánh **FAIL** nếu lý do tóm tắt bỏ sót vấn đề chính của người dùng (ví dụ: người dùng báo bị charge tiền 2 lần và muốn hủy tài khoản, nhưng Agent chỉ tóm tắt là "muốn hủy tài khoản").
    *   *Vì sao code không bắt tốt:* Cần hiểu ngữ cảnh để xem Agent có chọn lọc và tóm tắt đầy đủ các ý cốt lõi không.

---

### 7. Human / Expert Review

*   **Ai cần review?**
    *   **Support Lead / Support Supervisor**: Đây chính là **Domain Expert về mặt vận hành** (Operational Domain Expert). Họ là người nắm rõ nhất về phân loại nghiệp vụ, kịch bản hỗ trợ, và cam kết SLA của phòng phòng khám/công ty SaaS.
*   **Review những case nào?**
    *   Các case mà LLM Judge chấm **FAIL** hoặc có độ tự tin của Agent/Judge thấp (`confident_score` < 0.75).
    *   Các case có sự mâu thuẫn giữa LLM Judge và Code assertions.
    *   Lấy mẫu ngẫu nhiên (Random audit) khoảng 5-10% tổng số ticket hàng ngày để phát hiện intent mới (data drift).
*   **Có cần domain expert không?**
    *   *Có.* Support Lead đóng vai trò là Domain Expert chịu trách nhiệm duyệt nhãn phân loại và hiệu chỉnh hệ thống.
    
#### 7A. Màn hình cho Domain Expert (ASCII)

```text
+-----------------------------------------------------------------------------------------+
| MÀN HÌNH ĐỐI CHỨNG VÀ AUDIT AI TRIAGE (DÀNH CHO SUPPORT LEAD)                           |
+-----------------------------------------------------------------------------------------+
| [TICKET] T-115 | Khách hàng: Minh Phát Logistics | Phân khúc: Enterprise                |
| Tiêu đề: Lỗi đăng nhập sau khi reset mật khẩu                                           |
| Nội dung: "Reset mật khẩu 2 lần rồi mà tài khoản admin vẫn không vào được..."           |
|-----------------------------------------------------------------------------------------|
| [AI GỢI Ý]                                                                              |
| - Danh mục (Category):  [ Technical ]        --> (Expected: Technical)                  |
| - Mức khẩn (Urgency):   [ High      ]        --> (Expected: Critical)                   |
| - Định tuyến (Route):   [ Technical Support] --> (Expected: Technical Support)          |
| - Cần người (Escalate): [ Có        ]        --> (Expected: Có)                         |
| - Độ tin cậy (Confidence): 0.91                                                         |
| - Lý do: "Lỗi đăng nhập admin chặn công việc của khách hàng doanh nghiệp"               |
|-----------------------------------------------------------------------------------------|
| [KIỂM TRA TỰ ĐỘNG BẰNG CODE]                                                            |
| - Schema: PASS                                                                          |
| - Cross-field: FAIL - Cảnh báo: Khách Enterprise gặp lỗi chặn công việc phải là Critical|
|-----------------------------------------------------------------------------------------|
| [QUYẾT ĐỊNH CỦA SUPPORT LEAD]                                                           |
| Chấp nhận gợi ý AI: [ YES ]  [ NO (Ghi đè nhãn bên dưới) ]                              |
| Ghi đè Danh mục:    ( ) Technical  ( ) Billing  ( ) Feature Request                     |
| Ghi đè Mức khẩn:    ( ) High       (x) Critical ( ) null                                |
| Ghi đè Định tuyến:  ( ) Tech Supp  ( ) Billing  ( ) Product                             |
| Ghi đè Cờ escalate: ( ) Có         ( ) Không                                            |
| Ý kiến của Lead:    "Khách bị chặn công việc nghiêm trọng cần nâng lên Critical."       |
|-----------------------------------------------------------------------------------------|
|                      [ LƯU & TIẾP TỤC ]    [ BỎ QUA ]                                   |
+-----------------------------------------------------------------------------------------+
```

#### 7B. Tiêu chí review của Domain Expert (Support Lead)

1.  **Tính chính xác của Phân loại (Category Accuracy):** Đảm bảo AI không nhầm lẫn giữa lỗi kỹ thuật (Technical) và nghiệp vụ tài chính (Billing).
2.  **Tính đúng hạn của Escalation (Escalation SLA):** Kiểm tra xem mọi khách hàng Enterprise có dấu hiệu bị chặn công việc đều được gắn đúng cờ `requires_human = true` và nâng urgency lên mức `Critical` để không trễ SLA.
3.  **Độ trung thực của lý do (Summarization Grounding):** Lý do AI tóm tắt phải cực kỳ ngắn gọn (dưới 20 từ) và tuyệt đối không suy diễn thông tin nằm ngoài nội dung ticket.


---

### 8. Release Gate

*   **Điều kiện Chặn (Block Release):**
    *   Số lỗi nghiêm trọng P0 = 0 (Không bị lỗi lộ dữ liệu nhạy cảm UUID/email, không có lỗi bỏ sót cờ escalation của khách hàng VIP/Enterprise).
    *   Tỷ lệ pass schema (`schema_pass_rate`) = 100%.
    *   Độ chính xác phân loại danh mục (`category_accuracy`) >= 90% trên tập tham chiếu.
    *   Tỷ lệ bắt cờ escalation (`escalation_recall`) >= 95%.
    *   Tỷ lệ bịa đặt thông tin (Hallucination) do LLM Judge chấm = 0.
*   **Điều kiện Cảnh báo (Warn):**
    *   Chi phí API trung bình mỗi ticket (`average_cost`) tăng > 15% so với baseline.
    *   Thời gian phản hồi trung bình (`average_latency`) tăng > 20% so với baseline.
    *   Tỷ lệ bất đồng ý kiến giữa LLM Judge và Human (`judge_disagreement_rate`) > 10%.

---

### 9. Kế hoạch chạy thử và dự toán chi phí

#### Giả định bộ số thử nghiệm (Pilot Setups):
*   Tập test pilot dataset: **100 cases** (bao gồm đầy đủ happy path, edge cases, ambiguous inputs).
*   Số lần chạy thử / tối ưu prompt (Iterations): **40 lần chạy**.
*   Model làm Agent chính (Worker): **gpt-5.4-nano** (giá siêu rẻ, phù hợp xử lý text hàng ngày).
*   Model làm LLM Judge: **gpt-5.4-mini** (độ thông minh vượt trội hơn bản nano để đóng vai trò kiểm tra lỗi).

#### Giá API thật (Phương án siêu tiết kiệm):
*   **gpt-5.4-nano**: $0.20/1M input tokens, $1.25/1M output tokens.
*   **gpt-5.4-mini**: $0.75/1M input tokens, $4.50/1M output tokens.

#### Tính toán chi phí API:
*   Tổng số request chạy thử: 100 cases * 40 lần = 4,000 requests.
*   *Chi phí Agent (gpt-5.4-nano)*: 
    *   Input trung bình: 1,500 tokens/request.
    *   Output trung bình: 200 tokens/request.
    *   Mỗi request: (1,500 * $0.20 + 200 * $1.25) / 1,000,000 = $0.00055.
    *   Tổng chi phí Agent: 4,000 * $0.00055 = **$2.20**.
*   *Chi phí LLM Judge (gpt-5.4-mini)*:
    *   Input trung bình (bao gồm prompt judge, message gốc và output của agent): 2,000 tokens/request.
    *   Output trung bình: 150 tokens/request.
    *   Mỗi request: (2,000 * $0.75 + 150 * $4.50) / 1,000,000 = $0.002175.
    *   Tổng chi phí Judge: 4,000 * $0.002175 = **$8.70**.
*   **Tổng chi phí API key:** $2.20 + $8.70 = **$10.90** (khoảng ~270,000 VND).

#### Ước lượng giờ công nhân sự (Tối ưu hóa nhờ trợ lý AI):
*   **PM / Thiết kế Eval:** 3 giờ (Sử dụng AI hỗ trợ sinh nhanh các edge cases, phác thảo prompt test và định nghĩa rubrics).
*   **Vận hành / Kỹ thuật (Developer):** 5 giờ (Sử dụng AI code assistant để sinh schema check, setup code assertion rules và kết nối API LLM Judge).
*   **Human Review (Support Lead):** 2 giờ (Sử dụng AI hỗ trợ gắn nhãn hàng loạt cho Golden Dataset, chỉ tập trung review các case tranh chấp thực tế).
*   **Domain Expert:** 0 giờ (Không áp dụng).

#### Tổng chi phí & Thời gian dự kiến:
*   **Tổng chi phí Pilot:** Khoảng 10 giờ công nhân sự nội bộ + $10.90 chi phí API.
*   **Tổng thời gian dự kiến:** 3 ngày (Thay vì 2 tuần nếu làm thủ công từ đầu).

#### Đánh giá tính khả thi:
*   Giá API được lấy trực tiếp từ biểu phí của phương án siêu tiết kiệm (gpt-5.4-nano và gpt-5.4-mini).
*   Nhờ ứng dụng AI vào các khâu thiết kế bộ test, viết code kiểm thử tự động và hỗ trợ dán nhãn, tổng giờ công của team được tối ưu giảm đi hơn 4 lần (từ 45 giờ xuống còn 10 giờ).
*   Với chi phí API cực rẻ (~270,000 VND) và thời gian thực hiện chỉ trong 3 ngày, dự án pilot này có tính khả thi cực kỳ cao, giúp chứng minh nhanh chất lượng phân loại của AI trước khi quyết định đầu tư nhân lực phát triển sâu.

---

### 10. Đề xuất thêm 5 Dataset Edge Cases

1.  **Happy path**:
    *   *Input:* Ticket gửi: "Mình không thể đăng nhập sau khi reset mật khẩu, tài khoản admin là abc.vn". Khách hàng phân khúc `free`.
    *   *Bắt failure:* Đảm bảo AI phân loại đúng `category` = `Technical`, `route_to` = `technical_support` và không kích hoạt nhầm cờ `requires_human = true` (vì không phải khách VIP/Enterprise).
2.  **Ambiguous input**:
    *   *Input:* Ticket chỉ có tiêu đề "Lỗi" và nội dung "Giúp tôi với gấp lắm".
    *   *Bắt failure:* Đảm bảo AI không tự suy đoán (hallucinate) phân loại bừa bãi; kỳ vọng gán đúng `category` = `Clarification Needed` và `route_to` = `customer_support`.
3.  **Missing information**:
    *   *Input:* Ticket gửi: "Tôi bị lỗi thanh toán khi gia hạn gói Pro". Không có ID giao dịch hay SĐT.
    *   *Bắt failure:* Đảm bảo AI nhận diện đúng danh mục `category` = `Billing` nhưng phần tóm tắt lý do (`reason`) không được tự bịa ra thông tin giao dịch hoặc lịch sử tài khoản.
4.  **High-risk / escalation (Enterprise VIP)**:
    *   *Input:* Ticket gửi: "Hệ thống billing bị lỗi làm tài khoản Enterprise của bên mình bị khóa. Yêu cầu xử lý gấp". Khách hàng phân khúc `enterprise`.
    *   *Bắt failure:* Đảm bảo AI kích hoạt đúng `requires_human` = `true` và `urgent_level` = `Critical` theo đúng business rules để đẩy lên hàng đợi ưu tiên cao nhất của `billing_ops`.
5.  **Regression case (Robustness tiếng Việt viết tắt/không dấu)**:
    *   *Input:* Ticket gửi: "reset mk 2 lan roi ma tk admin van ko vao dc. dang bi chan cong viec".
    *   *Bắt failure:* Đảm bảo hệ thống AI không bị lỗi phân loại hay định tuyến (không bị route nhầm sang Feature Request hay Billing) do ngôn ngữ viết tắt ("mk", "tk", "ko", "dc") hoặc không dấu của người Việt.

