# 📅 Reserva - SaaS Booking & Scheduling Platform

![NestJS](https://img.shields.io/badge/nestjs-%23E0234E.svg?style=for-the-badge&logo=nestjs&logoColor=white)
![TypeScript](https://img.shields.io/badge/typescript-%23007ACC.svg?style=for-the-badge&logo=typescript&logoColor=white)
![Postgres](https://img.shields.io/badge/postgres-%23316192.svg?style=for-the-badge&logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/redis-%23DD0031.svg?style=for-the-badge&logo=redis&logoColor=white)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
![Prisma](https://img.shields.io/badge/Prisma-3982CE?style=for-the-badge&logo=Prisma&logoColor=white)

**Reserva** là một hệ thống backend mô phỏng nền tảng đặt lịch với khả năng xử lý đồng thời cao (tương tự Calendly hoặc Booking.com) được xây dựng với **NestJS Microservices**.

Dự án tập trung vào việc giải quyết các thách thức backend thực tế: **Tính nhất quán dữ liệu (Data Consistency)**, **Xử lý tranh chấp (Race Conditions)** khi đặt slot, và **Giao tiếp độ trễ thấp** giữa các service sử dụng gRPC.

---

## 📑 Mục lục

- [🌟 Tính năng chính](#-tính-năng-chính)
- [🏗 Kiến trúc hệ thống](#-kiến-trúc-hệ-thống)
- [💾 Database Design](#-database-design)
- [🚀 Getting Started](#-getting-started)
- [🔧 Giao tiếp gRPC](#-giao-tiếp-grpc)
- [⚡ Logic nghiệp vụ cốt lõi](#-logic-nghiệp-vụ-cốt-lõi)
- [🗺 Roadmap](#-roadmap)
- [🧪 Testing Strategy](#-testing-strategy)

---

## 🌟 Tính năng chính

- **Kiến trúc Microservices**: Cấu trúc Monorepo sử dụng NestJS, phân tách các service riêng biệt (Auth, Provider, Booking).
- **Giao tiếp hiệu năng cao**: Sử dụng **gRPC (Protobuf)** cho việc giao tiếp giữa các service thay vì REST.
- **Tính toán Slot động**: Thuật toán sinh ra các khung giờ trống dựa trên cấu hình provider, giờ mở cửa và booking hiện có.
- **Xử lý đồng thời**: Triển khai **Redis Distributed Lock (Redlock)** để ngăn chặn đặt trùng (Race Conditions) khi nhiều user đặt cùng slot.
- **Thiết kế Database vững chắc**: Sử dụng **PostgreSQL** với **Prisma ORM** để tương tác database type-safe.
- **Infrastructure mở rộng**: Môi trường Dockerized hoàn chỉnh (App, DB, Redis) sẵn sàng deploy.

---

## 🏗 Kiến trúc hệ thống

Hệ thống được chia thành các microservices sau:

| Service | Tech Stack | Mô tả |
| :--- | :--- | :--- |
| **API Gateway** | NestJS (HTTP) | Điểm vào cho clients. Xử lý routing, aggregation, và chuyển đổi REST-to-gRPC. |
| **Auth Service** | NestJS (gRPC) | Xử lý đăng ký User, Login, phát hành JWT, và RBAC. |
| **Provider Service** | NestJS (gRPC) | Quản lý danh mục dịch vụ, cấu hình cửa hàng, và giờ mở cửa. |
| **Booking Service** | NestJS (gRPC) | **Logic cốt lõi**. Xử lý tính toán slot, giữ chỗ, và cơ chế locking. |
| **Payment Service** | NestJS (gRPC/Event) | Mock thanh toán, cập nhật trạng thái đơn hàng. |

### Sơ đồ luồng dữ liệu

```
Client (Web/Mobile)
    ↓
API Gateway (REST/HTTP)
    ↓ (gRPC)
┌─────────────┬──────────────┬───────────────┐
│ Auth Service│Provider Svc  │Booking Service│
└─────────────┴──────────────┴───────────────┘
    ↓              ↓                ↓
PostgreSQL     PostgreSQL      PostgreSQL
                                    ↓
                                  Redis
```

---

## 💾 Database Design

### Auth DB

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  password  String   // Hashed with bcrypt
  role      String   // USER, PROVIDER, ADMIN
  name      String?
  createdAt DateTime @default(now())
}
```

### Provider DB

```prisma
model ProviderConfig {
  id          String   @id @default(uuid())
  providerId  Int      @unique // Link tới User.id bên Auth
  openTime    String   @default("08:00")
  closeTime   String   @default("22:00")
  timeStep    Int      @default(30) // Bước nhảy: 15p, 30p, 60p
}

model Service {
  id          String   @id @default(uuid())
  providerId  Int
  name        String
  duration    Int      // Thời lượng (phút)
  price       Decimal  @db.Decimal(10, 2)
  createdAt   DateTime @default(now())
  
  @@index([providerId])
}
```

### Booking DB

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
  updatedAt   DateTime @updatedAt
  
  @@index([providerId, startTime, endTime])
  @@index([userId])
}
```

---

## 🚀 Getting Started

### Prerequisites

- Node.js (v18+) & npm/pnpm
- Docker & Docker Compose

### Installation

1. **Clone repository**
   ```bash
   git clone https://github.com/yourusername/reserva.git
   cd reserva
   ```

2. **Start Infrastructure (Postgres & Redis)**
   ```bash
   docker-compose up -d
   ```

3. **Install Dependencies**
   ```bash
   npm install
   # hoặc
   pnpm install
   ```

4. **Setup Database (Prisma)**
   ```bash
   # Run migration cho từng service
   npx prisma migrate dev --name init
   npx prisma generate
   ```

5. **Run Services**
   ```bash
   # Development mode
   npm run start:dev
   
   # Hoặc chạy riêng từng service
   npm run start:dev -- auth
   npm run start:dev -- gateway
   ```

### Testing API

**Register User:**
```bash
curl -X POST http://localhost:3000/auth/register \
   -H 'Content-Type: application/json' \
   -d '{
     "email": "demo@reserva.com",
     "password": "123456",
     "name": "Demo User"
   }'
```

**Login:**
```bash
curl -X POST http://localhost:3000/auth/login \
   -H 'Content-Type: application/json' \
   -d '{
     "email": "demo@reserva.com",
     "password": "123456"
   }'
```

---

## 🔧 Giao tiếp gRPC

### auth.proto

```protobuf
service AuthService {
  rpc Register(RegisterRequest) returns (AuthResponse);
  rpc Login(LoginRequest) returns (AuthResponse);
  rpc ValidateToken(TokenRequest) returns (UserResponse);
}
```

### provider.proto

```protobuf
service ProviderService {
  rpc CreateService(CreateServiceRequest) returns (ServiceResponse);
  rpc GetProviderConfig(ProviderIdRequest) returns (ConfigResponse);
  rpc UpdateConfig(UpdateConfigRequest) returns (ConfigResponse);
}
```

### booking.proto

```protobuf
service BookingService {
  // API khó nhất - Tính toán slot trống
  rpc GetAvailableSlots(GetSlotsRequest) returns (SlotsResponse);
  
  // Tạo booking với locking mechanism
  rpc CreateBooking(CreateBookingRequest) returns (BookingResponse);
  
  rpc CancelBooking(CancelBookingRequest) returns (BookingResponse);
}
```

---

## ⚡ Logic nghiệp vụ cốt lõi

### 5.1. Thuật toán tính Slot trống (Get Available Slots)

**Input:** 
- Ngày: `2024-01-20`
- Dịch vụ: Cắt tóc (45 phút)
- Shop: Mở cửa 8h-22h, timeStep = 30p

**Các bước:**

1. **Tạo Time Grid**
   ```typescript
   // Từ 08:00 đến 22:00, bước nhảy 30p
   const timeGrid = ['08:00', '08:30', '09:00', ..., '21:30'];
   ```

2. **Lấy Bookings hiện có**
   ```sql
   SELECT * FROM bookings 
   WHERE providerId = ? 
   AND DATE(startTime) = '2024-01-20'
   ```

3. **Lọc slot available**
   ```typescript
   const availableSlots = timeGrid.filter(slot => {
     const slotEnd = addMinutes(slot, 45); // 45p
     // Kiểm tra không đè lên bất kỳ booking nào
     return !hasOverlap(slot, slotEnd, existingBookings);
   });
   ```

**Output:** `['08:00', '09:30', '11:00', ...]`

### 5.2. Xử lý Race Condition với Redis Lock

**Flow khi User đặt chỗ:**

```typescript
async createBooking(userId, providerId, startTime, serviceId) {
  const lockKey = `lock:provider:${providerId}:time:${startTime}`;
  
  // 1. Acquire Lock (10s TTL)
  const lock = await redis.set(lockKey, 'locked', 'EX', 10, 'NX');
  if (!lock) {
    throw new ConflictException('Slot vừa được đặt bởi người khác');
  }
  
  try {
    // 2. Double-check DB
    const existing = await db.booking.findFirst({
      where: {
        providerId,
        startTime: { lte: endTime },
        endTime: { gte: startTime }
      }
    });
    
    if (existing) {
      throw new ConflictException('Slot không còn trống');
    }
    
    // 3. Insert Booking
    const booking = await db.booking.create({
      data: { userId, providerId, serviceId, startTime, endTime }
    });
    
    return booking;
    
  } finally {
    // 4. Release Lock
    await redis.del(lockKey);
  }
}
```

---

## 🗺 Roadmap

### ✅ Phase 1: Foundation (Hoàn thành)

- [x] Setup Monorepo NestJS
- [x] Setup Docker (Postgres)
- [x] Auth Service (JWT, Hash Password)
- [x] API Gateway routing

### 🚧 Phase 2: Provider & Service Catalog (Đang làm)

- [ ] Provider Service với gRPC
- [ ] API tạo Config (Giờ mở cửa)
- [ ] API quản lý Services (CRUD)
- [ ] Seed data mẫu

### ⚡ Phase 3: Booking Engine (Core)

- [ ] Booking Service
- [ ] Thuật toán tính Slot (Logic 5.1)
- [ ] API `POST /bookings` (chưa có Lock)

### 🔥 Phase 4: Advanced Features

- [ ] Redis Distributed Lock (Redlock)
- [ ] Stress Test Script (k6/Artillery)
- [ ] Payment Mock Integration
- [ ] Notification Service (Email/SMS)

---

## 🧪 Testing Strategy

### Unit Test
```bash
npm run test
```
- Test logic tính toán slot
- Test validation rules
- Test business logic isolation

### Integration Test
```bash
npm run test:e2e
```
- Test flow: `Gateway → gRPC → DB`
- Test authentication flow
- Test booking flow end-to-end

### Load/Stress Test

**Mục tiêu:** Kiểm tra Race Condition handling

```bash
cd stress-test
node race-condition.js
```

**Scenario:**
- Bắn 100 requests đồng thời vào cùng 1 slot
- **Kỳ vọng:** Chỉ 1 booking được tạo thành công
- **Metric:** Latency, Success Rate, Error Distribution

---

## 📚 Learning Outcomes

Qua việc xây dựng Reserva, tôi đã học được:

- ✅ **Monorepo Management** với NestJS CLI
- ✅ **gRPC vs REST** - Performance trade-offs
- ✅ **Database Locking** - Optimistic vs Pessimistic
- ✅ **Distributed Systems** - CAP theorem trong thực tế
- ✅ **Event-Driven Architecture** concepts
- ✅ **High-Concurrency** patterns

---

## 📄 License

MIT License

## 👨‍💻 Author

**Hoang**

---

<div align="center">

**Happy Coding! 🚀**

Made with ❤️ using NestJS

</div>
