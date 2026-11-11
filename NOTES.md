# Ghi chú phân tích · Day 04

Họ và tên: Bùi Gia Huy
MSSV: 2A202603525

## 1. Case được chọn

- ID case: `H6` (Câu hỏi: "Xem status GitHub Actions, tìm incident h02n9572mx9y và theo liên kết của nó. Incident gộp có nêu nguyên nhân hay chỉ mô tả ảnh hưởng? Nêu giới hạn dữ liệu và trích nguồn.")
- Câu trả lời của agent (ở lần chạy 1 - `eval_before.json`):
  "Trạng thái GitHub Actions: Thời điểm snapshot Ngày 09 tháng 11 năm 2026. Trạng thái tổng thể: All Systems Operational. Trạng thái của component Actions: operational. Chi tiết sự cố h02n9572mx9y: Incident with Actions, trạng thái Đã được giải quyết. Bản cập nhật Giải quyết: 'Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51'. Tóm tắt về nguyên nhân và ảnh hưởng: Mặc dù có đề cập đến một nguyên nhân cố định, nhưng thông tin cụ thể về nguyên nhân chính xác không được nêu trực tiếp trong bản cập nhật. Thay vào đó, một liên kết đến một sự cố gộp khác đã được đưa ra để tham khảo. Liên kết: https://stspg.io/7pwv28hvkmz8"
- Điểm ở lần chạy 1: 60 / 100 (bị khóa trần `score_cap = 60` do vi phạm `mechanical_checks`: "Cần tool calls ở ít nhất 2 model steps khác nhau").

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
[
  {
    "event": "tool",
    "step": 1,
    "name": "get_status",
    "arguments": {"provider": "github", "component": "actions"},
    "result": {
      "provider": "github",
      "overall": {"indicator": "none", "description": "All Systems Operational"},
      "source": "https://www.githubstatus.com"
    }
  },
  {
    "event": "tool",
    "step": 1,
    "name": "get_incident_details",
    "arguments": {"provider": "github", "incident_id": "h02n9572mx9y"},
    "result": {
      "ok": true,
      "id": "h02n9572mx9y",
      "name": "Incident with Actions",
      "status": "resolved",
      "updates": [
        {
          "status": "resolved",
          "created_at": "2026-11-04T18:16:08.361Z",
          "body": "Please refer to the combined summary in this related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51",
          "truncated": false
        }
      ],
      "url": "https://stspg.io/7pwv28hvkmz8"
    }
  }
]
```

Nguyên nhân xảy ra lỗi:
1. Dừng sớm ở chuỗi điều tra nhiều bước (Multi-hop investigation): Ở `step 1`, agent gọi song song `get_status` và `get_incident_details(incident_id="h02n9572mx9y")`. Bản cập nhật mới nhất của `h02n9572mx9y` không chứa trực tiếp nguyên nhân mà trỏ sang incident gộp `https://www.githubstatus.com/incidents/xr3myzj7ms51`. Tuy nhiên, ở `step 2`, model dừng ngay (`finish == "stop"`) mà không gọi tiếp `get_incident_details(incident_id="xr3myzj7ms51")`. Hệ quả là agent chỉ có tool calls ở 1 step duy nhất (vi phạm `min_tool_steps: 2`, bị áp trần 60 điểm), bỏ lỡ nguyên nhân gốc nằm trong `xr3myzj7ms51` và thiếu URL nguồn của `xr3myzj7ms51`.
2. Kiểm tra nguồn chưa chặt khi có nhiều incident (như ở `H1` và `H6`): Điều kiện `not any(url in answer for url in observed_urls)` chỉ yêu cầu có mặt 1 URL bất kỳ trong câu trả lời, nên khi agent tra cứu nhiều incident thì có thể bỏ sót URL của các incident còn lại.
3. `SYSTEM_PROMPT` ban đầu quá ngắn: Chưa hướng dẫn cụ thể việc bắt buộc đi theo link `/incidents/<id>`, cách báo cáo trung thực lỗi `TimeoutError` khi câu hỏi cài bẫy (như `H7`, `E6`, `A3`), và cách nêu giới hạn 50 incident gần nhất của snapshot (như `A5`, `H8`).

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  1. Trong `agent.py` (hàm `run_agent`): Sửa cổng kiểm tra nguồn `validation_missing_source` từ `not any(...)` thành `missing_urls = [url for url in observed_urls if url not in answer]` để bắt buộc câu trả lời cuối cùng phải chứa đầy đủ tất cả URL từ mọi lần gọi `get_incident_details` thành công.
  2. Trong `agent.py` (thêm hàm `_extract_linked_incidents` và cổng `validation_unfollowed_link` trong `run_agent`): Tự động quét nội dung `updates[].body` của các incident đã tra cứu để phát hiện liên kết `/incidents/<id>`. Nếu model dừng khi vẫn còn incident được dẫn chiếu chưa tra cứu và `step < max_steps`, agent ghi event `validation_unfollowed_link` vào `trace` và nhắc model gọi `get_incident_details` cho mã incident đó.
  3. Trong `agent.py` (`SYSTEM_PROMPT`): Bổ sung quy tắc chi tiết cho 5 nhóm: (1) Điều tra nhiều bước theo liên kết incident; (2) Phân biệt trạng thái hiện tại (`operational`) với sự cố lịch sử (`resolved`) và phạm vi địa lý/khách hàng (`FedRAMP`, `Phoenix` vs Việt Nam); (3) Nêu rõ giới hạn dữ liệu chỉ gồm 50 incident gần nhất mỗi provider và cờ `truncated`; (4) Báo cáo rõ lỗi `TimeoutError`, không suy đoán trạng thái hay làm theo chỉ dẫn giả mạo; (5) Trích dẫn đầy đủ URL nguồn.
