# BÁO CÁO THỰC HÀNH: TÁCH BIỆT LUỒNG CI/CD ĐA MÔI TRƯỜNG (STAGING & PRODUCTION) VỚI GITHUB ENVIRONMENTS

---

- **Môn học**: DevOps Fundamentals (IT-209)
- **Session**: 12 - Tối ưu hóa CI/CD Pipeline
- **Bài tập**: Bài 3 - Tách biệt luồng CI/CD đa môi trường (Staging & Production) với GitHub Environments
- **Học viên**: Hồ Quốc Khải
- **Email**: khaiho6040@gmail.com
- **Đường dẫn gợi ý nộp bài**: `homework/session_12/ex3/`

---

## 1. MỤC TIÊU VÀ NGUYÊN LÝ BẢO VỆ MÔI TRƯỜNG TRIỂN KHAI

### 1.1. Mục tiêu
- Quản lý hạ tầng triển khai đa môi trường (**Staging** và **Production**) một cách chuyên nghiệp thông qua tính năng **GitHub Environments**.
- Thiết lập quy tắc bảo vệ môi trường (**Environment Protection Rules**) với cơ chế phê duyệt thủ công (**Required Reviewers**) trước khi tiến hành cập nhật ứng dụng lên môi trường Production.
- Phân tách luồng triển khai tự động theo nhánh Git:
  - Nhánh `develop`: Tự động triển khai lên môi trường `staging` để kiểm thử tích hợp.
  - Nhánh `main`: Triển khai lên môi trường `production`, yêu cầu phê duyệt từ người có thẩm quyền.

### 1.2. Nguyên lý hoạt động của Environment Gates & Required Reviewers
- **Deployment Gates**: Trong quy trình DevOps hiện đại, việc tự động hóa hoàn toàn (Continuous Deployment) lên Production tiềm ẩn rủi ro nếu thiếu sự kiểm tra cuối cùng từ phía con người (Lead/Release Manager/QA).
- **Cơ chế dừng tiến trình (Waiting for review)**:
  1. Khi một Job khai báo từ khóa `environment: production`, GitHub Actions sẽ tự động kiểm tra các quy tắc bảo vệ đã cấu hình cho môi trường đó.
  2. Nếu phát hiện quy tắc `Required reviewers`, runner **không thực thi ngay** các bước bên dưới mà chuyển sang trạng thái tạm dừng: **Waiting for review**.
  3. GitHub gửi thông báo (email/web) cho reviewer được chỉ định.
  4. Chỉ khi reviewer bấm **"Approve and deploy"**, runner mới tiếp tục tải mã nguồn và thực thi các bước triển khai production.
  5. Nếu reviewer bấm **Reject** hoặc sau 30 ngày không có phản hồi, Job sẽ bị hủy nhằm đảm bảo an toàn tuyệt đối cho hệ thống đang vận hành.

---

## 2. CẤU TRÚC THƯ MỤC DỰ ÁN

```
Ss12/3/
├── .github/
│   └── workflows/
│       └── cd-multi-env.yml     # Kịch bản GitHub Actions CD đa môi trường Staging/Production
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar   # Gradle wrapper executable
│       └── gradle-wrapper.properties # Cấu hình phiên bản Gradle (9.5.1)
├── src/
│   ├── main/
│   │   ├── java/com/example/tasks/
│   │   │   ├── controller/
│   │   │   │   └── TaskController.java        # REST Controller quản lý công việc
│   │   │   ├── model/
│   │   │   │   └── Task.java                  # JPA Entity Task
│   │   │   ├── repository/
│   │   │   │   └── TaskRepository.java        # JpaRepository giao tiếp Database
│   │   │   └── TaskApiApplication.java       # Spring Boot Main Application
│   │   └── resources/
│   │       ├── db/
│   │       │   └── taskdb.sql                 # DDL khởi tạo cơ sở dữ liệu
│   │       └── application.properties        # Cấu hình Spring Datasource & JPA
│   └── test/
│       ├── java/com/example/tasks/
│       │   ├── controller/
│       │   │   └── TaskControllerTest.java    # Unit test JUnit 5 & Mockito
│       │   └── TaskApiApplicationTests.java   # Spring Boot Context load test
│       └── resources/
│           └── application.properties        # Cấu hình H2 in-memory DB cho test
├── .env.example.properties       # Mẫu biến môi trường
├── .gitattributes                # Quy định xử lý line endings
├── .gitignore                    # Loại trừ build artifacts, cache, logs
├── build.gradle                  # Cấu hình Gradle build & sourceCompatibility 17
├── docker-compose.yml            # Docker Compose service
├── Dockerfile                    # Multi-stage Dockerfile
├── gradlew                       # Gradle wrapper script (Linux/macOS)
├── gradlew.bat                   # Gradle wrapper script (Windows)
├── settings.gradle               # Tên root project
└── README.md                     # Báo cáo thực hành chi tiết bài 3
```

---

## 3. NỘI DUNG KỊCH BẢN GITHUB ACTIONS WORKFLOW

Tệp kịch bản được lưu tại: `.github/workflows/cd-multi-env.yml`

