# Bài 1 — Chuẩn hóa dependency Spring Cloud cho hệ thống nhiều service

**Cấp độ:** Vận dụng cơ bản  
**Session 03 — Từ Monolithic đến Microservice**  
**Hệ thống:** FoodX (restaurant-service, order-service, delivery-service)

---

## 1. Mục tiêu kiến thức

Hiểu và áp dụng đúng cách khai báo **Spring Cloud BOM** để đồng bộ phiên bản giữa các service, tránh xung đột thư viện khi tích hợp thêm các module Spring Cloud.

---

## 2. Phân tích vấn đề trong build.gradle gốc

### Code gốc (có vấn đề)

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    // Khai báo trực tiếp version cho từng thư viện Spring Cloud,
    // không dùng BOM để quản lý tập trung
    implementation 'org.springframework.cloud:spring-cloud-starter-config:3.1.4'
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client:3.1.5'
}

// Thiếu hoàn toàn khối dependencyManagement khai báo Spring Cloud BOM
```

### Vấn đề của việc khai báo version cố định, rời rạc

1. **Phiên bản không đồng bộ**  
   `spring-cloud-starter-config:3.1.4` và `spring-cloud-starter-netflix-eureka-client:3.1.5` thuộc **hai release khác nhau**. Chúng có thể phụ thuộc các thư viện transitive với phiên bản xung đột (Jackson, Spring Framework, Netty…).

2. **Khó mở rộng**  
   Khi thêm starter mới (Gateway, OpenFeign, Circuit Breaker, Config Server…), lập trình viên phải **tự tìm đúng version tương thích**. Dễ chọn nhầm → build fail hoặc runtime error.

3. **Không đảm bảo tương thích với Spring Boot**  
   Mỗi train Spring Cloud (2021.0.x, 2022.0.x, 2023.0.x…) chỉ tương thích với một dải Spring Boot cụ thể. Khai báo version thủ công dễ lệch train.

4. **Khó bảo trì nhiều service**  
   restaurant-service, order-service, delivery-service nếu mỗi nơi hard-code version khác nhau → hệ thống không thống nhất, khó upgrade đồng loạt.

**Rủi ro khi dự án phát triển thêm:**  
Xung đột classpath, `NoSuchMethodError`, `ClassNotFoundException`, hoặc hành vi khác nhau giữa các service dù dùng cùng starter.

---

## 3. Cách sửa đúng — Dùng Spring Cloud BOM

### Nguyên tắc

- Khai báo **một lần** Spring Cloud BOM trong `dependencyManagement`.
- Các dependency Spring Cloud **không ghi version** → BOM tự quản lý.
- Cả 3 service dùng **cùng một train** Spring Cloud → đồng bộ tuyệt đối.

### Mapping phiên bản được chọn (ổn định cho bài học)

| Thành phần              | Phiên bản      |
|-------------------------|----------------|
| Spring Boot             | 3.2.5          |
| Spring Cloud Release Train | 2023.0.1 (Leyton) |

> Spring Cloud `2023.0.x` tương thích với Spring Boot `3.2.x` và `3.3.x`.

### File build.gradle đã sửa (áp dụng cho cả 3 service)

Xem chi tiết trong các thư mục:
- [`restaurant-service/build.gradle`](restaurant-service/build.gradle)
- [`order-service/build.gradle`](order-service/build.gradle)
- [`delivery-service/build.gradle`](delivery-service/build.gradle)

**Cấu trúc chung:**

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.5'
    id 'io.spring.dependency-management' version '1.1.4'
}

group = 'com.foodx'
version = '1.0.0'

java {
    sourceCompatibility = '17'
}

repositories {
    mavenCentral()
}

ext {
    set('springCloudVersion', "2023.0.1")
}

dependencyManagement {
    imports {
        // Spring Cloud BOM — quản lý tập trung toàn bộ version
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'

    // KHÔNG còn khai báo version cố định
    implementation 'org.springframework.cloud:spring-cloud-starter-config'
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

**Thay đổi chính:**
- Thêm khối `dependencyManagement` import `spring-cloud-dependencies`.
- Xóa version `3.1.4`, `3.1.5` khỏi từng dependency.
- Dùng biến `springCloudVersion` để dễ nâng cấp sau này.

---

## 4. Áp dụng cho cả 3 service

| Service              | Thư mục                | Trạng thái                          |
|----------------------|------------------------|-------------------------------------|
| restaurant-service   | `restaurant-service/`  | Đã dùng Spring Cloud BOM 2023.0.1   |
| order-service        | `order-service/`       | Đã dùng Spring Cloud BOM 2023.0.1   |
| delivery-service     | `delivery-service/`    | Đã dùng Spring Cloud BOM 2023.0.1   |

Cả 3 service giờ **cùng phiên bản** Spring Cloud → sẵn sàng tích hợp Config Server, Eureka Server ở các bài tiếp theo mà không lo xung đột.

---

## 5. Spring Cloud BOM giải quyết vấn đề gì? Vì sao cần thiết?

### BOM (Bill of Materials) là gì?

BOM là một file đặc biệt (pom) liệt kê **bộ phiên bản tương thích** của hàng chục thư viện Spring Cloud. Khi import BOM, Gradle/Maven sẽ tự chọn đúng version cho mọi starter mà bạn khai báo **không cần ghi số version**.

### Lợi ích chính

| Lợi ích                        | Giải thích                                                                 |
|--------------------------------|----------------------------------------------------------------------------|
| **Đồng bộ phiên bản**          | Tất cả starter (Config, Eureka, Gateway, OpenFeign…) dùng chung một train |
| **Tránh xung đột transitive**  | Các dependency phụ thuộc bên trong đã được kiểm tra tương thích           |
| **Dễ nâng cấp**                | Chỉ cần đổi một dòng `springCloudVersion` là nâng toàn bộ hệ thống        |
| **Chuẩn bị cho bài sau**       | Config Server, Eureka Server/Client, Gateway… đều yêu cầu cùng train      |
| **Giảm lỗi build/runtime**     | Không còn `NoSuchMethodError` do version lệch                             |

**Kết luận:**  
Trước khi tích hợp bất kỳ module Spring Cloud nào (Config Server, Eureka, Gateway…), **bắt buộc** phải khai báo Spring Cloud BOM. Đây là bước nền tảng để hệ thống nhiều service hoạt động ổn định và dễ bảo trì.

---

## 6. Cấu trúc repository

```
HN_KS24_CNTT4_IT214_Session03_Bai01/
├── README.md
├── restaurant-service/
│   └── build.gradle
├── order-service/
│   └── build.gradle
└── delivery-service/
    └── build.gradle
```