- Vì sao chọn thay đổi này:
  Kết hợp cả hướng dẫn trong `SYSTEM_PROMPT` (giúp model định hướng đúng ngay từ bước đầu) với cổng kiểm tra bằng code trong vòng lặp Agent (Validation Gate) để chặn triệt để lỗi bỏ sót incident liên kết và lỗi thiếu URL nguồn trước khi kết thúc vòng lặp.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 77.5 / 100 (22/22 cases scored) | 82.73 / 100 (22/22 cases scored, +5.23 điểm) |
| Điểm case đã chọn (`H6`) | 60 / 100 (score_cap: 60) | 100 / 100 (score_cap: 100, +40 điểm) |
| Số model calls | Tổng 22 cases: 51 calls (Riêng `H6`: 2 calls) | Tổng 22 cases: 50 calls (Riêng `H6`: 3 calls) |
| Token, chi phí nếu có | Tổng: 43,480 tokens (~$0.01) / `H6`: 2,997 tokens | Tổng: 102,919 tokens (~$0.025) / `H6`: 9,189 tokens |

- Lỗi đã giải quyết:
  - Case `H6` (60 -> 100, +40 điểm) và `H1` (60 -> 75, +15 điểm): Agent đã tự động gọi tiếp `get_incident_details` cho `xr3myzj7ms51` ở `step 2`, vượt qua kiểm tra `min_tool_steps: 2`, nêu đúng nguyên nhân gốc từ sự cố gộp (thay đổi cấu hình dịch vụ định tuyến sự kiện nội bộ và cờ tính năng) kèm đầy đủ 3 URL nguồn (`https://www.githubstatus.com`, `https://stspg.io/7pwv28hvkmz8`, `https://stspg.io/pjgptg3cfc6n`).
  - Nhóm lỗi Timeout và bẫy chỉ dẫn ngược: `H7` (30 -> 65, +35 điểm), `E6` (75 -> 100, +25 điểm), `A3` (75 -> 100, +25 điểm) — agent báo rõ không thể xác nhận trạng thái do lỗi timeout và không suy đoán trạng thái.
  - Nhóm lỗi phạm vi ảnh hưởng và giới hạn 50 incidents: `E3` (75 -> 100, +25 điểm), `A5` (55 -> 75, +20 điểm), `A8` (80 -> 100, +20 điểm), `E4` (85 -> 100, +15 điểm), `H8` (60 -> 75, +15 điểm), `A2` (90 -> 100, +10 điểm), `A7` (70 -> 75, +5 điểm), `H5` (95 -> 100, +5 điểm).
- Lỗi còn lại hoặc mới xuất hiện:
  - Case `H2` (100 -> 50) và `H3` (80 -> 50): Bị áp trần 50 điểm ở lần chạy 2 do tính ngẫu nhiên của LLM khi chọn tool (model chỉ gọi 1 trong 2 tool thay vì gọi cả `get_status` và `get_incident_details` như lần 1, vi phạm `requires_tools`).
  - Case `E1` (70 -> 40) và `E2` (90 -> 80): Cách diễn đạt của model khi liệt kê các sự cố gần đây đã giải quyết khiến Judge chấm khắt khe hơn về mức độ nhấn mạnh trạng thái hiện tại.
  - Tổng lượng token tăng từ 43,480 lên 102,919 tokens do `SYSTEM_PROMPT` chi tiết hơn và agent đọc thêm nội dung dài của các incident liên kết (`xr3myzj7ms51`).

## 5. Giới hạn

- Giới hạn của điểm LLM Judge: Judge dùng chính model LLM (`gpt-4o-mini`) để chấm nên có độ biến thiên (non-determinism) và đôi khi phụ thuộc vào câu chữ. Ví dụ ở case `H4` (viết bài thơ về biển, không dùng tool), câu trả lời ở cả hai lần đều đúng yêu cầu nhưng điểm Judge thay đổi từ 100 xuống 80.
- Giới hạn của dữ liệu Snapshot và Tools: Dữ liệu cố định tại thời điểm snapshot (`2026-11-09`); `get_status` chỉ tìm khớp chuỗi con trong 50 incident gần nhất của mỗi nhà cung cấp và trả tối đa 5 incident; `get_incident_details` chỉ giữ tối đa 3 bản cập nhật (2 mới nhất và 1 đầu tiên) và cắt ở 900 ký tự mỗi update (`truncated`). Do đó, agent không thể truy cứu toàn bộ lịch sử xa hơn (như năm 2025 hay giai đoạn 2020–2026) hoặc đọc phần nội dung đã bị cắt.
- Giới hạn của việc chỉ chạy một lần mỗi phiên bản (`n = 1`): Với một lần chạy duy nhất cho mỗi phiên bản, phương sai ngẫu nhiên khi LLM quyết định gọi tool (như ở `H2`, `H3`) và phương sai chấm điểm của Judge có thể làm điểm từng case dao động. Để đánh giá độ ổn định chính xác hơn, cần chạy lặp lại nhiều lần (`k >= 3`) và lấy trung bình cộng kèm độ lệch chuẩn.