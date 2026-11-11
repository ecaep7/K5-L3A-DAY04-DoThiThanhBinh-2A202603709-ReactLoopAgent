# Demo CP7 · Status Investigation Agent

Họ và tên: Đỗ Thị Thanh Bình · MSSV: 2A202603709 · Lớp: K5-L3A

Agent tự viết bằng Python, không dùng framework: gọi LLM, thực thi tool, trả kết quả kèm đúng `tool_call_id`, lặp đến khi có câu trả lời hoặc hết `max_steps`. Cải tiến chính là bước kiểm tra URL nguồn trước khi chấp nhận câu trả lời. Kết quả: số case thiếu URL giảm từ 11/16 xuống 0/15, điểm eval từ 92.32 lên 93.77.

## 1. Cấu hình

| Mục | Giá trị |
|---|---|
| Provider / model (agent và judge) | `anthropic` / `claude-haiku-5-5` |
| `max_steps` | 8 |
| Dataset | core, 22 case, snapshot 2026-11-09 03:50 UTC |
| `max_tokens` trong `providers.py` | 4096 (gốc 1024, xem mục 4) |

Hai lần eval dùng cùng cấu hình trên.

## 2. Agent loop (`agent.py`)

1. Tạo `messages` gồm system prompt và câu hỏi.
2. Gọi model. Lưu nguyên assistant message vào `messages`, vì trong đó có danh sách tool call mà các tool result phải khớp ID.
3. Với **mọi** tool call: parse arguments, chạy `tools.execute`, ghi trace `event: tool`, gửi kết quả về với đúng `tool_call_id`.
4. Không còn tool call và `finish == 'stop'`: kiểm tra nguồn (mục 3), đạt thì trả `completed`.
5. Xử lý lỗi:

| Tình huống | Cách xử lý | Event |
|---|---|---|
| Provider lỗi (mạng, key, hết credit) | Dừng, trả `error` | `provider_error` |
| Model bị cắt (`finish == 'length'`) | Dừng, trả `error` | `abnormal_finish` |
| JSON arguments hỏng, tool lỗi hoặc timeout | Không dừng, trả lỗi cho model làm observation | `tool` với `ok: false` |
| Hết `max_steps` | Trả `max_steps` | `max_steps` |

## 3. Trace nhiều bước: A8 sau cải tiến

Câu hỏi: so sánh incident GitHub `t89g2q1800fr` với Cloudflare `yaa0witbl2me` (thời điểm, trạng thái, phạm vi, **các nguồn**), có đủ căn cứ nói cùng nguyên nhân không.

```text
step 1  model finish=tool_calls, 2 calls
step 1  get_incident_details github t89g2q1800fr      -> url https://stspg.io/lv4q3xm9r0rk
step 1  get_incident_details cloudflare yaa0witbl2me  -> url https://www.cloudflarestatus.com/incidents/yaa0witbl2me
step 2  model finish=stop
step 2  validation_missing_source  missing_urls = [cả 2 URL trên]
step 3  model finish=stop  -> câu trả lời có đủ 2 URL, status completed
```

Ở lần 1, câu trả lời A8 có bảng so sánh 9 hàng nhưng không có hàng nguồn, dù câu hỏi yêu cầu. Điểm 83, evidence 13/20. Lần 2: 90 điểm, evidence 19/20.

Cách kiểm tra: mỗi lần `get_incident_details` trả `ok: true`, `url` được lưu vào `seen_urls`. Khi model trả lời, URL nào chưa có trong câu trả lời thì ghi event `validation_missing_source` và yêu cầu model viết lại trong giới hạn `max_steps`. Tool lỗi không ghi URL, nên câu hỏi về incident không tồn tại (H3) không bị bắt phải có link. `SYSTEM_PROMPT` giữ nguyên để chỉ có một thay đổi.

## 4. Một lỗi của agent: H1 bị cắt output

Trước khi tăng `max_tokens`, chạy thử case H1 (đọc `h02n9572mx9y`, đi theo liên kết sang `xr3myzj7ms51`, đối chiếu Cloudflare `yaa0witbl2me`):

```text
step 1  model finish=tool_calls, 2 calls (h02n9572mx9y, yaa0witbl2me)
step 2  model finish=tool_calls, 1 call  (xr3myzj7ms51, đi theo liên kết trong update)
step 3  model finish=length   output_tokens 1024 (thinking 416)
step 3  abnormal_finish
status: error | Model dừng bất thường: finish=length
```

Agent điều tra đúng chuỗi incident nhưng câu trả lời cuối bị cắt ở trần 1024 token của adapter Anthropic. Code báo `error`, không báo `completed`. Case thành `unscored` thì điểm tổng sẽ là `null`, nên tôi tăng `max_tokens` lên 4096 cho cả agent và judge, áp dụng giống nhau ở hai lần eval. File trace: `demo_h1_length.json`.

## 5. So sánh hai lần eval

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 92.32 | 93.77 |
| Điểm A8 | 83 (evidence 13/20) | 90 (evidence 19/20) |
| Số model calls của agent | 56 | 68 |
| Token agent (input / output) | 88.104 / 38.833 | 130.874 / 53.611 |
| Token judge (input / output) | 78.135 / 25.371 | 83.581 / 27.016 |
| Case thiếu URL nguồn | 11/16 | 0/15 |
| Case kích hoạt `validation_missing_source` | (chưa có) | 9 |

Case kích hoạt validation ở lần 2: E2, E3, A6, A7, A8, H1, H2, H5, H6. Cả 9 case đều sửa xong sau một lần viết lại. Đổi lại, agent gọi model thêm 12 lần và token input tăng khoảng 49%.

## 6. Giới hạn

- Điểm tăng 1,45 nhỏ hơn mức dao động của judge: E6 tăng 8 và H3 tăng 7 dù không kích hoạt validation, A1 giảm 5 và A5 giảm 4. Bằng chứng đáng tin hơn là 11/16 xuống 0/15.
- Judge cùng model với agent nên có thể dễ dãi với lối viết của chính nó.
- Bước kiểm tra chỉ biết URL có mặt, không biết nội dung đúng. A8 vẫn sai câu "cách nhau khoảng 3 giờ" (factuality 37 xuống 34). Bước này cũng chỉ theo dõi URL từ `get_incident_details`, không theo dõi `source` của `get_status` (H8).
- Dữ liệu là snapshot 50 incident gần nhất; mỗi incident chỉ giữ 3 update, mỗi update 900 ký tự (`xr3myzj7ms51` có 9 update). Chưa thử `--mode live`.
- Mỗi phiên bản chỉ chạy một lần. Lần chạy riêng A8 trước đó model tự nhớ URL (94 điểm, không có validation); lần chạy cả bộ thì quên và phải viết lại. Cần chạy nhiều lần để kết luận về độ ổn định.

## 7. Lệnh tái hiện

```powershell
python -m unittest discover -s tests -v
python evals/run_evals.py --output eval_before.json   # trước khi thêm kiểm tra nguồn
python evals/run_evals.py --output eval_after.json    # sau khi thêm kiểm tra nguồn
```
