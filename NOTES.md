# Ghi chú phân tích · Day 04

Họ và tên: Đỗ Thị Thanh Bình
MSSV: 2A202603709

## 1. Case được chọn

- ID case: A8
- Câu trả lời của agent: Agent kết luận không đủ căn cứ để nói hai incident cùng nguyên nhân. Agent so sánh hai incident bằng một bảng gồm thời điểm bắt đầu và kết thúc, trạng thái (cả hai đều resolved), component (Actions và Workers Builds), triệu chứng, phạm vi và nguyên nhân (cả hai bên đều chưa công bố). Agent cũng nêu giới hạn bằng chứng. Tuy nhiên, câu trả lời không có URL nguồn nào, dù câu hỏi yêu cầu nêu “các nguồn”. Xem đầy đủ ở eval_before.json, case A8, trường answer
- Điểm ở lần chạy 1: 83

## 2. Bằng chứng trong trace

```
step 1  get_incident_details {"provider": "github", "incident_id": "t89g2q1800fr"}
        -> result.url = https://stspg.io/lv4q3xm9r0rk
step 1  get_incident_details {"provider": "cloudflare", "incident_id": "yaa0witbl2me"}
        -> result.url = https://www.cloudflarestatus.com/incidents/yaa0witbl2me
step 2  model finish=stop, num_calls=0
answer: không chứa URL nào trong hai URL trên
grade:  83 — evidence 13/20; reason: "không nêu URL của hai incident dù expected yêu cầu"

```
SYSTEM_PROMPT chỉ ghi chung chung “Dẫn nguồn từ tool”, không nói rõ phải chép URL.
Trong vòng lặp không có bước nào kiểm tra lại câu trả lời trước khi chấp nhận. Model trả lời xong là code coi như hoàn thành luôn.
## 3. Thay đổi đã thực hiện

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: mình thêm một bước kiểm tra nguồn vào hàm `run_agent` trong `agent.py`, các chỗ có comment đánh số (5). Mỗi lần `get_incident_details` trả về `ok: true`, code lưu `url` của incident vào danh sách `seen_urls`. Lúc model trả lời xong (`finish == 'stop'`, không còn tool call), code dò xem câu trả lời có chứa đủ các URL trong danh sách không. Nếu thiếu, code ghi event `validation_missing_source` vào trace kèm các URL còn thiếu, gửi lại cho model một tin nhắn liệt kê đúng những URL đó và yêu cầu viết lại, rồi gọi model thêm một lượt. Lượt viết lại vẫn tính vào `max_steps`. Hết lượt mà vẫn thiếu thì trả `max_steps`, không trả `completed`. Lần gọi tool bị lỗi không được ghi vào danh sách, nên câu hỏi về incident không tồn tại (kiểu H3) không bị bắt phải có link.

- Vì sao chọn thay đổi này: Ở lần chạy 1, trong 16 case agent đọc được chi tiết incident thì 11 case không có URL nào trong câu trả lời. A8 bị trừ nặng nhất ở mục evidence (13/20). Điều làm mình chọn sửa bằng code thay vì sửa prompt: `SYSTEM_PROMPT` đã có câu "Dẫn nguồn từ tool", và câu hỏi A8 còn ghi rõ "các nguồn", vậy mà model vẫn dựng một bảng 9 hàng không có hàng nào cho nguồn. Đã nhắc hai lần mà vẫn quên thì nhắc lần thứ ba trong prompt chưa chắc ăn thua. Cần một bước kiểm tra trước khi chấp nhận câu trả lời, giống quy trình duyệt hồ sơ phải soát chứng từ đính kèm trước khi ký. mình giữ nguyên `SYSTEM_PROMPT` để chỉ có một thay đổi, như vậy điểm lần 2 khác đi thì biết là do bước kiểm tra này chứ không lẫn với prompt.

- Đánh đổi: Mỗi lần agent bị trả về viết lại là tốn thêm một lượt gọi model, phần input còn mang theo toàn bộ lịch sử trước đó, nên số model calls và token ở lần 2 có thể tăng. Bước kiểm tra cũng chỉ so chuỗi URL có nằm trong câu trả lời hay không. Nó không chứng minh từng ý trong câu trả lời đúng với nguồn, và model hoàn toàn có thể dán URL vào cuối bài cho đủ mà nội dung vẫn sai.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 92.32 | 93.77 |
| Điểm case đã chọn (A8) | 83 (evidence 13/20) | 90 (evidence 19/20) |
| Số model calls | 56 | 68 (+21%) |
| Token, chi phí nếu có | Agent: 88.104 input / 38.833 output; Judge: 78.135 / 25.371 | Agent: 130.874 input / 53.611 output; Judge: 83.581 / 27.016 |
| Case thiếu URL nguồn | 11/16 | 0/15 |
| Case kích hoạt `validation_missing_source` | (chưa có) | 9 |

