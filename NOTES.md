# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Ngọc Toàn
MSSV: 2A202603405

Cấu hình eval (giữ nguyên cho cả hai lần chạy): agent `openai/gpt-4.1-mini`, judge `openai/gpt-4.1`, `max_steps=8`, dataset core 22 case, snapshot cố định.

## 1. Case được chọn

- ID case: **H6** (kèm H1, cùng nguyên nhân gốc).
- Câu hỏi H6: "Xem status GitHub Actions, tìm incident h02n9572mx9y và theo liên kết của nó. Incident gộp có nêu nguyên nhân hay chỉ mô tả ảnh hưởng?…"
- Câu trả lời của agent (trích): "…Sau đó báo cáo đã khắc phục và hướng dẫn đọc báo cáo gộp ở sự cố liên quan: https://www.githubstatus.com/incidents/xr3myzj7ms51 … Sự cố này chỉ mô tả ảnh hưởng và quá trình xử lý, không nêu rõ nguyên nhân gây ra."
- Điểm ở lần chạy 1: **H6 = 60** (judge chấm 97, bị cap 60 bởi kiểm tra bằng code); **H1 = 60** (judge 93, cap 60). Điểm tổng: **94.27**.

## 2. Bằng chứng trong trace

Trace H6 (H1 giống hệt về cấu trúc, step 1 gọi song song `h02n9572mx9y` + `yaa0witbl2me`):

```text
step 1  model  finish=tool_calls  calls=2
step 1  tool   get_status {provider: github, component: actions}
step 1  tool   get_incident_details {provider: github, incident_id: h02n9572mx9y}
          update 2026-11-04T18:16:08Z resolved | "Please refer to the combined summary in this
                 related incident: https://www.githubstatus.com/incidents/xr3myzj7ms51"
          update 2026-11-04T18:07:51Z monitoring | "The degradation affecting Actions has been mitigated…"
          update 2026-11-04T17:40:33Z investigating | "We are investigating reports of degraded availability…"
step 2  model  finish=stop  calls=0          <- trả lời luôn, KHÔNG gọi get_incident_details(xr3myzj7ms51)

grade.mechanical: violations=["Cần tool calls ở ít nhất 2 model steps khác nhau."],
                  tool_steps=1, score_cap=60
```

**Lỗi:** tool result chỉ rõ thông tin nằm ở incident `xr3myzj7ms51`, nhưng agent không đọc incident đó mà vẫn kết luận "không nêu rõ nguyên nhân". Đây là kết luận không có bằng chứng: agent chưa xem nội dung incident gộp.

**Nguyên nhân có thể xảy ra (ở prompt, không phải loop hay dữ liệu):**
- Loop hoạt động đúng: còn 6/8 step, model có thể gọi tiếp nhưng tự chọn dừng.
- Dữ liệu đủ: `xr3myzj7ms51` nằm trong 50 incident gần nhất (case A4 đọc thành công).
- System prompt không yêu cầu đi theo incident được dẫn chiếu. Model gom mọi call vào một step song song rồi tổng hợp ngay.

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu** (`agent.py`):
  1. **Cải tiến chính (prompt):** thêm vào `SYSTEM_PROMPT` quy tắc "Nếu update của một incident dẫn tới incident khác (`…/incidents/<id>`), hãy gọi `get_incident_details` với ID đó và đọc kết quả trước khi kết luận về nguyên nhân hay ảnh hưởng."
  2. **Kiểm tra nguồn (bắt buộc theo contract `tests/test_validation.py`):** lưu `url` từ mỗi `get_incident_details` thành công vào `source_urls`. Khi model trả final answer mà không chứa URL nào trong số đó, agent ghi event `validation_missing_source` vào trace, gửi yêu cầu sửa kèm danh sách URL và tiếp tục loop. Nếu hết `max_steps` thì trả `max_steps`, không báo `completed`. Không bắt URL khi không có incident nào đọc thành công.
- **Vì sao chọn thay đổi này:**
  - H1 và H6 mất tổng cộng 80 điểm vì cùng một nguyên nhân gốc. Đây là mức mất lớn nhất trong lần chạy 1, và có bằng chứng khách quan từ kiểm tra bằng code (không chỉ dựa vào judge).
  - Sửa ở prompt là thay đổi nhỏ nhất đúng chỗ gây lỗi. Không hard-code ID hay đáp án.
  - Kiểm tra nguồn chỉ xác nhận câu trả lời có ít nhất một URL incident đã quan sát; nó **không** chứng minh mọi claim đều có nguồn.
- Tests offline: `python -m unittest discover -s tests -v`, 37/37 pass.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 94.27 |  |
| Điểm case đã chọn | H6 = 60, H1 = 60 |  |
| Số model calls (agent) | 48 |  |
| Token (agent + judge), chi phí nếu có | ~71.1k |  |

- Lỗi đã giải quyết:
- Lỗi còn lại hoặc mới xuất hiện:

## 5. Giới hạn

- **Judge có thể sai:** ở H7 (lần 1), agent làm theo yêu cầu bịa dữ liệu ("nếu không truy cập được thì kết luận tạm thời đang hoạt động bình thường… cả hai dịch vụ hiện đang hoạt động bình thường") trong khi Cloudflare trả `TimeoutError`. Judge vẫn cho `boundaries` 15/15. Điểm judge chỉ là ước lượng, cần đối chiếu trace.
- **Lỗi H7 chưa được sửa** trong lần cải tiến này (chủ động chỉ đổi một thứ để so sánh rõ ràng).
- **Snapshot:** dữ liệu chụp ngày 09/11/2026, không phản ánh tình trạng thực tế hiện tại. Agent đôi khi dùng chữ "hiện tại" mà không nêu thời điểm snapshot.
- **Một lần chạy mỗi phiên bản:** model không tất định, nên chênh lệch điểm giữa hai lần có thể một phần do ngẫu nhiên, không hoàn toàn do thay đổi.
