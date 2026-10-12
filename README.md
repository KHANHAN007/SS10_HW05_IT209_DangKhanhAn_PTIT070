## 1. Cấu trúc bài nộp

```text
ex05/
├── docker-compose.yml
├── .env.example
├── .gitignore
└── README.md
```

File `.env` chứa cấu hình thực tế trên máy và không đưa lên GitHub. File `.env.example` cung cấp các giá trị mẫu để người khác tạo cấu hình riêng.

## 2. Chuẩn bị

Sử dụng lại image `hw-backend:v1` đã build ở EX2. Người chạy bài cần có image này trên máy trước khi khởi động Compose.

```bash
docker image inspect hw-backend:v1 > /dev/null
cp .env.example .env
nano .env
```

Các biến cần khai báo: `DB_NAME`, `DB_USER`, `DB_PASSWORD` và `PORT`. Trong lần kiểm thử này, cổng trên máy cá nhân là `8083`.

Kiểm tra cấu hình:

```bash
docker compose config --quiet
```

Lệnh hoàn tất không có thông báo lỗi.

## 3. Kiến trúc

| Thành phần | Cấu hình |
|---|---|
| Service `backend` | Image `hw-backend:v1`, cổng nội bộ `3000` |
| Service `database` | Image `postgres:15-alpine`, cổng nội bộ `5432` |
| Network | `backend-net`, loại bridge |
| Named Volume | `pgdata`, mount tại `/var/lib/postgresql/data` |
| Thứ tự khởi động | Backend chờ database healthy bằng `depends_on` |
| Biến cấu hình | Đọc từ `.env` |

Compose sử dụng tên dự án `ex05`, nên network và volume thực tế có tên `ex05_backend-net` và `ex05_pgdata`.

## 4. Khởi chạy hệ thống

```bash
docker compose up -d
docker compose ps
```

Kết quả khởi chạy thực tế:

```text
Network ex05_backend-net  Created
Volume ex05_pgdata        Created
Container ex05-database-1 Healthy
Container ex05-backend-1  Started
```

Trích kết quả `docker compose ps`:

```text
NAME              IMAGE                SERVICE    STATUS                   PORTS
ex05-backend-1    hw-backend:v1         backend    Up Less than a second    0.0.0.0:8083->3000/tcp, [::]:8083->3000/tcp
ex05-database-1   postgres:15-alpine    database   Up 5 seconds (healthy)   5432/tcp
```

Hai service đã chạy; PostgreSQL ở trạng thái healthy. Database không publish cổng ra máy host.

## 5. Kiểm tra REST API

```bash
curl -i http://localhost:8083
```

Trích phản hồi thực tế:

```text
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: application/json; charset=utf-8
Content-Length: 100
```

Response JSON:

```json
{
  "status": "success",
  "message": "Hello from Docker Single-stage!",
  "student": "Dang Khanh An - PTIT070"
}
```

## 6. Kiểm tra DNS và kết nối mạng nội bộ

```bash
docker compose exec backend ping -c 2 database
```

Kết quả thực tế:

```text
PING database (172.23.0.2): 56 data bytes
64 bytes from 172.23.0.2: seq=0 ttl=64 time=0.591 ms
64 bytes from 172.23.0.2: seq=1 ttl=64 time=0.153 ms

--- database ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.153/0.372/0.591 ms
```

Backend phân giải được tên service `database` thành IP nội bộ. Image Backend của EX2 nhận biến môi trường kết nối nhưng chưa có mã truy vấn PostgreSQL; kết quả ping chứng minh DNS và kết nối mạng, không chứng minh truy vấn database từ ứng dụng.

## 7. Theo dõi log

```bash
docker compose logs -f
```

Trích các dòng xác nhận dịch vụ sẵn sàng:

```text
database-1 | PostgreSQL init process complete; ready for start up.
database-1 | LOG: listening on IPv4 address "0.0.0.0", port 5432
database-1 | LOG: listening on IPv6 address "::", port 5432
database-1 | LOG: database system is ready to accept connections
backend-1  | Server listening on port 3000
```

Nhấn `Ctrl+C` để thoát theo dõi log. Thao tác này không dừng hai service.

## 8. Hạ hệ thống

```bash
docker compose down
```

Kết quả thực tế:

```text
Container ex05-backend-1  Removed
Container ex05-database-1 Removed
Network ex05_backend-net Removed
```

Kiểm tra:

```bash
docker compose ps
```

Kết quả:

```text
NAME      IMAGE     COMMAND   SERVICE   CREATED   STATUS    PORTS
```

Không còn container của dự án đang chạy. Lệnh `down` không có tùy chọn `-v` nên giữ lại Named Volume.

## 9. Quản lý cấu hình

- `.env` được bỏ qua bởi `.gitignore`.
- `.env.example` chỉ chứa cấu hình mẫu, không chứa mật khẩu thực tế.
- Khi đổi tên user hoặc mật khẩu PostgreSQL trong `.env`, cần lưu ý các biến khởi tạo chỉ áp dụng khi thư mục dữ liệu còn trống; volume đã có database không tự thay đổi thông tin đăng nhập.

## 10. Kết quả

Docker Compose tự tạo network, volume và khởi chạy hai service. Backend trả HTTP 200, phân giải được tên `database`, PostgreSQL healthy và hệ thống được hạ thành công bằng `docker compose down`.
