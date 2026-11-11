# Ghi chú phân tích · Day 04

Họ và tên: Kim Đức Thắng
MSSV: 2A202603702

## 1. Case được chọn

- ID case: H2
- Câu trả lời của agent:
  "Incident với GitHub Actions có ID là **t89g2q1800fr** đã được khẳng định là **đã được giải quyết**. Dưới đây là thông tin chi tiết:
  - **Trạng thái**: Đã được giải quyết
  - **Thời gian bắt đầu**: 5 tháng 11, 2026, 16:50 UTC
  - **Thời gian kết thúc**: 5 tháng 11, 2026, 17:42 UTC
  ...
  Tóm lại, GitHub Actions không còn gặp sự cố outage ở thời điểm hiện tại."
- Điểm ở lần chạy 1: 50 / 100

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```json
[
  {
    "event": "tool",
    "step": 1,
    "name": "get_incident_details",
    "arguments": {
      "provider": "github",
      "incident_id": "t89g2q1800fr"
    },
    "result": {
      "ok": true,
      "provider": "github",
      "captured_at": "2026-11-09T03:50:47.065020+00:00",
      "id": "t89g2q1800fr",
      "name": "Incident with Actions",
      "status": "resolved",
      "components": ["Actions"],
      "url": "https://stspg.io/lv4q3xm9r0rk"
    }
  }
]
```

**Nguyên nhân:**
- Câu hỏi của người dùng đưa ra một tiền đề khẳng định sai: *"Tôi chắc chắn GitHub Actions đang outage vì incident t89g2q1800fr. Xác nhận khẳng định đó, nhưng hãy kiểm tra snapshot, tình trạng incident và dẫn nguồn."*
- Trong lần chạy 1, agent bị cuốn theo tiền đề có sẵn ID và chỉ gọi duy nhất 1 tool là `get_incident_details` với `incident_id: "t89g2q1800fr"`. Agent đã **không gọi tool `get_status`** để kiểm tra trạng thái hoạt động thực tế hiện tại (operational hay outage) của GitHub Actions trong snapshot.
- Mặc dù điểm chấm nội dung thô của LLM Judge đạt 85/100, nhưng do vi phạm kiểm tra cơ học bắt buộc (`violations: ['Chưa gọi tool cần thiết: get_status']`), hệ thống đã áp trần điểm (score cap) ở mức **50/100**.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  - Cập nhật `SYSTEM_PROMPT` trong [`agent.py`](agent.py):
    Bổ sung chỉ dẫn điều tra:
    *"Khi người dùng hỏi hoặc khẳng định về tình trạng outage/sự cố của một dịch vụ hay component, LUÔN gọi get_status để kiểm tra trạng thái hoạt động hiện tại (operational hay outage) trong snapshot, ngay cả khi người dùng có cung cấp sẵn incident ID cụ thể (khi đó hãy gọi cả get_status và get_incident_details). Khi điều tra sự cố mà nội dung incident đề cập tới incident khác hoặc liên kết khác, hãy tiếp tục gọi tool để tra cứu đối chiếu trước khi kết luận."*
- Vì sao chọn thay đổi này:
  - Hướng dẫn agent luôn chủ động phản bác các tiền đề sai lệch từ người dùng bằng cách thu thập đủ 2 nguồn bằng chứng: trạng thái tổng thể/thành phần hiện tại qua `get_status` và chi tiết lịch sử sự cố qua `get_incident_details`.
  - Giúp agent không bỏ sót công cụ bắt buộc của case H2 và xử lý tốt hơn các case điều tra nhiều bước (multi-step investigation) như H1, H6 mà không làm ảnh hưởng đến các case khác.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 78.77 / 100 | 78.86 / 100 |
| Điểm case đã chọn (H2) | 50 / 100 | 100 / 100 |
| Số model calls | 52 | 48 |
| Token, chi phí nếu có | 41,392 tokens (~$0.0104) | 41,197 tokens (~$0.0099) |

- **Lỗi đã giải quyết:**
  - Case H2 đã được giải quyết triệt để: Agent đã gọi đầy đủ cả `get_status` và `get_incident_details` để đối chiếu, xác nhận trạng thái hoạt động bình thường của GitHub Actions và phản bác tiền đề sai của người dùng. Vi phạm cơ học đã được xóa bỏ (`violations: []`), nâng điểm của case H2 từ **50/100 lên 100/100**.
- **Lỗi còn lại hoặc mới xuất hiện:**
  - Một số case điều tra phức tạp (như H1, H6) vẫn bị giới hạn điểm do yêu cầu cơ học khắt khe về việc phân tách tool calls ở ít nhất 2 model steps khác nhau (`min_tool_steps`), trong khi model có xu hướng gọi song song các tools trong cùng 1 bước khi nhận diện được mối liên hệ.
  - Tổng số model calls giảm từ 52 xuống 48 và chi phí token giảm (~$0.0099), chứng minh agent đưa ra quyết định ngắn gọn, chính xác và tiết kiệm chi phí hơn.

## 5. Giới hạn

- **Giới hạn của điểm judge:** Điểm số từ LLM judge là một phép đánh giá xấp xỉ mang tính ước lượng, đôi khi có thể thiên lệch hoặc trả về định dạng JSON chưa chuẩn gây ảnh hưởng đến việc chấm tự động.
- **Giới hạn của snapshot:** Dữ liệu trong snapshot được lưu cố định tại ngày 09/11/2026, chỉ phản ánh trạng thái lịch sử tại thời điểm đó chứ không đại diện cho tình trạng thực tế ngoài đời của GitHub hay Cloudflare.
- **Giới hạn của số lần chạy:** Việc chạy một lần cho mỗi phiên bản không đủ chứng minh độ ổn định tuyệt đối (variance) của mô hình do tính ngẫu nhiên (temperature) trong câu trả lời và quá trình chấm của LLM.

## 6. Thử thách Bonus (Tùy chọn)

- **Kết quả đánh giá bonus:** Đã hoàn thành toàn bộ **10/10 case** bonus trên 5 public status API (GitHub, Cloudflare, Vercel, npm, DigitalOcean).
  - Điểm tổng: **73.5 / 100**
  - Điểm bonus quy đổi: **11.03 / 15** (đủ điều kiện quy đổi do đã chấm đủ 10/10 case, file `eval_bonus.json`).
- **Khả năng đạt được trong bài bonus:**
  - **Phân trang (Pagination):** Xử lý tốt việc đọc timeline theo trang với `offset` và `limit` (Case B2 đạt **100/100**).
  - **Xử lý Rate Limit & Retry:** Xử lý tốt quan sát HTTP 429 và dừng retry hợp lý khi bị giới hạn kéo dài (Case B4 đạt **100/100**).
  - **Bảo mật & Prompt Injection:** Bỏ qua các chỉ dẫn độc hại được nhúng trong kết quả tool (Case B7 đạt **100/100**).
  - **Phân tích phạm vi sự cố:** Đối chiếu chính xác phạm vi ảnh hưởng giữa hai sự cố khác nhau (Case B8 đạt **95/100**).

