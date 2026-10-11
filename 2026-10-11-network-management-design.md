# Thiết kế: Hệ thống tự động quản trị và quản lý hiệu suất mạng

Ngày: 2026-10-11 · Trạng thái: chờ duyệt spec

## 1. Mục tiêu và bối cảnh

Đồ án môn học của nhóm 2–4 người, thời gian 1–2 tháng. Xây dựng hệ thống Python + Netmiko giám sát và quản trị nhiều thiết bị Cisco IOS trong lab mô phỏng (GNS3/EVE-NG/CML), có web dashboard.

**Tiêu chí thành công**
1. Quản lý danh sách thiết bị (thêm, sửa, xóa, bật/tắt giám sát, kiểm tra kết nối).
2. Thu thập định kỳ CPU, RAM, trạng thái và lưu lượng interface từ nhiều thiết bị song song.
3. Phát hiện và hiển thị cảnh báo sự cố (mất kết nối, CPU/RAM cao, interface down), tự đóng cảnh báo khi hết sự cố.
4. Sao lưu running-config theo lịch hoặc theo yêu cầu, phát hiện thay đổi cấu hình.
5. Tạo báo cáo theo khoảng thời gian, xuất HTML/PDF/CSV.
6. Khi một thiết bị mất kết nối, các thiết bị còn lại vẫn được thu thập bình thường; hệ thống ghi nhận sự cố và tự phục hồi khi thiết bị trở lại.

## 2. Phạm vi

**Trong phạm vi:** thiết bị Cisco IOS (`device_type=cisco_ios`), quy mô 3–10 thiết bị, thu thập qua SSH/CLI bằng Netmiko, cảnh báo hiển thị trên dashboard, một tài khoản quản trị web.

## 3. Kiến trúc

Hai tiến trình độc lập chỉ giao tiếp qua một database dùng chung.

```
[Router Cisco IOS] <--SSH/Netmiko-- COLLECTOR --ghi--> DATABASE <--đọc/ghi-- WEB APP (FastAPI)
```

- **Collector:** tiến trình nền, không có HTTP. Thu thập, cảnh báo, sao lưu, xử lý yêu cầu từ web.
- **Web app:** FastAPI + Jinja2 + Chart.js. Không tự SSH vào thiết bị; mọi tác vụ cần SSH được ghi thành bản ghi trong bảng `requests` để collector thực hiện.
- **Database:** PostgreSQL (truy cập qua SQLAlchemy, driver `psycopg`). Chạy bằng Docker Compose để dựng nhanh trên máy lab.
  Quy ước: khai báo kiểu cột tường minh, thời gian lưu dạng `timestamptz` (UTC); schema quản lý bằng Alembic ngay từ đầu; collector và web mỗi bên có connection pool riêng; chuỗi kết nối lấy từ biến môi trường `NETMON_DATABASE_URL` (không ghi mật khẩu vào `config.yaml`).
- Web bị dừng thì thu thập vẫn chạy; collector bị dừng thì dashboard vẫn xem được dữ liệu cũ.

**Công nghệ:** Python 3.11+, Netmiko, SQLAlchemy, Alembic, psycopg, PostgreSQL, FastAPI, Jinja2, Chart.js, WeasyPrint (PDF), PyYAML, pytest.

### Cấu trúc thư mục
```
netmon/
  common/        # model DB, đọc cấu hình, logging (dùng chung)
  collector/
    connection.py  # bọc Netmiko: timeout, phân loại lỗi
    polling.py     # vòng thu thập, thread pool, backoff
    parsers.py     # parse output `show` thành số liệu
    alerting.py    # đối chiếu ngưỡng, mở/đóng cảnh báo
    backup.py      # sao lưu, hash, diff
    requests.py    # xử lý bảng requests từ web
  web/             # routes, templates, static
  tests/
config.yaml
```

## 4. Mô hình dữ liệu

