# BÁO CÁO KỸ THUẬT: TỐI ƯU TỐC ĐỘ CI BẰNG GRADLE DEPENDENCY CACHING

- **Học viên:** Nguyễn Văn Toàn
- **Môn học:** CI/CD & DevOps Automation
- **Bài thực hành:** Bài 3 - Session 11: Tối ưu tốc độ CI bằng Gradle Dependency Caching
- **Kho lưu trữ GitHub:** [https://github.com/nguyenvantoan-code/homework-session_11-ex3-](https://github.com/nguyenvantoan-code/homework-session_11-ex3-)

---

## 1. Mục tiêu

Trong quy trình CI (Continuous Integration), mỗi khi một commit mới được đẩy lên GitHub, GitHub Actions Runner (máy ảo tạm thời `ubuntu-latest`) sẽ được khởi tạo từ trạng thái hoàn toàn trắng (clean environment). Theo mặc định:
- Gradle và các thư viện phụ thuộc (dependencies từ Maven Central) phải tải lại toàn bộ từ Internet qua mạng mỗi lần chạy.
- Quá trình này tiêu tốn băng thông, làm chậm thời gian phản hồi của pipeline CI và giảm năng suất phát triển phần mềm.

**Mục tiêu của bài thực hành:**
- Kích hoạt tính năng caching cho Gradle Dependencies thông qua tham số `cache: 'gradle'` trong action `actions/setup-java@v5`.
- Thực hiện push code 2 lần liên tiếp để đo lường, kiểm chứng sự chênh lệch thời gian và xác minh cơ chế Cache Hit / Cache Restore thành công.

---

## 2. Cấu hình kịch bản GitHub Actions (`.github/workflows/ci.yml`)

Kịch bản CI được cấu hình bổ sung thuộc tính `cache: 'gradle'` trong bước thiết lập Java:

```yaml
name: Java Spring Boot CI (Gradle)

on:
  push:
    branches: ["main"]

jobs:
  test:
    name: Run Unit Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: "17"
          distribution: "temurin"
          cache: "gradle"

      - name: Cấp quyền cho Gradle Wrapper
        run: chmod +x gradlew

      - name: Thực thi JUnit Test với Gradle
        run: ./gradlew test

      - name: Đóng gói tệp JAR với Gradle
        run: ./gradlew bootJar

      - name: Lưu trữ thành phẩm JAR Artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: build/libs/*.jar
```

---

## 3. Cơ chế hoạt động của Gradle Caching trong `actions/setup-java`

Action `actions/setup-java@v5` tích hợp sẵn cơ chế cache thông minh của GitHub Actions Toolkit:
1. **Khóa Cache (Cache Key):** Được tự động sinh ra dựa trên hệ điều hành, kiến trúc và giá trị băm (hash) của tất cả các tệp cấu hình Gradle trong dự án (bao gồm `build.gradle`, `settings.gradle`, `gradle-wrapper.properties`):
   - Cache dependencies: `setup-java-Linux-x64-gradle-<hash>`
   - Cache wrapper: `setup-java-Linux-x64-gradle-wrapper-<hash>`
2. **Lần chạy đầu tiên (Run #1 - Cache Miss):**
   - Runner kiểm tra trên GitHub Cache Storage nhưng chưa tồn tại khóa cache phù hợp (`gradle cache is not found`).
   - Gradle tiến hành tải Gradle Distribution (9.8.1) và toàn bộ các dependencies từ Maven Central.
   - Tại bước hậu kỳ (Post Step: `Post Thiết lập môi trường Java JDK 17`), action tự động nén thư mục `~/.gradle/caches` và `~/.gradle/wrapper` rồi tải lên lưu trữ cache của GitHub.
3. **Lần chạy thứ hai (Run #2 - Cache Hit):**
   - Runner tìm thấy khóa cache đã lưu từ Run #1.
   - Quá trình khôi phục cache diễn ra ngay trong bước `setup-java` (`Cache restored successfully from key: ...`).
   - Khi bước `./gradlew test` chạy, Gradle sử dụng trực tiếp các thư viện đã có trong cache cục bộ mà không cần tải lại từ Internet, giúp rút ngắn đáng kể thời gian thực thi.

---

## 4. Bảng số liệu so sánh thời gian thực thi thực tế

Dưới đây là số liệu trích xuất chính xác từ GitHub Actions API cho 2 lần chạy liên tiếp trên kho lưu trữ `homework-session_11-ex3`:

| Tiêu chí / Bước thực thi | Lần 1: Chưa có Cache (Run #1) | Lần 2: Đã có Cache (Run #2) | Chênh lệch & Hiệu quả |
| :--- | :---: | :---: | :---: |
| **Run ID trên GitHub Actions** | `38584634966` | `38585195136` | — |
| **Trạng thái (Conclusion)** | `success` | `success` | Thành công 100% |
| **1. Checkout mã nguồn** | 1s | 1s | Không đổi |
| **2. Thiết lập Java JDK 17** | 1s (Cache Miss) | 6s (Tải & Giải nén ~184 MB Cache) | +5s (bù lại cho test) |
| **3. Cấp quyền Gradle Wrapper** | 0s | 0s | Không đổi |
| **4. Thực thi JUnit Test (`./gradlew test`)** | **33s** (Tải Gradle + Dependencies) | **16s** (Dùng dependencies từ Cache) | **Giảm 17s (~51.5%)** 🚀 |
| **5. Đóng gói JAR (`./gradlew bootJar`)** | 0s | 1s | Tức thì |
| **6. Lưu trữ JAR Artifact** | 2s | 2s | Không đổi |
| **7. Post Step (Lưu Cache)** | 4s (Nén & Upload Cache lên storage) | 0s (Cache đã tồn tại, bỏ qua) | **Giảm 4s** |
| **Tổng thời gian chạy Job** | **~43 giây** | **~28 giây** | **Nhanh hơn ~35% toàn Job** |

---

## 5. Bằng chứng trích xuất từ Log thực tế của GitHub Actions

### 5.1. Log Lần 1 (Push 1 - Run #1): Cache Miss & Cache Save
Trích đoạn log từ bước `Thiết lập môi trường Java JDK 17` và `Post Thiết lập môi trường Java JDK 17`:

```text
2026-10-14T14:54:11.5827398Z   cache: gradle
2026-10-14T14:54:11.6914449Z Resolved Java 17.0.20+1 from tool-cache
2026-10-14T14:54:11.8749414Z gradle cache is not found
...
2026-10-14T14:54:12.3330093Z Downloading https://services.gradle.org/distributions/gradle-9.8.1-bin.zip
...
2026-10-14T14:54:49.6626099Z gradle-wrapper cache saved with the key: setup-java-Linux-x64-gradle-wrapper-dd4f180b86ca141440ef1cf694b931590248d062288eb72db65e47efd1a90839
2026-10-14T14:54:51.3937812Z Cache saved with the key: setup-java-Linux-x64-gradle-a6cb0cc429622b53bfbffe4b1e9eb178df28e1b57855db2edfb92f7a1e161f19
```

### 5.2. Log Lần 2 (Push 2 - Run #2): Cache Hit & Cache Restored Successfully
Trích đoạn log từ bước `Thiết lập môi trường Java JDK 17`:

```text
2026-10-14T14:57:56.5580458Z   cache: gradle
2026-10-14T14:57:56.6905000Z Resolved Java 17.0.20+1 from tool-cache
2026-10-14T14:57:56.9009094Z Cache hit for: setup-java-Linux-x64-gradle-a6cb0cc429622b53bfbffe4b1e9eb178df28e1b57855db2edfb92f7a1e161f19
2026-10-14T14:57:57.8402163Z Cache Size: ~40 MB (41577800 B)
2026-10-14T14:57:58.0879640Z Cache restored successfully
2026-10-14T14:57:58.0900810Z Cache restored from key: setup-java-Linux-x64-gradle-a6cb0cc429622b53bfbffe4b1e9eb178df28e1b57855db2edfb92f7a1e161f19
2026-10-14T14:57:58.2363359Z Cache hit for: setup-java-Linux-x64-gradle-wrapper-dd4f180b86ca141440ef1cf694b931590248d062288eb72db65e47efd1a90839
2026-10-14T14:58:02.5599059Z Cache Size: ~144 MB (150852740 B)
2026-10-14T14:58:02.7419977Z Cache restored successfully
2026-10-14T14:58:02.7484304Z gradle-wrapper cache restored from key: setup-java-Linux-x64-gradle-wrapper-dd4f180b86ca141440ef1cf694b931590248d062288eb72db65e47efd1a90839
```

---

## 6. Đánh giá và Kết luận

1. **Hiệu quả rõ rệt:** 
   - Bước `./gradlew test` giảm từ **33 giây xuống còn 16 giây** (nhanh hơn gấp đôi).
   - Tổng thời gian thực thi toàn bộ pipeline giảm từ **43 giây xuống còn 28 giây**, tiết kiệm đáng kể tài nguyên runner phút của GitHub Actions.
2. **Khả năng mở rộng:** Với các dự án thực tế trong doanh nghiệp có hàng trăm dependencies và nhiều microservices, Gradle Dependency Caching có thể tiết kiệm từ vài phút đến hàng chục phút cho mỗi lần chạy CI, giảm tình trạng nghẽn hàng đợi CI và tăng tốc độ triển khai phần mềm liên tục.