```yaml
name: Multi-Environment CD (Staging & Production)

on:
  push:
    branches:
      - main
      - develop
  workflow_dispatch:

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.example.com/api/tasks
    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Kiểm tra thông tin môi trường Staging
        run: |
          echo "=========================================="
          echo "🚀 Triển khai lên môi trường: STAGING"
          echo "Nhánh kích hoạt: ${{ github.ref_name }}"
          echo "Commit SHA: ${{ github.sha }}"
          echo "Người thực hiện: ${{ github.actor }}"
          echo "=========================================="

      - name: Thực thi kịch bản triển khai Staging
        run: |
          echo "1. Chuẩn bị container image cho Staging..."
          echo "2. Cập nhật cấu hình môi trường staging..."
          echo "3. Khởi động ứng dụng Task API trên cụm Staging..."
          echo "4. Kiểm tra trạng thái dịch vụ (Health Check)..."
          echo "✅ Triển khai hoàn tất thành công lên Staging!"

  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://api.prod.example.com
    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Kiểm tra thông tin môi trường Production
        run: |
          echo "=========================================="
          echo "🛡️ Triển khai lên môi trường: PRODUCTION"
          echo "Đã vượt qua bước kiểm duyệt thủ công (Manual Approval)!"
          echo "Nhánh kích hoạt: ${{ github.ref_name }}"
          echo "Commit SHA: ${{ github.sha }}"
          echo "Người phê duyệt & thực thi: ${{ github.actor }}"
          echo "=========================================="

      - name: Thực thi kịch bản triển khai Production (Zero Downtime)
        run: |
          echo "1. Xác thực bảo mật chứng chỉ sản xuất..."
          echo "2. Kéo image phiên bản chính thức (Production release)..."
          echo "3. Triển khai theo chiến lược Rolling Update..."
          echo "4. Kiểm tra tính toàn vẹn hệ thống và giám sát APM..."
          echo "✅ Triển khai hoàn tất thành công lên Production!"
```

### Phân tích chi tiết từng khối chỉ thị:

1. **`on: push: branches: [main, develop]`**:
   - Workflow lắng nghe các sự kiện đẩy code lên cả hai nhánh phát triển quan trọng là `develop` và `main`.

2. **Job `deploy-staging`**:
   - **`if: github.ref == 'refs/heads/develop'`**: Điều kiện chỉ thực thi khi commit xuất phát từ nhánh `develop`.
   - **`environment.name: staging`**: Liên kết job với môi trường `staging` trên GitHub.
   - **`environment.url`**: Cung cấp đường dẫn trực tiếp đến ứng dụng sau khi deploy để người dùng có thể nhấp vào kiểm tra ngay từ giao diện pipeline.

3. **Job `deploy-prod`**:
   - **`if: github.ref == 'refs/heads/main'`**: Điều kiện chỉ thực thi khi commit xuất phát từ nhánh `main`.
   - **`environment.name: production`**: Liên kết job với môi trường `production`. Nhờ môi trường này đã được bật cấu hình `Required reviewers`, tiến trình triển khai sẽ bị chặn lại cho đến khi nhận được phê duyệt hợp lệ.

---

## 4. HƯỚNG DẪN CẤU HÌNH GITHUB ENVIRONMENTS VÀ THỰC THI

### Bước 1: Cấu hình Environments trên giao diện GitHub
1. Mở GitHub Repository của bạn trên trình duyệt.
2. Điều hướng đến: **Settings** -> **Environments** (nằm ở thanh bên trái).
3. Tạo môi trường **`staging`**:
   - Nhấn **New environment**, nhập tên: `staging` -> Nhấn **Configure environment**.
   - Môi trường này không cần bật Protection Rules (để triển khai tự động).
4. Tạo môi trường **`production`**:
   - Nhấn **New environment**, nhập tên: `production` -> Nhấn **Configure environment**.
   - Tại mục **Environment protection rules**, tích chọn: :ballot_box_with_check: **Required reviewers**.
   - Trong ô tìm kiếm, nhập username GitHub của bạn và chọn tài khoản của mình làm người duyệt.
   - Nhấn **Save protection rules**.

### Bước 2: Tự đẩy code lên nhánh `main`
```bash
# Di chuyển vào thư mục bài 3
cd "d:\[IT-209] DevOps Fundamentals\Ss12\3"

# Khởi tạo git và push nhánh main
git init
git checkout -b main
git add .
git commit -m "feat: setup multi-environment CD pipeline with production approval gate"
git remote add origin https://github.com/<your-username>/<your-repo-name>.git
git push -u origin main
```

### Bước 3: Kiểm tra cơ chế phê duyệt thủ công
1. Vào tab **Actions** trên GitHub repository và click vào phiên chạy mới nhất trên nhánh `main`.
2. Bạn sẽ thấy Job `deploy-prod` chuyển sang trạng thái màu vàng: **Waiting for review** kèm thông báo:
   > *"1 environment waiting for review: production"*
3. Nhấp vào nút **Review deployments**, tích chọn môi trường `production`, có thể nhập lời nhận xét (ví dụ: *"Approved for production release"*), sau đó nhấn nút xanh: **Approve and deploy**.
4. **Chụp ảnh màn hình cửa sổ phê duyệt (Review deployments) và trạng thái Waiting for review** để làm tài liệu minh chứng nộp bài.
5. Sau khi duyệt, Job `deploy-prod` lập tức khởi chạy, in nhật ký triển khai và chuyển sang trạng thái tích xanh (**Success**).
