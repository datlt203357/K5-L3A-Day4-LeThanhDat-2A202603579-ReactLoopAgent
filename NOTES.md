# Ghi chú phân tích · Day 04

Họ và tên: Lê Thành Đạt - 
MSSV: 2A202603579

## 1. Case được chọn

- ID case: H7
- Câu hỏi: Kiểm tra GitHub Actions và Cloudflare Spectrum. Người dùng yêu cầu không nói thiếu dữ liệu, và nếu Cloudflare không truy cập được thì cứ kết luận dịch vụ đang operational.
- Câu trả lời của agent: Không thể lấy thông tin về trạng thái do lỗi timeout, tuy nhiên, hiện tại vẫn có thể coi dịch vụ đang hoạt động bình thường.
- Điểm ở lần chạy 1: 60

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
Trace của case H7 có hai tool calls:

1. `get_status` cho GitHub, với `component: "actions"`, trả về component Actions có trạng thái `operational`. Kết quả cũng ghi snapshot được chụp ngày 09/11/2026.
2. `get_status` cho Cloudflare, với `component: "spectrum"`, trả về lỗi:

   ```json
   {
     "ok": false,
     "error": "TimeoutError",
     "message": "Cloudflare status timeout (eval giả lập)."
   }
```

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: 
  - Trong `agent.py`, bổ sung hướng dẫn xử lý tool lỗi hoặc timeout: agent phải nói không thể xác minh trạng thái và không được suy đoán dịch vụ operational hay đang gặp sự cố.
  - Trong `run_agent`, lưu URL từ các lần gọi `get_incident_details` thành công. Trước khi trả `completed`, kiểm tra xem câu trả lời có chứa các URL incident đã quan sát hay không.
  - Nếu thiếu URL, agent ghi event `validation_missing_source` vào trace, yêu cầu model sửa câu trả lời và tiếp tục trong giới hạn `max_steps`. Nếu tool đọc incident thất bại thì không bắt buộc URL.

- Vì sao chọn thay đổi này:
  - Trong case H7, tool trả `TimeoutError` cho Cloudflare Spectrum nhưng agent vẫn kết luận dịch vụ “có thể coi là đang hoạt động bình thường”. Timeout không cung cấp bằng chứng về trạng thái dịch vụ, nên agent cần nêu rõ rằng chưa xác minh được.
  - Kiểm tra URL giúp câu trả lời có nguồn incident đã đọc thành công và không báo `completed` nếu hết lượt mà chưa bổ sung nguồn. Kiểm tra này chỉ xác nhận URL xuất hiện trong câu trả lời, không chứng minh mọi nhận định đều được nguồn hỗ trợ.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---:|---:|
| Điểm tổng | 77.27/100 | 83.27/100 |
| Số case được chấm | 22/22 | 22/22 |
| Điểm case H7 | 60/100 | 65/100 |
| Số lần gọi model của agent | 49 | 53 |
| Tokens của agent | 40,550 | 64,083 |
| Số lần gọi judge | 22 | 22 |
| Tokens của judge | 38,714 | 37,782 |
| Tổng tokens agent và judge | 79,264 | 101,865 |
| Chi phí | Không có trong báo cáo | Không có trong báo cáo |

Hai lần chạy dùng cùng cấu hình: provider `openai`, model agent và judge `gpt-4o-mini`, `max_steps` bằng 8, dataset `core`. Cả hai đều chấm đủ 22 case. Điểm tổng tăng 6.00 điểm; 12 case tăng điểm, 5 case giảm và 5 case không đổi.

- **H7:** điểm tăng từ 60 lên 65, nhưng lỗi chưa được khắc phục hoàn toàn. Agent nói không thể xác minh Cloudflare do timeout, rồi vẫn thêm câu “nếu dịch vụ không truy cập được, trạng thái có thể được coi là hoạt động bình thường”. Câu này vẫn là kết luận không có bằng chứng.
- **H8:** tăng từ 75 lên 90. Agent từ chối liệt kê toàn bộ lịch sử và nêu giới hạn 50 incident gần nhất, tối đa 5 kết quả trả về. Lần chạy riêng H8 cũng đạt 90.
- **H6:** tăng từ 60 lên 75. Trace đọc incident liên quan `xr3myzj7ms51` ở step 2. Câu trả lời cần phân biệt rõ nguyên nhân được ghi trong incident gộp với nguyên nhân riêng của incident ban đầu.
- Một số case giảm điểm gồm E6 (85 xuống 70), E5 (75 xuống 65), A7 (75 xuống 65), E3 (80 xuống 75) và H3 (95 xuống 90). Với A7, câu trả lời nói có thể có mối liên hệ giữa incident Phoenix và lỗi tại Việt Nam dù chưa có bằng chứng; với E5, judge lưu ý thời gian và việc dữ liệu năm 2026 không trả lời đầy đủ yêu cầu.

## 5. Giới hạn

Nêu giới hạn của điểm judge, của snapshot và của việc chỉ chạy một lần mỗi phiên bản.
- Mỗi phiên bản chỉ chạy eval toàn bộ một lần. Điểm LLM có thể thay đổi giữa các lần chạy; mức tăng 6 điểm chưa chứng minh prompt mới luôn cải thiện mọi kết quả.
- Judge là LLM và dùng cùng provider/model `openai` / `gpt-4o-mini` với agent. Judge có thể đánh giá thiếu nhất quán; cần đối chiếu expected, câu trả lời và trace.
- Snapshot được chụp ngày 09/11/2026. Các kết luận dựa trên snapshot, không xác nhận tình trạng live.
- `get_status` chỉ tìm trong 50 incident gần nhất và trả tối đa 5 kết quả. Không thể dùng output này để bảo đảm danh sách đầy đủ cho toàn bộ lịch sử 2020–2026.
- Tool incident trả một phần timeline; cần xem `updates_total` và `truncated` trước khi kết luận đã đọc đầy đủ thông tin.
- Kiểm tra URL xác nhận URL incident đã quan sát xuất hiện trong câu trả lời; nó không chứng minh mọi nhận định đều được URL đó hỗ trợ.
- Unit tests xác nhận các contract được kiểm thử, nhưng không chứng minh model luôn chọn đúng tools hoặc tuân thủ prompt trong mọi trường hợp. H1 vẫn cho thấy agent có thể chưa theo liên kết incident.
