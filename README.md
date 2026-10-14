# BÁO CÁO BÀI KIỂM TRA ĐẦU GIỜ

**Đề tài: Đóng gói và chạy Campus Hub với Docker**  
**Thời gian làm bài theo đề: 45 phút**

---

## 1. Thông tin sinh viên

- **Họ và tên:** Nguyễn Trí Thắng
- **Mã sinh viên:** B24DTCN450
- **Lớp:** CNTT4
- **Dự án sử dụng:** Campus Hub Base
- **Repository nộp bài:** https://github.com/trithangggggggg/PTIT_HNKS24_CNTT4_DevOps_Session13_BaiKiemTra

---

## 2. Mục tiêu và cấu trúc dự án

Đóng gói backend Campus Hub thành Docker Image và chạy backend cùng MySQL bằng Docker Compose. Backend nhận cấu hình database qua biến môi trường và sử dụng tên service `mysql` trong chuỗi kết nối.

Campus Hub hỗ trợ đăng ký, đăng nhập, xem và tạo bài đăng, bình luận, bày tỏ quan tâm và quản lý trạng thái bài đăng. Backend sử dụng Java 21, Spring Boot, Gradle, Spring Data JPA và Flyway; frontend sử dụng React và Vite.

| Tệp | Vai trò |
|---|---|
| `backend/Dockerfile` | Build JAR và tạo image chạy backend |
| `backend/.dockerignore` | Loại bỏ cache, kết quả build và cấu hình IDE khỏi build context |
| `backend/build.gradle` | Khai báo thư viện, Java 21 và tác vụ kiểm thử |
| `backend/src/main/resources/application.yml` | Nhận cấu hình kết nối database và cổng ứng dụng |
| `docker-compose.yml` | Khởi chạy MySQL, backend và frontend |
| `.env.example` | Cấu hình môi trường mẫu |
| `screenshots/` | Lưu ảnh minh chứng |

---

## 3. Phần 1 — Đóng gói backend thành Docker Image (50 điểm)

### 3.1. Kiểm thử backend bằng Gradle

Yêu cầu máy có JDK 21 để chạy kiểm thử trực tiếp. Từ thư mục gốc dự án, chạy trên PowerShell:

```powershell
cd backend
.\gradlew.bat test
cd ..
```

Trên Linux/macOS, dùng `./gradlew test` trong thư mục `backend`.

Kết quả cần đối chiếu là `BUILD SUCCESSFUL`. Báo cáo kiểm thử được Gradle tạo tại `backend/build/reports/tests/test/index.html`. Cấu hình test sử dụng H2 ở chế độ tương thích MySQL, không cần khởi động MySQL để chạy các bài kiểm thử này.

### 3.2. Phương án Dockerfile và .dockerignore

Dockerfile hiện có sử dụng hai stage:
1. **Build:** dùng `eclipse-temurin:21-jdk`, sao chép Gradle Wrapper và mã nguồn, xử lý xuống dòng CRLF của `gradlew`, sau đó chạy `./gradlew bootJar --no-daemon -x test` để tạo JAR.
2. **Runtime:** dùng `eclipse-temurin:21-jre-alpine`, sao chép JAR thành `/app/app.jar`, chạy bằng tài khoản `appuser` và công khai cổng `8080`.

Lệnh build JAR trong Dockerfile bỏ qua test, vì vậy cần chạy kiểm thử riêng ở bước 3.1.

Tệp `.dockerignore` loại bỏ `.gradle`, `build`, `out`, `.idea`, `*.iml`, `.git` và `.gitignore`.

### 3.3. Build và kiểm tra image

Từ thư mục gốc dự án:

```powershell
docker build -t local/campus-hub-backend:latest ./backend
docker images local/campus-hub-backend
```

Tên image trên khớp với cấu hình Compose mặc định khi chưa đặt `DOCKERHUB_USERNAME` và `IMAGE_TAG`. Kết quả cần có image `local/campus-hub-backend` với tag `latest` trong danh sách.

---

## 4. Phần 2 — Chạy bằng Docker Compose (50 điểm)

### 4.1. Cấu hình các service

| Service | Cấu hình chính | Cổng máy → container |
|---|---|---|
| `mysql` | Image `mysql:8.0`, database mặc định `campus_hub` | `3306:3306` |
| `backend` | Build từ `backend/Dockerfile`, đợi MySQL healthy | `8080:8080` |
| `frontend` | Build từ `frontend/Dockerfile`, phục vụ giao diện qua Nginx | `80:80` |

