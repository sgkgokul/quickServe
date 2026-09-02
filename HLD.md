# QuickServe — High-Level Design (HLD)

> **Version:** 1.0
> **Last Updated:** 2026-04-10
> **Status:** Finalized

---

## 1. Architecture Style

**Modular Monolith** — a single Spring Boot application organized into well-separated modules (packages). This approach gives us:

- All Kafka learning benefits (producers, consumers, topics, event-driven patterns).
- No deployment overhead of managing multiple microservices.
- Easier debugging and development in a single codebase.
- Can be split into microservices later if the project scales.

---

## 2. Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Backend Framework** | Spring Boot 3 (Java 17+) | REST APIs, business logic, Kafka integration |
| **Event Broker** | Apache Kafka | Async event processing for bookings and notifications |
| **Primary Database** | PostgreSQL + PostGIS | Relational data storage + geo-spatial queries |
| **Cache** | Redis | Provider online/offline status, geo lookups cache |
| **Authentication** | Spring Security + JWT | Stateless auth, role-based access |
| **API Documentation** | SpringDoc OpenAPI (Swagger UI) | Auto-generated interactive API docs |
| **Frontend** | React (Vite) | Responsive single-page application |
| **Containerization** | Docker + Docker Compose | Local dev environment (Kafka, Postgres, Redis) |
| **Build Tool** | Maven | Dependency management and build |

---

## 3. Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    FRONTEND (React SPA)                      │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────────┐  │
│  │ Customer Views│  │Provider Views │  │ Role Switch Toggle│  │
│  └──────────────┘  └───────────────┘  └──────────────────┘  │
└──────────────────────────┬──────────────────────────────────┘
                           │ REST API (HTTP/JSON)
┌──────────────────────────▼──────────────────────────────────┐
│              SPRING BOOT APPLICATION (Monolith)              │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌───────────┐  │
│  │   Auth   │  │   User   │  │  Catalog  │  │  Search   │  │
│  │  Module  │  │  Module  │  │  Module   │  │  Module   │  │
│  └──────────┘  └──────────┘  └───────────┘  └───────────┘  │
│                                                              │
│  ┌──────────┐  ┌──────────────┐  ┌──────────┐              │
│  │ Booking  │  │ Notification │  │  Rating  │              │
│  │  Module  │  │    Module    │  │  Module  │              │
│  └────┬─────┘  └──────▲───────┘  └──────────┘              │
│       │               │                                      │
└───────┼───────────────┼──────────────────────────────────────┘
        │  publish      │ consume
┌───────▼───────────────┼──────────────────────────────────────┐
│                  APACHE KAFKA                                │
│  ┌──────────────────┐  ┌──────────────────────────────────┐  │
│  │  booking-events  │  │  provider-availability-events    │  │
│  └──────────────────┘  └──────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
        │                        │
