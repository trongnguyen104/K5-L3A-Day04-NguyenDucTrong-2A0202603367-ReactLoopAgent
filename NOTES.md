# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Đức Trọng
MSSV: 2A202603367

## 1. Case được chọn

- ID case: **H6**
- Câu hỏi: "Xem status GitHub Actions, tìm incident `h02n9572mx9y` và theo liên kết của nó. Incident gộp có nêu nguyên nhân hay chỉ mô tả ảnh hưởng? Nêu giới hạn dữ liệu và trích nguồn."
- Câu trả lời của agent (lần chạy 1, `eval_before2.json`): agent gọi đúng `get_incident_details` cho `h02n9572mx9y` rồi `xr3myzj7ms51`, nhưng ở bước trả lời cuối lại trích URL dạng `https://www.githubstatus.com/incidents/<id>` thay vì URL nguồn thật sự có trong `updates` của tool result (`https://stspg.io/7pwv28hvkmz8`, `https://stspg.io/pjgptg3cfc6n`). Evidence Gate liên tục báo `validation_missing_source` và yêu cầu sửa, nhưng model lặp lại đúng lỗi cũ ở mọi bước.
- Điểm ở lần chạy 1: **0/100**, `status = max_steps` (hết 8 bước mà không hoàn tất).

## 2. Bằng chứng trong trace

```text
[step 1] tool=get_incident_details args={provider: github, incident_id: h02n9572mx9y}
[step 2] tool=get_incident_details args={provider: github, incident_id: xr3myzj7ms51}
[step 3] model finish=stop
[step 3] validation_missing_source: missing_urls=[
    'https://stspg.io/7pwv28hvkmz8', 'https://stspg.io/pjgptg3cfc6n']
[step 4] model finish=stop
[step 4] validation_missing_source: missing_urls=[... giống step 3 ...]
...
[step 8] model finish=stop
[step 8] validation_missing_source: missing_urls=[... giống step 3 ...]

Grade: {
  "score": 0, "status": "max_steps",
  "reason": "Agent hết số bước trước khi hoàn tất.",
  "mechanical": {
     "violations": ["Chưa gọi tool cần thiết: get_status"],
     "score_cap": 50
  }
}
```

**Nguyên nhân có thể xảy ra:**
1. Agent không được hướng dẫn rõ phải tiếp tục đọc incident liên kết (`xr3myzj7ms51`) khi thấy update của `h02n9572mx9y` nhắc tới nó dưới dạng "combined summary" — dẫn đến việc gọi tool đúng nhưng không chắc chắn đã lấy đủ ngữ cảnh nguyên nhân gốc.
2. Vòng lặp sửa lỗi nguồn (Evidence Gate) trong `agent_before.py` chỉ gửi lại yêu cầu "bổ sung URL" dưới dạng chung chung, không chỉ rõ URL nào đã quan sát được ở đâu trong message, khiến model lặp lại cùng một lỗi trích dẫn sai nguồn ở tất cả các bước còn lại cho tới khi hết `max_steps` → bị chặn ở `score_cap: 50` và cuối cùng là `0` do không hoàn tất.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Trong `agent.py`, bổ sung mục **3. ĐIỀU TRA NHIỀU BƯỚC (Multi-step Chaining)** vào `SYSTEM_PROMPT`:
  > "Khi xem chi tiết một incident mà nội dung cập nhật có nhắc đến, trỏ tới hoặc thông báo đã gộp vào một incident khác ..., bạn BẮT BUỘC phải tiếp tục gọi tool `get_incident_details` với ID incident liên kết đó ở bước tiếp theo để đọc đầy đủ nguyên nhân và ảnh hưởng trước khi kết luận."

  Ngoài ra message nhắc Evidence Gate trong vòng lặp cũng được sửa để liệt kê rõ danh sách URL đã quan sát được (`urls_str = ', '.join(sorted(observed_urls))`) thay vì chỉ nói chung chung.
- Vì sao chọn thay đổi này: Lỗi gốc nằm ở việc agent không biết chắc phải bám theo chuỗi liên kết incident trước khi kết luận, và khi bị yêu cầu sửa nguồn thì không có đủ thông tin cụ thể (URL nào) để sửa đúng. Hai thay đổi trên trực tiếp giải quyết hai điểm này mà không thay đổi luồng xử lý tool-calling.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before2.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng (partial_mean_0_100) | 86.43 | 91.32 |
| Số case có điểm / tổng | 21/22 (1 unscored) | 19/22 (3 unscored) |
| Điểm case H6 | 0 | 50 |
| Trạng thái case H6 | `max_steps` | `completed` |
| Số tool calls (H6) | 2 | 2 |
| Số bước tới khi dừng (H6) | 8 (hết max_steps) | 3 |
| Tổng token (H6, usage) | 22 245 | 6 760 |

- Lỗi đã giải quyết: Case H6 không còn kẹt vòng lặp vô hạn ở Evidence Gate; agent dừng đúng ở bước 3 với `status = completed`, tiết kiệm ~70% token so với trước. Điểm tổng toàn bộ eval tăng từ 86.43 lên 91.32.
- Lỗi còn lại hoặc mới xuất hiện:
  - Case H6 vẫn bị `score_cap: 50` do vi phạm "Chưa gọi tool cần thiết: `get_status`" — câu hỏi yêu cầu "Xem status GitHub Actions" trước khi tra incident, nhưng agent bỏ qua bước này ở cả hai lần chạy. Đây là lỗi chưa được xử lý bởi thay đổi hiện tại.
  - Ở bản after xuất hiện thêm 2 case lỗi mới so với bản before: **A5** (`status = error`, 0 tool calls) và **H3** (`status = error`) không có điểm, trong khi bản before cả hai case này đều `completed`. Cần kiểm tra lại log lỗi cụ thể của provider cho hai case này trước khi kết luận nguyên nhân.

## 5. Giới hạn

- **Giới hạn của điểm judge**: Judge (`gemini-3.1-flash-lite`) chấm điểm dựa trên suy luận ngôn ngữ tự nhiên theo 4 tiêu chí (factuality, completeness, evidence, boundaries) — có thể không ổn định giữa các lần chấm cùng một câu trả lời, và điểm số có thể bị ảnh hưởng bởi cách diễn đạt câu trả lời hơn là tính đúng đắn tuyệt đối.
- **Giới hạn của snapshot**: Dữ liệu trạng thái GitHub/Cloudflare được chụp tại một thời điểm cố định (`captured_at`), chỉ phản ánh 50 incident gần nhất; không chứng minh được việc "không có sự cố" một cách tuyệt đối, và không cập nhật theo thời gian thực.
- **Giới hạn của việc chỉ chạy một lần mỗi phiên bản**: Do các model ngôn ngữ có tính ngẫu nhiên (temperature > 0 hoặc sampling), kết quả một lần chạy không đại diện đầy đủ cho hiệu năng trung bình của agent. Sự xuất hiện mới của 2 case lỗi (A5, H3) ở bản after có thể là nhiễu ngẫu nhiên từ provider chứ không hẳn do thay đổi code — cần chạy lặp lại nhiều lần (ví dụ n ≥ 5) để có kết luận đáng tin cậy hơn.