Đề yêu cầu tối thiểu backend và MySQL; Compose của base project có thêm frontend để kiểm tra giao diện.

Các service cùng tham gia network `campus-network`. Volume `mysql_data` lưu dữ liệu MySQL. MySQL có healthcheck bằng `mysqladmin ping`; backend sử dụng `depends_on` với điều kiện `service_healthy`.

Backend nhận các biến môi trường `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` và `SERVER_PORT`. Chuỗi kết nối trong Compose:

```text
jdbc:mysql://mysql:3306/${DB_NAME:-campus_hub}?createDatabaseIfNotExist=true&useSSL=false&allowPublicKeyRetrieval=true
```

Hostname `mysql` được phân giải qua network của Compose, giúp backend kết nối đúng container database. Flyway tạo schema khi ứng dụng khởi động; JPA kiểm tra schema bằng `ddl-auto: validate`.

### 4.2. Lệnh chạy hệ thống

Chuẩn bị Docker Engine hoặc Docker Desktop đang hoạt động. Chạy các lệnh sau từ thư mục gốc dự án:

```powershell
# Kiểm tra tính hợp lệ của cấu hình Compose
docker compose config --quiet

# Build và khởi chạy các service
docker compose up -d --build

# Kiểm tra trạng thái container
docker compose ps

# Xem log để đối chiếu quá trình khởi động và kết nối database
docker compose logs --tail=100 backend mysql
```

Compose có giá trị mặc định để chạy thử. Có thể sao chép `.env.example` thành `.env` và sửa các biến khi cần; không đưa mật khẩu thật hoặc token lên repository. Với cấu hình hiện tại, nếu đổi `DB_USERNAME` sang tài khoản khác `root`, cần tạo và cấp quyền tài khoản MySQL tương ứng.

Nếu chỉ chạy hai service theo phạm vi đề:

```powershell
docker compose up -d --build mysql backend
```

Dừng hệ thống:

```powershell
docker compose down
```

Lệnh trên giữ lại volume dữ liệu MySQL.

### 4.3. Cổng và URL kiểm tra

| Thành phần | Địa chỉ |
|---|---|
| Backend health | http://localhost:8080/actuator/health |
| API danh sách bài đăng | http://localhost:8080/api/posts |
| API chuyên mục | http://localhost:8080/api/categories |
| Frontend | http://localhost:80 |

Kiểm tra cả mã HTTP và nội dung health endpoint trên PowerShell:

```powershell
curl.exe -i http://localhost:8080/actuator/health
```

Trên Linux/macOS, dùng `curl -i http://localhost:8080/actuator/health`. Kết quả cần đạt là HTTP `200` và JSON có `"status":"UP"`. Đối chiếu thêm trạng thái container và log backend để xác nhận ứng dụng khởi động, kết nối MySQL thành công.

---

## 5. Ảnh minh chứng

### 5.1. Danh sách Docker Image

Ảnh chụp kết quả lệnh `docker images`:

![Danh sách Docker Image](screenshots/docker_images.png)

### 5.2. Trạng thái Docker Compose

Ảnh chụp kết quả lệnh `docker compose ps` thể hiện backend và frontend ở trạng thái `Up`, MySQL ở trạng thái `Up (healthy)`:

![Trạng thái các service](screenshots/docker_compose_ps.png)

### 5.3. Kết quả health endpoint

Ảnh chụp kiểm tra endpoint `/actuator/health` trả về HTTP `200` và `{"status":"UP"}`:

![HTTP 200 và trạng thái UP](screenshots/check.png)

---

## 6. Đối chiếu trước khi nộp

- [ ] Chạy Gradle test thành công và lưu kết quả.
- [ ] Build thành công image backend và kiểm tra bằng `docker images`.
- [ ] Kiểm tra cấu hình bằng `docker compose config --quiet`.
- [ ] Xác nhận backend và MySQL cùng hoạt động, backend kết nối được database.
- [ ] Gọi health endpoint nhận HTTP `200` và trạng thái `UP`.
- [ ] Thêm ảnh chụp thực tế vào thư mục `screenshots/` (`docker_images.png`, `docker_compose_ps.png`, `check.png`).
- [ ] Push mã nguồn, Dockerfile, `.dockerignore`, Compose, README cùng ảnh minh chứng lên GitHub repository: https://github.com/trithangggggggg/PTIT_HNKS24_CNTT4_DevOps_Session13_BaiKiemTra.
