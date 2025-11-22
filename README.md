# 📘 PROJECT RESERVA: TECHNICAL DESIGN DOCUMENT

**Mô tả:** Hệ thống SaaS đặt lịch dịch vụ & quản lý thời gian thực (Booking & Scheduling Platform).

**Kiến trúc:** Microservices, Event-Driven.

**Tech Stack:** NestJS, gRPC, Prisma, PostgreSQL, Redis, Docker.

---

## 📑 Mục lục

1. [Tổng quan (Overview)](#1-tổng-quan-overview)
2. [Kiến trúc hệ thống (System Architecture)](#2-kiến-trúc-hệ-thống-system-architecture)
3. [Database Design (Prisma Schema)](#3-database-design-prisma-schema)
4. [Giao tiếp gRPC (Proto Contracts)](#4-giao-tiếp-grpc-proto-contracts)
5. [Logic nghiệp vụ cốt lõi (Core Logic)](#5-logic-nghiệp-vụ-cốt-lõi-core-logic)
6. [Kế hoạch triển khai (Roadmap)](#6-kế-hoạch-triển-khai-roadmap)
7. [Chiến lược Testing (Testing Strategy)](#7-chiến-lược-testing-testing-strategy)

---

## 1. TỔNG QUAN (OVERVIEW)

### 1.1. Bài toán kinh doanh

Xây dựng nền tảng cho phép các **Nhà cung cấp (Providers)** (Salon tóc, Sân cầu lông, Phòng khám...) tạo dịch vụ và quản lý lịch làm việc. **Người dùng (Users)** có thể tìm kiếm, xem khung giờ trống (Slots) và đặt lịch (Booking) theo thời gian thực.

### 1.2. Mục tiêu kỹ thuật (Learning Goals)

- **Microservices Communication:** Thành thạo gRPC (Protobuf) giữa các service.
- **High Concurrency:** Xử lý tranh chấp (Race Condition) khi nhiều người đặt cùng 1 slot (Redis Lock / DB Isolation).
- **Data Consistency:** Đảm bảo tính nhất quán dữ liệu phân tán (Saga Pattern / Two-phase commit - ở mức đơn giản).
- **Complex Logic:** Thuật toán tính toán khung giờ trống (Time Slot Calculation).

---

## 2. KIẾN TRÚC HỆ THỐNG (SYSTEM ARCHITECTURE)

**Mô hình:** Monorepo (Nx hoặc NestJS Workspace).

### 2.1. Sơ đồ các Service

#### **API Gateway (HTTP/REST)**
- Cổng duy nhất nhận request từ Client.
- Xác thực Token (JWT).
- Điều hướng request sang gRPC client tương ứng.

#### **Auth Service (gRPC)**
- Quản lý User, Phân quyền (Admin/Provider/User).
- DB: `reserva_auth` (Postgres).

#### **Provider Service (gRPC)**
- Quản lý thông tin cửa hàng, giờ mở cửa (Config), danh sách dịch vụ.
- DB: `reserva_provider` (Postgres).

#### **Booking Service (gRPC - Core)**
- Tính toán Slot trống.
- Tạo Booking, xử lý giữ chỗ (Hold).
- DB: `reserva_booking` (Postgres).

#### **Payment Service (gRPC/Event)**
- Mock thanh toán, cập nhật trạng thái đơn hàng.

### 2.2. Infrastructure

- **Docker Compose:** Orchestration.
- **PostgreSQL:** Database chính (mỗi service 1 DB logic hoặc schema riêng).
- **Redis:** Caching (lưu Slot, Token) & Distributed Lock.

---

## 3. DATABASE DESIGN (PRISMA SCHEMA)

Dù là Microservices, ta hình dung cấu trúc dữ liệu tổng thể như sau:

### 3.1. Auth DB

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  password  String   // Hash
  role      String   // USER, PROVIDER, ADMIN
}
```

### 3.2. Provider DB

```prisma
model ProviderConfig {
  id          String   @id @default(uuid())
  providerId  Int      // Link tới User.id bên Auth
  openTime    String   @default("08:00")
  closeTime   String   @default("22:00")
  timeStep    Int      @default(30) // Bước nhảy: 15p, 30p, 60p
}

model Service {
  id          String   @id @default(uuid())
  providerId  Int
  name        String
  duration    Int      // Thời lượng (phút)
  price       Decimal
}
```

### 3.3. Booking DB

```prisma
model Booking {
  id          String   @id @default(uuid())
  userId      Int      // Người đặt
  providerId  Int      // Chủ sân/shop
  serviceId   String
  startTime   DateTime
  endTime     DateTime
  status      String   // PENDING, CONFIRMED, CANCELLED
  createdAt   DateTime @default(now())
  
  @@index([providerId, startTime, endTime]) // Index quan trọng để query nhanh
}
```

---

## 4. GIAO TIẾP GRPC (PROTO CONTRACTS)

### 4.1. auth.proto

```protobuf
rpc Register(RegisterReq) returns (RegisterRes)
rpc Login(LoginReq) returns (LoginRes)
rpc ValidateToken(TokenReq) returns (UserRes)
```

### 4.2. provider.proto

```protobuf
rpc CreateService(CreateServiceReq) returns (ServiceRes)
rpc GetProviderConfig(ProviderIdReq) returns (ConfigRes)
```

### 4.3. booking.proto

```protobuf
rpc GetAvailableSlots(GetSlotsReq) returns (SlotsRes)  // API Khó nhất
rpc CreateBooking(CreateBookingReq) returns (BookingRes)
```

**Chi tiết:**
- `GetAvailableSlots`: Input: Ngày, ServiceID. Output: List giờ có thể đặt.
- `CreateBooking`: Input: UserID, Time, ServiceID.

---

## 5. LOGIC NGHIỆP VỤ CỐT LÕI (CORE LOGIC)

### 5.1. Logic tạo Slot (Get Available Slots)

**Input:** Ngày 20/10, Dịch vụ cắt tóc (45 phút), Shop mở 8h-22h.

**Các bước:**

1. **Lấy config giờ mở cửa** → Tạo mảng các mốc thời gian (Time Grid) dựa trên `timeStep` (ví dụ 30p):
   ```
   [08:00, 08:30, 09:00, ..., 21:30]
   ```

2. **Lấy danh sách Booking đã tồn tại** trong ngày 20/10 từ DB.

3. **Duyệt từng mốc thời gian** trong Time Grid:
   - Nếu `(Mốc đó + 45p)` KHÔNG đè lên bất kỳ Booking nào
   - → Thêm vào danh sách Available.

**Output:** `['08:00', '09:30', ...]`

### 5.2. Logic Chống trùng lặp (Race Condition)

**Khi User bấm "Đặt chỗ":**

1. **Redis Lock:**
   - Tạo key: `lock:provider:{id}:time:{start_time}`
   - Nếu tồn tại → Trả lỗi "Vừa có người đặt"
   - Nếu chưa → Set key (expire 10s)

2. **Double Check DB:**
   - Query DB lần nữa xem khoảng thời gian đó có ai đặt chưa
   - (Đề phòng Redis mất data)

3. **Insert DB:**
   - Ghi Booking vào Postgres

4. **Release Lock:**
   - Xóa key Redis

---

## 6. KẾ HOẠCH TRIỂN KHAI (ROADMAP)

### Phase 1: The Foundation ✅ (Đã làm một phần)

- [x] Setup Monorepo (NestJS)
- [x] Setup Docker (Postgres)
- [x] Hoàn thiện Auth Service (JWT, Hash Password)
- [x] API Gateway forward request auth thành công

### Phase 2: Provider & Service Catalog 🚧

- [ ] Tạo `provider-service`
- [ ] Define `provider.proto`
- [ ] API cho phép Provider tạo Config (Giờ mở cửa) và Dịch vụ (Tên, Giá, Thời lượng)
- [ ] Seed data (Tạo dữ liệu mẫu)

### Phase 3: The Booking Engine ⚡ (Thách thức)

- [ ] Tạo `booking-service`
- [ ] Thuật toán tính toán Slot trống (Logic 5.1)
- [ ] API `POST /bookings`: Xử lý đặt lịch cơ bản (chưa cần Lock)

### Phase 4: Advanced Engineering 🔥 (Nâng cao)

- [ ] Tích hợp Redis vào Docker
- [ ] Cài đặt Redis Lock (Redlock) xử lý Race Condition
- [ ] Viết Script Stress Test (JMeter/k6/NodeJS Script) để test vụ tranh chấp slot
- [ ] Payment Mock integration

---

## 7. CHIẾN LƯỢC TESTING (TESTING STRATEGY)

### Unit Test
- Test logic tính toán Slot
- Đầu vào: Giờ này
- Đầu ra: Phải là list này

### Integration Test
- Test luồng: `Gateway → gRPC → DB`

### Load/Stress Test
- Dùng script bắn **100 request/giây** vào cùng 1 slot
- Kiểm chứng Redis Lock hoạt động tốt
- **Kết quả mong đợi:** Chỉ 1 booking được tạo

---

## 📝 License

UNLICENSED - Private project

## 👨‍💻 Author

Hoang

---

**Happy Coding! 🚀**