┌───────▼────────┐      ┌───────▼────────┐
│   PostgreSQL   │      │     Redis      │
│   + PostGIS    │      │   (Cache)      │
└────────────────┘      └────────────────┘
```

---

## 4. Module Breakdown

### 4.1 Auth Module
- **Responsibility:** User registration (email + password), login, JWT token generation and validation.
- **Key APIs:** `POST /api/auth/register`, `POST /api/auth/login`, `POST /api/auth/refresh`
- **Details:** Issues JWT with user ID and current active role. Role switch triggers a new token with updated role context.

### 4.2 User Module
- **Responsibility:** User profile management, provider profile setup, role switching.
- **Key APIs:** `GET/PUT /api/users/profile`, `POST /api/users/become-provider`, `POST /api/users/switch-role`
- **Kafka:** Publishes to `provider-availability-events` when a provider goes online/offline.

### 4.3 Catalog Module
- **Responsibility:** Service categories and sub-services CRUD. Admin-managed.
- **Key APIs:** `GET /api/categories`, `GET /api/categories/{id}/sub-services`
- **Seed Data:** Pre-loaded with 5 MVP categories and their sub-services.

### 4.4 Search Module
- **Responsibility:** Find available providers near a customer's location, filtered by category.
- **Key APIs:** `GET /api/search/providers?category={id}&lat={lat}&lng={lng}&radius={km}`
- **How it works:**
  1. Checks Redis for providers currently marked as ONLINE.
  2. Queries PostgreSQL + PostGIS for providers within the radius offering that category.
  3. Joins with rating data, sorts by proximity + rating.

### 4.5 Booking Module
- **Responsibility:** Booking creation, state transitions, booking history.
- **Key APIs:**
  - `POST /api/bookings` — Customer creates a booking.
  - `PUT /api/bookings/{id}/accept` — Provider accepts.
  - `PUT /api/bookings/{id}/decline` — Provider declines.
  - `PUT /api/bookings/{id}/start` — Provider starts work.
  - `PUT /api/bookings/{id}/complete` — Provider completes work.
  - `GET /api/bookings` — List bookings (filtered by role, status).
  - `GET /api/bookings/{id}` — Booking details.
- **Kafka:** Publishes to `booking-events` on every state change.

### 4.6 Notification Module
- **Responsibility:** In-app notifications — create, store, list, mark as read.
- **Key APIs:**
  - `GET /api/notifications` — List user's notifications (paginated).
  - `GET /api/notifications/unread-count` — Badge count.
  - `PUT /api/notifications/{id}/read` — Mark as read.
- **Kafka:** Consumes `booking-events` → creates appropriate notifications for the relevant user.

### 4.7 Rating Module
- **Responsibility:** Customer rates a provider after booking is completed.
- **Key APIs:**
  - `POST /api/bookings/{id}/rating` — Submit rating (1–5 stars + review text).
  - `GET /api/providers/{id}/ratings` — List reviews for a provider.
- **Kafka:** Publishes to `booking-events` (BOOKING_RATED event) → triggers notification to provider.

---

## 5. Kafka Design

### 5.1 Topics

| Topic | Partitions | Purpose |
|-------|-----------|---------|
| `booking-events` | 3 | All booking state transitions |
| `provider-availability-events` | 3 | Provider goes online/offline |

### 5.2 Event Schemas

**booking-events:**
```json
{
  "eventId": "uuid",
  "eventType": "BOOKING_CREATED | BOOKING_ACCEPTED | BOOKING_DECLINED | BOOKING_STARTED | BOOKING_COMPLETED | BOOKING_RATED",
  "bookingId": "uuid",
  "customerId": "uuid",
  "providerId": "uuid",
  "categoryId": "uuid",
  "timestamp": "ISO-8601",
  "metadata": {}
}
```

**provider-availability-events:**
```json
{
  "eventId": "uuid",
  "eventType": "PROVIDER_ONLINE | PROVIDER_OFFLINE",
  "providerId": "uuid",
  "latitude": 12.9716,
  "longitude": 77.5946,
  "timestamp": "ISO-8601"
}
```

### 5.3 Producer → Consumer Mapping

```
Booking Module ──publishes──► booking-events ──consumed by──► Notification Module
                                                               (creates in-app notifications)

User Module ──publishes──► provider-availability-events ──consumed by──► Search Module
                                                                         (updates Redis cache)
```

### 5.4 Why Kafka Here (Learning Value)

| Scenario | Without Kafka | With Kafka |
|----------|--------------|------------|
| Booking state change | Booking service directly calls notification service (tight coupling) | Booking publishes event, notification consumes independently (loose coupling) |
| Provider goes online | Search module queries DB every time | Event updates Redis in real-time, search reads from cache |
| Adding a new consumer | Modify booking service code | Just add a new consumer — zero changes to producer |

This teaches: **event-driven architecture, producer-consumer pattern, topic design, serialization, consumer groups, and decoupled modules.**

---

## 6. Data Flow Diagrams

### 6.1 Customer Books a Provider

```
Customer Browser                Spring Boot API              Kafka                  PostgreSQL    Redis
      │                              │                         │                        │          │
      │── POST /api/bookings ───────►│                         │                        │          │
      │                              │── Save booking ────────►│(DB)                    │          │
      │                              │── Publish ─────────────►│ booking-events         │          │
      │◄── 201 Created ─────────────│                         │                        │          │
      │                              │                         │                        │          │
      │                              │◄─ Consume event ───────│                        │          │
      │                              │── Save notification ───►│(DB)                    │          │
      │                              │                         │                        │          │
```

### 6.2 Provider Goes Online

```
Provider Browser               Spring Boot API              Kafka                  PostgreSQL    Redis
      │                              │                         │                        │          │
      │── POST /availability ───────►│                         │                        │          │
      │                              │── Update status ───────►│(DB)                    │          │
      │                              │── Publish ─────────────►│ provider-avail-events  │          │
      │◄── 200 OK ──────────────────│                         │                        │          │
      │                              │                         │                        │          │
      │                              │◄─ Consume event ───────│                        │          │
      │                              │── Cache online status ──┼────────────────────────┼────────►│
      │                              │                         │                        │          │
```

### 6.3 Customer Searches Providers

```
Customer Browser               Spring Boot API              PostgreSQL              Redis
      │                              │                         │                       │
      │── GET /search/providers ────►│                         │                       │
      │                              │── Get online IDs ──────►│(Redis)                │
      │                              │◄─ Online provider IDs ──│                       │
      │                              │── Geo query ───────────►│(PostGIS)              │
      │                              │◄─ Providers + distance ─│                       │
      │◄── Provider list ───────────│                         │                       │
      │                              │                         │                       │
