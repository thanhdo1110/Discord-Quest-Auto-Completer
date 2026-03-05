# Discord Quest Auto-Completer

Công cụ tự động quét, nhận và hoàn thành các Quest trên Discord.

> **Server cộng đồng:** [Tham gia Discord để nghịch bot auto](https://t.me/dothanh1110/88)

---

## Mục lục

- [Tính năng](#tính-năng)
- [Yêu cầu](#yêu-cầu)
- [Cách sử dụng](#cách-sử-dụng)
- [Cấu hình](#cấu-hình)
- [Kiến trúc mã nguồn](#kiến-trúc-mã-nguồn)
  - [Hằng số & Cấu hình](#hằng-số--cấu-hình)
  - [Logging](#logging)
  - [Build Number](#build-number)
  - [DiscordAPI](#discordapi)
  - [Quest Helpers](#quest-helpers)
  - [QuestAutocompleter](#questautocompleter)
- [Luồng hoạt động](#luồng-hoạt-động)
- [Các loại task hỗ trợ](#các-loại-task-hỗ-trợ)

---

## Tính năng

| Tính năng | Mô tả |
|---|---|
| **Auto Scan** | Tự động quét quest mới theo chu kỳ (`POLL_INTERVAL`) |
| **Auto Accept** | Tự động đăng ký (enroll) các quest chưa nhận |
| **Auto Complete** | Tự động hoàn thành quest bằng cách gửi progress/heartbeat |
| **Rate Limit Handling** | Tự động chờ và retry khi bị rate limit (429) |
| **Build Number Fetch** | Tự động lấy `client_build_number` mới nhất từ Discord web app |

---

## Yêu cầu

- Python 3.7+
- Thư viện: `requests`

```bash
pip install requests
```

---

## Cách sử dụng

### 1. Truyền token qua argument

```bash
python3 main.py <DISCORD_TOKEN>
```

### 2. Lưu token vào file `.token`

Tạo file `.token` trong cùng thư mục chứa token, sau đó chạy:

```bash
python3 main.py
```

### 3. Nhập token thủ công

Chạy không có argument, chương trình sẽ yêu cầu nhập token:

```bash
python3 main.py
```

> **Lưu ý:** Dùng `Ctrl+C` để dừng chương trình.

---

## Cấu hình

Các hằng số cấu hình nằm ở đầu file `main.py`:

| Biến | Mặc định | Mô tả |
|---|---|---|
| `API_BASE` | `https://discord.com/api/v9` | Base URL của Discord API |
| `POLL_INTERVAL` | `60` | Số giây giữa các lần quét quest |
| `HEARTBEAT_INTERVAL` | `20` | Số giây giữa các lần gửi heartbeat |
| `AUTO_ACCEPT` | `True` | Tự động nhận tất cả quest khả dụng |
| `LOG_PROGRESS` | `True` | Hiển thị log tiến độ |
| `DEBUG` | `True` | Bật log debug chi tiết |

---

## Kiến trúc mã nguồn

### Hằng số & Cấu hình

- `SUPPORTED_TASKS` — Danh sách các loại task mà tool hỗ trợ hoàn thành tự động.

### Logging

**`Colors`** — Class chứa mã màu ANSI cho terminal output.

**`log(msg, level)`** — Hàm log với các mức:

| Level | Ý nghĩa |
|---|---|
| `info` | Thông tin chung |
| `ok` | Thành công |
| `warn` | Cảnh báo |
| `error` | Lỗi |
| `progress` | Tiến độ (có thể tắt qua `LOG_PROGRESS`) |
| `debug` | Debug (có thể tắt qua `DEBUG`) |

### Build Number

- **`fetch_latest_build_number()`** — Scrape trang Discord web app để lấy `client_build_number` mới nhất. Fallback về giá trị mặc định nếu thất bại.
- **`make_super_properties(build_number)`** — Tạo header `X-Super-Properties` dạng base64, giả lập Discord Desktop Client.

### DiscordAPI

Class quản lý HTTP session với Discord API.

| Method | Mô tả |
|---|---|
| `__init__(token, build_number)` | Khởi tạo session với headers giả lập Discord Client |
| `get(path)` | Gửi GET request |
| `post(path, payload)` | Gửi POST request |
| `validate_token()` | Kiểm tra token hợp lệ, in thông tin user |

### Quest Helpers

Các hàm tiện ích để trích xuất dữ liệu từ quest object (hỗ trợ cả camelCase và snake_case):

| Hàm | Mô tả |
|---|---|
| `_get(d, *keys)` | Lấy giá trị từ dict với nhiều key thay thế |
| `get_task_config(quest)` | Lấy task config từ quest |
| `get_quest_name(quest)` | Lấy tên quest (fallback qua nhiều field) |
| `get_expires_at(quest)` | Lấy thời gian hết hạn |
| `get_user_status(quest)` | Lấy trạng thái user cho quest |
| `is_completable(quest)` | Kiểm tra quest có thể hoàn thành không |
| `is_enrolled(quest)` | Kiểm tra đã đăng ký quest chưa |
| `is_completed(quest)` | Kiểm tra đã hoàn thành quest chưa |
| `get_task_type(quest)` | Lấy loại task của quest |
| `get_seconds_needed(quest)` | Tổng số giây cần để hoàn thành |
| `get_seconds_done(quest)` | Số giây đã hoàn thành |
| `get_enrolled_at(quest)` | Thời điểm đăng ký quest |

### QuestAutocompleter

Class chính điều khiển toàn bộ logic auto-complete.

| Method | Mô tả |
|---|---|
| `fetch_quests()` | Gọi API lấy danh sách quest, xử lý rate limit |
| `enroll_quest(quest)` | Đăng ký một quest (retry tối đa 3 lần) |
| `auto_accept(quests)` | Tự động đăng ký tất cả quest chưa nhận |
| `complete_video(quest)` | Hoàn thành quest dạng xem video (gửi `video-progress`) |
| `complete_heartbeat(quest)` | Hoàn thành quest dạng chơi/stream game (gửi `heartbeat`) |
| `complete_activity(quest)` | Hoàn thành quest dạng activity (gửi `heartbeat`) |
| `process_quest(quest)` | Xử lý một quest — chọn phương thức phù hợp |
| `run()` | Vòng lặp chính: quét → nhận → hoàn thành → chờ → lặp lại |

---

## Luồng hoạt động

```
Khởi động
  │
  ├─ Đọc token (argument / .token / nhập tay)
  ├─ Lấy build number từ Discord
  ├─ Validate token
  │
  └─ Vòng lặp chính (mỗi POLL_INTERVAL giây):
       │
       ├─ Fetch danh sách quest
       ├─ Hiển thị tổng quan (tổng / enrolled / completed)
       ├─ Auto-accept các quest chưa nhận
       ├─ Lọc quest cần hoàn thành
       │
       └─ Với mỗi quest cần hoàn thành:
            │
            ├─ WATCH_VIDEO / WATCH_VIDEO_ON_MOBILE
            │     └─ Gửi POST /quests/{id}/video-progress liên tục
            │
            ├─ PLAY_ON_DESKTOP / STREAM_ON_DESKTOP
            │     └─ Gửi POST /quests/{id}/heartbeat mỗi 20s
            │
            └─ PLAY_ACTIVITY
                  └─ Gửi POST /quests/{id}/heartbeat mỗi 20s
```

---

## Các loại task hỗ trợ

| Task | API Endpoint | Cơ chế |
|---|---|---|
| `WATCH_VIDEO` | `/quests/{id}/video-progress` | Gửi timestamp tăng dần, tốc độ ~7s/lần |
| `WATCH_VIDEO_ON_MOBILE` | `/quests/{id}/video-progress` | Tương tự `WATCH_VIDEO` |
| `PLAY_ON_DESKTOP` | `/quests/{id}/heartbeat` | Heartbeat mỗi 20s với `stream_key` ngẫu nhiên |
| `STREAM_ON_DESKTOP` | `/quests/{id}/heartbeat` | Tương tự `PLAY_ON_DESKTOP` |
| `PLAY_ACTIVITY` | `/quests/{id}/heartbeat` | Heartbeat mỗi 20s với `stream_key` cố định |
