# BÁO CÁO SO SÁNH TRADE-OFF: KỸ THUẬT PROMPTING NÂNG CAO (LOGIC & CHUỖI)

## 1. Bối cảnh & Yêu cầu bài toán
* **Bối cảnh:** Hệ thống bán lẻ **TechShop** cần tự động hóa quy trình kiểm tra điều kiện đổi trả/bảo hành đơn hàng và soạn email phản hồi khách hàng từ dữ liệu khiếu nại thô.
* **Mục tiêu:** Đề xuất 2 phương án thiết kế Prompt (Đơn giản vs. Chi tiết), phân tích sự đánh đổi (Trade-off) về chi phí Token, độ chính xác logic và khả năng chặn lỗi biên, từ đó chọn ra phương án tối ưu nhất.

---

## 2. Bước 1: Đề xuất 2 Prompt Mẫu

### 🔹 Giải pháp 1: Prompt Đơn giản, Ngắn gọn (Single-Step Zero-shot CoT)
Gộp toàn bộ yêu cầu vào 1 câu lệnh ngắn, sử dụng kích hoạt CoT cơ bản.
```text
Bạn là nhân viên CSKH TechShop. Hãy đọc khiếu nại của khách, suy nghĩ từng bước (Let's think step by step) xem đơn hàng có đủ điều kiện đổi trả (mua <= 15 ngày, lỗi NSX) không, sau đó viết email phản hồi cho khách.

Dữ liệu: "Mình mua chuột không dây mã DH-102 ngày 10/11/2026, hôm nay 17/11/2026 bấm không ăn, mình không làm rơi. Đổi giúp mình nhé!"
```

### 🔹 Giải pháp 2: Prompt Chi tiết, Đa ràng buộc (Multi-Step Prompt Chaining + CoT + Chống đứt gãy)
Chia tách thành chuỗi các trạm đơn nhiệm (Data Handoff qua JSON), có ràng buộc cấm bịa đặt, kiểm soát lỗi biên (thiếu dữ liệu) và quy tắc nối đoạn nếu bị ngắt.
```text
[Trạm 1 - Trích xuất & Bắt lỗi biên]
Role: Hệ thống xử lý dữ liệu TechShop. Trích xuất thông tin từ <data> sang JSON.
Rules: Cấm đoán mò. Nếu thiếu "order_id" hoặc "purchase_date", gán null và bật "missing_data": true. Chỉ xuất JSON.
<data> {Dữ liệu khách hàng} </data>

[Trạm 2 - Lập luận CoT & Quyết định]
Role: Chuyên viên thẩm định (Hôm nay: 2026-11-17). Policy: Đổi mới 1-1 nếu <= 15 ngày & lỗi NSX.
Task: Nhận JSON Trạm 1. Nếu "missing_data": true -> xuất "PENDING_INFO". Ngược lại, suy nghĩ từng bước trong "reasoning" rồi xuất "decision" (APPROVED / REJECTED) dưới dạng JSON.

[Trạm 3 - Soạn Email & Quy tắc chống đứt gãy]
Role: Trưởng phòng CSKH. Dựa vào JSON Trạm 2, viết Email phản hồi chuẩn Markdown (không lời thừa).
*Lưu ý chống đứt gãy: Nếu phản hồi bị ngắt ngang, chỉ nối tiếp ngay sau 4 từ cuối cùng trước điểm ngắt.*
```

---

## 3. Bước 2: Bảng So sánh Trade-off

| Tiêu chí đánh giá | Giải pháp 1 (Đơn giản) | Giải pháp 2 (Chi tiết - Chaining & CoT) |
| :--- | :--- | :--- |
| **Độ dễ hiểu & Triển khai** | **Cao** (Viết nhanh trong 30 giây, dễ đọc) | **Trung bình** (Cần thiết kế cấu trúc JSON & luồng nối trạm) |
| **Tốn Token & Chi phí** | **Ít** (Chỉ tốn 1 lượt gọi API, ít chữ) | **Nhiều hơn** (Tốn thêm Token cho khung JSON, lập luận CoT & gọi nhiều trạm) |
| **Chặn lỗi biên (Edge Cases)** | **Trung bình** (Dễ bị AI tự bịa ngày/mã đơn nếu khách ghi thiếu) | **Cực tốt** (Khóa chặt bằng cờ `missing_data`, cấm đoán mò) |
| **Độ chính xác Logic (CoT)** | **Khá** (Dễ bị phân tâm vì vừa phải tính toán vừa phải viết văn) | **Rất cao** (Tách bạch trạm tính toán và trạm viết văn) |
| **Xử lý đứt gãy ngữ cảnh** | **Kém** (Dễ bị ngắt cụt hoặc trôi dạt vai trò khi xử lý nhiều đơn) | **Tối ưu** (Dữ liệu nén gọn qua JSON + cơ chế mớm câu nối tiếp) |

---

## 4. Bước 3: Lựa chọn Giải pháp Tối ưu & Kết quả Chạy thử

### 4.1. Lựa chọn & Lý do
* **Phương án được chọn:** **Giải pháp 2 (Prompt Chi tiết - Chaining & CoT)**.
* **Lý do:** Trong môi trường vận hành thực tế của cửa hàng, chi phí Token tăng thêm ở Giải pháp 2 là rất nhỏ so với thiệt hại tài chính nếu AI duyệt nhầm đơn hàng quá hạn hoặc tự bịa thông tin (Hallucination). Giải pháp 2 cô lập được lỗi ở từng trạm, chặn đứng 100% bẫy dữ liệu trống và đảm bảo đầu ra JSON sạch để tích hợp vào hệ thống tự động hóa.

### 4.2. Kết quả Chạy thử Thực tế (Với Giải pháp 2)

**Input đầu vào:**
> *"Mình mua chuột không dây mã DH-102 ngày 10/11/2026, hôm nay 17/11/2026 bấm không ăn, mình không làm rơi. Đổi giúp mình nhé!"*

**Output Trạm 1 (JSON Trích xuất):**
```json
{
  "order_id": "DH-102",
  "purchase_date": "2026-11-10",
  "issue": "Bấm không ăn",
  "user_fault": false,
  "missing_data": false
}
```

**Output Trạm 2 (JSON Lập luận CoT):**
```json
{
  "reasoning": [
    "Bước 1: Kiểm tra missing_data = false -> Đủ thông tin xử lý.",
    "Bước 2: Tính thời gian sử dụng: 17/11/2026 - 10/11/2026 = 7 ngày (thỏa mãn <= 15 ngày).",
    "Bước 3: Xác định lỗi: Bấm không ăn, không rơi vỡ -> Lỗi nhà sản xuất.",
    "Bước 4: Kết luận đơn hàng đủ điều kiện đổi mới 1-1."
  ],
  "decision": "APPROVED"
}
```

**Output Trạm 3 (Email Phản hồi Hoàn chỉnh):**
> **Tiêu đề:** [TechShop] Xác nhận đổi mới 1-đổi-1 cho đơn hàng #DH-102
>
> Chào bạn, TechShop thành thật xin lỗi về sự cố chuột không dây trên đơn hàng **#DH-102** (mua ngày 10/11/2026). Sau khi kiểm tra, sản phẩm của bạn hoàn toàn đủ điều kiện **Đổi mới 1-đổi-1 miễn phí**. Bạn vui lòng mang sản phẩm đến cửa hàng TechShop gần nhất để được đổi thiết bị mới ngay lập tức. Trân trọng!