```

---

## 7. Spring Boot Package Structure

```
com.quickserve
├── QuickServeApplication.java
│
├── auth/
│   ├── controller/       AuthController.java
│   ├── service/          AuthService.java
│   ├── dto/              LoginRequest, RegisterRequest, AuthResponse
│   ├── security/         JwtTokenProvider, JwtAuthFilter, SecurityConfig
│   └── entity/           (uses User entity from user module)
│
├── user/
│   ├── controller/       UserController.java, ProviderController.java
│   ├── service/          UserService.java, ProviderService.java
│   ├── repository/       UserRepository.java, ProviderProfileRepository.java
│   ├── dto/              UserProfileDTO, ProviderProfileDTO, SwitchRoleRequest
│   └── entity/           User.java, ProviderProfile.java
│
├── catalog/
│   ├── controller/       CategoryController.java
│   ├── service/          CategoryService.java
│   ├── repository/       CategoryRepository.java, SubServiceRepository.java
│   ├── dto/              CategoryDTO, SubServiceDTO
│   └── entity/           ServiceCategory.java, SubService.java
│
├── search/
│   ├── controller/       SearchController.java
│   ├── service/          SearchService.java
│   ├── dto/              ProviderSearchResult, SearchFilter
│   └── kafka/            ProviderAvailabilityConsumer.java
│
├── booking/
│   ├── controller/       BookingController.java
│   ├── service/          BookingService.java
│   ├── repository/       BookingRepository.java
│   ├── dto/              CreateBookingRequest, BookingResponse, BookingStatusUpdate
│   ├── entity/           Booking.java, BookingStatus.java (enum)
│   └── kafka/            BookingEventProducer.java
│
├── notification/
│   ├── controller/       NotificationController.java
│   ├── service/          NotificationService.java
│   ├── repository/       NotificationRepository.java
│   ├── dto/              NotificationDTO
│   ├── entity/           Notification.java
│   └── kafka/            BookingEventConsumer.java
│
├── rating/
│   ├── controller/       RatingController.java
│   ├── service/          RatingService.java
│   ├── repository/       RatingRepository.java
│   ├── dto/              RatingRequest, RatingResponse
│   └── entity/           Rating.java
│
├── common/
│   ├── exception/        GlobalExceptionHandler, custom exceptions
│   ├── dto/              ApiResponse, PagedResponse
│   └── util/             GeoUtils.java
│
└── config/
    ├── KafkaConfig.java
    ├── RedisConfig.java
    ├── SwaggerConfig.java
    └── CorsConfig.java
```

---

## 8. Infrastructure (Local Development)

### 8.1 Docker Compose Services

| Service | Image | Port | Purpose |
|---------|-------|------|---------|
| PostgreSQL | postgres:16 + PostGIS | 5432 | Primary database |
| Redis | redis:7 | 6379 | Cache layer |
| Zookeeper | confluentinc/cp-zookeeper | 2181 | Kafka dependency |
| Kafka | confluentinc/cp-kafka | 9092 | Event broker |
| Kafka UI | provectuslabs/kafka-ui | 8090 | Visual Kafka topic browser (dev tool) |

### 8.2 Spring Boot Profiles

| Profile | Purpose |
|---------|---------|
| `dev` | Local development — connects to Docker containers |
| `test` | Embedded H2 + embedded Kafka for integration tests |
| `prod` | Production config (future) |

---

## 9. API Conventions

- **Base URL:** `/api/v1/`
- **Auth Header:** `Authorization: Bearer <jwt-token>`
- **Response Format:**
```json
{
  "success": true,
  "data": { },
  "message": "Operation successful",
  "timestamp": "ISO-8601"
}
```
- **Error Format:**
```json
{
  "success": false,
  "data": null,
  "message": "Validation failed",
  "errors": ["Field 'email' is required"],
  "timestamp": "ISO-8601"
}
```
- **Pagination:** `?page=0&size=20&sort=createdAt,desc`

---

## 10. Security Considerations

- Passwords hashed with **BCrypt**.
- JWT tokens with short expiry (1 hour) + refresh token flow.
- Role-based API access:
  - `/api/v1/bookings` (POST) → requires CUSTOMER role context.
  - `/api/v1/bookings/{id}/accept` → requires PROVIDER role context.
  - `/api/v1/admin/**` → requires ADMIN role.
- Input validation on all endpoints (Jakarta Validation annotations).
- CORS configured for frontend origin only.