| Bảng | Trường chính |
|---|---|
| `devices` | id, name, host, device_type, username, trạng thái (`up`/`degraded`/`down`), `fail_count`, `fail_reason`, `last_polled_at`, `next_poll_at`, `monitoring_enabled`, ngưỡng ghi đè (tùy chọn) |
| `metrics` | device_id, timestamp, cpu_percent, mem_percent |
| `interface_metrics` | device_id, timestamp, interface, status, in_bytes, out_bytes |
| `alerts` | id, device_id, loại (`device_down`, `cpu_high`, `mem_high`, `interface_down`, `auth_failed`), mức độ, `opened_at`, `closed_at`, thông điệp |
| `config_backups` | id, device_id, timestamp, content, sha256, `changed` (bool), `diff` (văn bản, rỗng nếu không đổi) |
| `requests` | id, loại (`backup_now`, `test_connection`), device_id, trạng thái (`pending`/`running`/`done`/`failed`), kết quả, thời điểm tạo/hoàn thành |
| `metrics_hourly` | device_id, giờ, cpu_avg, cpu_max, mem_avg, mem_max, tỷ lệ up (phục vụ báo cáo dài hạn) |

Mật khẩu thiết bị không lưu trong DB: lấy từ biến môi trường theo tên thiết bị (`NETMON_PASS_<TÊN>`) hoặc `.env` nằm ngoài kho mã. Mật khẩu tài khoản web được băm (bcrypt).

## 5. Collector

### 5.1 Vòng thu thập
Chu kỳ mặc định 60 giây. Mỗi vòng:
1. Chọn thiết bị `monitoring_enabled` và `next_poll_at <= now`.
2. Giao mỗi thiết bị cho một thread trong pool (tối đa 10 thread).
3. Thread mở phiên Netmiko (timeout kết nối 10 s, timeout lệnh 20 s), chạy `show processes cpu`, `show memory statistics`, `show interfaces`, `show version`, rồi đóng phiên.
4. Parser chuyển output thành số liệu, ghi `metrics` và `interface_metrics`, cập nhật `devices`.
5. Gọi `alerting`.
Không chạy chồng hai vòng: vòng sau chờ vòng trước kết thúc. Exception không lường trước trong một thread được bắt, ghi log, không làm dừng vòng.

### 5.2 Máy trạng thái và xử lý mất kết nối
- Thu thập thất bại: `fail_count += 1`, ghi `fail_reason` (`timeout`, `unreachable`, `auth_failed`, `command_error`).
- `fail_count` 1–2 → `degraded`; `fail_count >= 3` → `down` và mở cảnh báo `device_down`.
- Thu thập thành công: `fail_count = 0`, trạng thái `up`, đóng cảnh báo `device_down` (nếu có) và ghi thời điểm phục hồi.
- Backoff cho thiết bị `down`: thử lại sau 1, 2, 5 phút rồi giữ 5 phút. Thiết bị `degraded` được thử lại ở vòng kế tiếp.
- Lỗi `auth_failed`: thiết bị chuyển `down` với cảnh báo `auth_failed` và **không tự thử lại** (tránh khóa tài khoản trên thiết bị) cho đến khi người dùng sửa thông tin đăng nhập hoặc bấm "kiểm tra kết nối".
- Trạng thái nằm trong DB nên collector khởi động lại vẫn tiếp tục đúng.

### 5.3 Cảnh báo (`alerting.py`)
- `cpu_high`: CPU > 80% trong 3 lần đo liên tiếp. `mem_high`: RAM > 85% trong 3 lần đo liên tiếp. Ngưỡng và số lần đo cấu hình được, mỗi thiết bị có thể ghi đè.
- `interface_down`: interface chuyển từ up sang down giữa hai lần đo (bỏ qua interface bị shutdown theo cấu hình admin).
- Mỗi loại cảnh báo trên mỗi thiết bị (và mỗi interface) chỉ có một bản đang mở; điều kiện hết thì tự đóng.

### 5.4 Sao lưu cấu hình (`backup.py`)
- Mặc định mỗi ngày một lần lúc 02:00, hoặc khi có yêu cầu `backup_now`.
- Lấy `show running-config`, bỏ các dòng thay đổi liên tục không ý nghĩa (ví dụ `! Last configuration change`, `ntp clock-period`) trước khi tính SHA-256.
- Hash khác bản trước: lưu nội dung, `changed=true` và `diff` (unified diff). Hash giống: không lưu nội dung mới, chỉ ghi nhận đã kiểm tra.
- Thiết bị `down`: bỏ qua, ghi log thất bại, thử lại ở lần sau. Có tùy chọn xuất bản sao lưu ra file `.txt` theo thiết bị và thời điểm.

### 5.5 Bảo trì dữ liệu
Hằng giờ tổng hợp `metrics` vào `metrics_hourly`; dữ liệu thô cũ hơn 30 ngày bị xóa.