Provider `anthropic` không trả trường `cost` trong `usage`, nên chỉ so sánh được bằng token.

- Lỗi đã giải quyết: Lỗi thiếu URL nguồn. Đổi lại, agent gọi model thêm 12 lượt và token input của agent tăng khoảng 49%, vì mỗi lần viết lại phải gửi kèm toàn bộ lịch sử hội thoại.
- Lỗi còn lại hoặc mới xuất hiện: 
    - A8 tăng điểm nhờ phần evidence, nhưng factuality giảm từ 37 xuống 34. Câu “cách khoảng 3 giờ” vẫn sai, và lần 1 giám khảo cũng đã chỉ ra lỗi này. Bước kiểm tra chỉ biết URL có mặt trong câu trả lời, nó không biết nội dung có đúng hay không.
    - H8 bị trừ vì không dẫn link trang status chung (githubstatus.com). Bước kiểm tra chỉ theo dõi URL lấy từ get_incident_details, không theo dõi source của get_status.

## 5. Giới hạn

- **Điểm judge:** Judge và agent dùng cùng một model (`claude-haiku-5-5`), tức là model tự chấm bài của chính nó, nên có thể dễ dãi với lối viết của mình. Điểm judge cũng dao động ngay cả khi agent không đổi gì đáng kể. Ở lần 2, có những case không hề kích hoạt `validation_missing_source` mà điểm vẫn lệch vài điểm: E6 tăng 8, H3 tăng 7, A1 giảm 5, A5 giảm 4. Điểm tổng tăng 1,45 (92.32 lên 93.77), nhỏ hơn mức dao động đó, nên mình không coi đây là bằng chứng cải tiến có tác dụng. Bằng chứng mình tin hơn là phần kiểm tra bằng code: số case thiếu URL giảm từ 11/16 xuống 0/15. Ngay cả con số này cũng chỉ cho biết URL có mặt trong câu trả lời. Ở A8, evidence tăng từ 13 lên 19, nhưng factuality lại giảm từ 37 xuống 34, vì câu "cách nhau khoảng 3 giờ" vẫn sai như lần 1.

- **Snapshot:** Toàn bộ câu trả lời dựa trên dữ liệu chụp lúc 2026-11-09 03:50 UTC, chỉ gồm 50 incident gần nhất của mỗi hãng. Mỗi incident chỉ được giữ 3 update, mỗi update cắt ở 900 ký tự. Ví dụ xr3myzj7ms51 có 9 update nhưng agent chỉ thấy 3, và phần nguyên nhân bị cắt giữa câu. mình chưa chạy `--mode live`, nên chưa biết agent xử lý thế nào khi gặp rate limit, timeout thật hay dữ liệu thay đổi giữa hai lần gọi tool.

- **Mỗi phiên bản chỉ chạy một lần:** Lần chạy thử riêng A8 trước khi chạy cả bộ cho thấy rõ điều này. Lúc đó model tự chép đủ URL ngay lần đầu, validation không kích hoạt, mà điểm vẫn lên 94. Đến lần chạy cả bộ, cũng case đó model lại quên URL và phải viết lại mới được 90. Cùng code, cùng câu hỏi, nhưng hai lần chạy cho hai đường đi khác nhau. Muốn kết luận chắc chắn thì cần chạy mỗi phiên bản vài lần rồi so sánh trung bình và độ lệch. Lab này chưa làm được việc đó vì giới hạn thời gian và chi phí.

- **Cấu hình:** Hai lần eval đều dùng provider `anthropic`. mình đã tăng `max_tokens` trong `providers.py` từ 1024 lên 4096, vì ở mức 1024 câu trả lời tiếng Việt cộng phần thinking bị cắt (trace H1: `output_tokens` 1024, `finish=length`, case thành `unscored`). Thay đổi này áp dụng cho cả agent lẫn judge và giữ nguyên ở cả hai lần chạy. Nhưng nó có nghĩa là kết quả của mình không so sánh trực tiếp được với bài của bạn khác dùng cấu hình gốc.