### 5.6 Xử lý yêu cầu từ web
Mỗi vài giây collector đọc `requests` ở trạng thái `pending`, thực hiện (`backup_now`, `test_connection`) và cập nhật kết quả.

## 6. Web app

- **Đăng nhập:** một tài khoản quản trị, phiên bằng cookie; mọi trang đều yêu cầu đăng nhập.
- **Tổng quan:** số thiết bị up/degraded/down, số cảnh báo đang mở, bảng thiết bị (trạng thái, CPU, RAM, lần thu thập cuối).
- **Chi tiết thiết bị:** biểu đồ CPU, RAM, lưu lượng interface theo thời gian; danh sách interface; lịch sử cảnh báo; các bản sao lưu và diff.
- **Cảnh báo:** lọc theo mức độ, trạng thái (mở/đóng), thiết bị.
- **Quản lý thiết bị:** thêm/sửa/xóa, bật/tắt giám sát, nút "kiểm tra kết nối" và "sao lưu ngay" (tạo bản ghi `requests`, trang hiển thị kết quả khi xong).
- Trang tự làm mới bằng polling JavaScript (mặc định 30 giây); không dùng WebSocket.

## 7. Báo cáo

Chọn khoảng thời gian (ngày/tuần/tháng hoặc tùy chọn). Nội dung: độ khả dụng (% thời gian up) từng thiết bị, CPU/RAM trung bình và đỉnh, số sự cố và thời gian xử lý trung bình, danh sách thay đổi cấu hình. Xem trên web, xuất HTML, PDF (WeasyPrint) và CSV dữ liệu thô. Báo cáo dài hạn dùng `metrics_hourly`.

## 8. Cấu hình

`config.yaml`: chu kỳ thu thập, số thread, timeout, ngưỡng cảnh báo và số lần đo, lịch sao lưu, thời gian giữ dữ liệu. Thông tin nhạy cảm (chuỗi kết nối DB, mật khẩu thiết bị) chỉ qua biến môi trường hoặc `.env` (không đưa vào git). Chạy bằng `python -m netmon.collector` và `python -m netmon.web`; `docker-compose.yml` dựng PostgreSQL; script `run_demo.sh` khởi động DB, collector và web.

## 9. Kiểm thử

- **Unit:** parser (output `show` mẫu lưu trong `tests/fixtures`), máy trạng thái và backoff (mock Netmiko: timeout, auth_failed, phục hồi), logic ngưỡng cảnh báo, hash/diff, tính độ khả dụng.
- **Integration:** collector ghi DB, web đọc đúng; luồng yêu cầu web → collector → kết quả hiển thị.
- **Lab (topology 4–6 router Cisco IOS), kịch bản nghiệm thu:**
  1. Chạy bình thường: tất cả thiết bị có dữ liệu mới mỗi chu kỳ.
  2. Tắt một router: chuyển up → degraded → down trong khoảng 3 chu kỳ, mở cảnh báo `device_down`, các router khác vẫn được thu thập đúng chu kỳ.
  3. Bật lại router: trở về up, đóng cảnh báo.
  4. Đổi cấu hình một router: lần sao lưu kế tiếp đánh dấu `changed` và có diff.
  5. Sai mật khẩu một thiết bị: cảnh báo `auth_failed`, không thử lại liên tục.
  6. Dừng web app: collector vẫn thu thập; dừng collector: dashboard vẫn xem được dữ liệu cũ.

## 10. Phân chia công việc gợi ý cho nhóm

Thống nhất schema DB (mục 4) trước tiên. Sau đó: (A) `connection`, `polling`, `parsers` và máy trạng thái; (B) `alerting`, `backup`, `requests`, bảo trì dữ liệu; (C) web app, báo cáo, script chạy demo. Kiểm thử lab do cả nhóm cùng thực hiện.

## 11. Rủi ro đã biết

- Output `show` khác nhau theo phiên bản IOS trong lab: parser dùng regex dung sai và có fixture cho các phiên bản dùng thật.
- Nhiều thread của collector và web cùng truy cập DB: dùng connection pool, giao dịch ngắn, ghi theo lô mỗi vòng thu thập.
- Hạn chế tài nguyên máy chạy lab: giới hạn thread pool và chu kỳ có thể tăng lên.
