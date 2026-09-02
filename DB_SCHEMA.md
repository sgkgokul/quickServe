# QuickServe — Database Schema Design

> **Version:** 1.0
> **Last Updated:** 2026-04-10
> **Database:** PostgreSQL 16 + PostGIS

---

## 1. Tables Overview

| # | Table | Purpose | Key Relationships |
|---|-------|---------|-------------------|
| 1 | `users` | All registered users (customers, providers, admins) | Parent of provider_profiles, bookings, notifications |
| 2 | `provider_profiles` | Extra profile data for users who are providers | Belongs to users; parent of provider_services, bookings |
| 3 | `service_categories` | Top-level service categories (5 MVP) | Parent of sub_services |
| 4 | `sub_services` | Sub-services under each category | Belongs to service_categories |
| 5 | `provider_services` | Which sub-services a provider offers + their rate | Links provider_profiles ↔ sub_services |
| 6 | `bookings` | Core booking records with full lifecycle | Links customer (user) ↔ provider ↔ sub_service |
| 7 | `ratings` | Customer reviews after completed bookings | Belongs to bookings, customer (user), provider |
| 8 | `notifications` | In-app notification records | Belongs to users; optionally linked to bookings |

---

## 2. Table Definitions

### 2.1 `users`

All users in the system. A user can be a customer, a provider, or both.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK, DEFAULT gen_random_uuid() | Primary key |
| `name` | VARCHAR(100) | NOT NULL | Display name |
| `email` | VARCHAR(255) | NOT NULL, UNIQUE | Login email |
| `password_hash` | VARCHAR(255) | NOT NULL | BCrypt hashed password |
| `profile_photo_url` | VARCHAR(500) | NULLABLE | Profile image URL |
| `active_role` | VARCHAR(20) | NOT NULL, DEFAULT 'CUSTOMER' | Current active mode: CUSTOMER / PROVIDER / ADMIN |
| `is_provider` | BOOLEAN | NOT NULL, DEFAULT FALSE | Whether user has completed provider onboarding |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Account creation time |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Last profile update |

**Indexes:**
- `idx_users_email` — UNIQUE index on `email` (for login lookup)

---

### 2.2 `provider_profiles`

Created when a user opts to "Become a Provider". One-to-one with `users`.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | Primary key |
| `user_id` | UUID | FK → users.id, UNIQUE, NOT NULL | Owning user |
| `bio` | TEXT | NULLABLE | Short description of experience |
| `latitude` | DOUBLE PRECISION | NULLABLE | Provider's base location (lat) |
| `longitude` | DOUBLE PRECISION | NULLABLE | Provider's base location (lng) |
| `address` | VARCHAR(500) | NULLABLE | Text address |
| `working_radius_km` | INTEGER | NOT NULL, DEFAULT 10 | Max travel distance in km |
| `is_online` | BOOLEAN | NOT NULL, DEFAULT FALSE | Currently accepting jobs? |
| `avg_rating` | DECIMAL(3,2) | NOT NULL, DEFAULT 0.00 | Denormalized average rating (1.00–5.00) |
| `total_reviews` | INTEGER | NOT NULL, DEFAULT 0 | Denormalized review count |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Profile creation time |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Last update |

**Indexes:**
- `idx_provider_user_id` — UNIQUE index on `user_id`
- `idx_provider_location` — PostGIS spatial index on (`latitude`, `longitude`) for geo queries
- `idx_provider_online` — Index on `is_online` (filter online providers)

**Why denormalize `avg_rating` and `total_reviews`?**
- Avoids expensive JOIN + AVG query every time a provider appears in search results.
- Updated via application logic whenever a new rating is submitted.

---

### 2.3 `service_categories`

Top-level service types. Admin-managed. Pre-seeded with 5 MVP categories.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | Primary key |
| `name` | VARCHAR(100) | NOT NULL, UNIQUE | Category name (e.g., "Plumber") |
| `icon` | VARCHAR(100) | NULLABLE | Icon identifier for frontend |
| `description` | TEXT | NULLABLE | Category description |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | Soft delete / disable |
| `display_order` | INTEGER | NOT NULL, DEFAULT 0 | Ordering on frontend |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Creation time |

---

### 2.4 `sub_services`

Specific services under a category. E.g., "Leak repair" under "Plumber".

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | Primary key |
| `category_id` | UUID | FK → service_categories.id, NOT NULL | Parent category |
| `name` | VARCHAR(100) | NOT NULL | Sub-service name |
| `description` | TEXT | NULLABLE | Details |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | Soft delete |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Creation time |

**Indexes:**
- `idx_sub_services_category` — Index on `category_id`
- UNIQUE constraint on (`category_id`, `name`) — no duplicate names within a category

---

### 2.5 `provider_services`

Junction table: which provider offers which sub-services and at what rate.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | Primary key |
| `provider_profile_id` | UUID | FK → provider_profiles.id, NOT NULL | The provider |
| `sub_service_id` | UUID | FK → sub_services.id, NOT NULL | The sub-service |
| `rate_per_hour` | DECIMAL(10,2) | NOT NULL | Provider's rate for this service |
| `rate_type` | VARCHAR(20) | NOT NULL, DEFAULT 'PER_HOUR' | PER_HOUR / FIXED |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT TRUE | Provider can toggle services on/off |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Creation time |

**Indexes:**
- UNIQUE constraint on (`provider_profile_id`, `sub_service_id`) — provider can't list same service twice
- `idx_provider_services_sub` — Index on `sub_service_id` (search by service)

---

### 2.6 `bookings`

Core booking table. Tracks the full lifecycle from request to completion.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | Primary key |
| `customer_id` | UUID | FK → users.id, NOT NULL | Customer who booked |
| `provider_id` | UUID | FK → provider_profiles.id, NOT NULL | Provider who is booked |
| `sub_service_id` | UUID | FK → sub_services.id, NOT NULL | What service |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT 'REQUESTED' | REQUESTED / ACCEPTED / DECLINED / IN_PROGRESS / COMPLETED / RATED |
| `description` | TEXT | NULLABLE | Special instructions from customer |
| `estimated_cost` | DECIMAL(10,2) | NULLABLE | rate × estimated hours |
| `estimated_duration_hrs` | INTEGER | NULLABLE | Expected duration in hours |
| `customer_address` | VARCHAR(500) | NULLABLE | Service location (text) |
| `customer_lat` | DOUBLE PRECISION | NULLABLE | Service location (lat) |
| `customer_lng` | DOUBLE PRECISION | NULLABLE | Service location (lng) |
| `booked_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | When customer created booking |
| `accepted_at` | TIMESTAMP | NULLABLE | When provider accepted |
| `started_at` | TIMESTAMP | NULLABLE | When provider started work |
| `completed_at` | TIMESTAMP | NULLABLE | When provider completed work |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Record creation |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | Last update |

**Indexes:**
- `idx_bookings_customer` — Index on `customer_id` (customer's booking history)
- `idx_bookings_provider` — Index on `provider_id` (provider's booking history)
- `idx_bookings_status` — Index on `status` (filter by state)
- `idx_bookings_created` — Index on `created_at DESC` (recent bookings first)

---

### 2.7 `ratings`

One rating per completed booking. Customer rates the provider.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | Primary key |
| `booking_id` | UUID | FK → bookings.id, UNIQUE, NOT NULL | One rating per booking |
| `customer_id` | UUID | FK → users.id, NOT NULL | Who gave the rating |
| `provider_id` | UUID | FK → provider_profiles.id, NOT NULL | Who received the rating |
| `stars` | INTEGER | NOT NULL, CHECK (1–5) | Star rating |
| `review_text` | TEXT | NULLABLE | Written review |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | When rating was submitted |

**Indexes:**
- `idx_ratings_booking` — UNIQUE index on `booking_id`
- `idx_ratings_provider` — Index on `provider_id` (list provider's reviews)

---

### 2.8 `notifications`

In-app notifications stored in DB. Fetched via polling from frontend.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK | Primary key |
| `user_id` | UUID | FK → users.id, NOT NULL | Recipient |
| `booking_id` | UUID | FK → bookings.id, NULLABLE | Related booking (if any) |
| `type` | VARCHAR(50) | NOT NULL | BOOKING_CREATED / BOOKING_ACCEPTED / BOOKING_DECLINED / BOOKING_STARTED / BOOKING_COMPLETED / BOOKING_RATED |
| `title` | VARCHAR(200) | NOT NULL | Notification headline |
| `message` | TEXT | NOT NULL | Full notification message |
| `is_read` | BOOLEAN | NOT NULL, DEFAULT FALSE | Read/unread state |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT NOW() | When notification was created |

**Indexes:**
- `idx_notifications_user_unread` — Composite index on (`user_id`, `is_read`) for unread count badge
- `idx_notifications_user_created` — Composite index on (`user_id`, `created_at DESC`) for listing

---

## 3. Relationships Summary

```
users (1) ──────────── (0..1) provider_profiles
users (1) ──────────── (0..*) bookings (as customer)
users (1) ──────────── (0..*) notifications
users (1) ──────────── (0..*) ratings (as customer)

provider_profiles (1) ── (0..*) provider_services
provider_profiles (1) ── (0..*) bookings (as provider)
provider_profiles (1) ── (0..*) ratings (as provider)

service_categories (1) ── (1..*) sub_services

sub_services (1) ──────── (0..*) provider_services
sub_services (1) ──────── (0..*) bookings

bookings (1) ──────────── (0..1) ratings
bookings (1) ──────────── (0..*) notifications
```

---

## 4. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| UUID primary keys | Globally unique, safe for distributed systems, no sequential ID guessing |
| Denormalized `avg_rating` on provider_profiles | Avoids expensive aggregate query on every search result |
| Separate `provider_profiles` table | Not all users are providers; keeps `users` table lean |
| `provider_services` junction table | Providers can offer multiple services at different rates |
| `status` as VARCHAR (not FK to lookup) | Simple enum mapping in Java; only 6 known states |
| Timestamps on booking state changes | Track SLA metrics later (time to accept, time to complete) |
| Soft delete via `is_active` flags | Categories/services can be disabled without losing data |

---

## 5. Seed Data (MVP)

Pre-loaded on application startup:

| Category | Sub-services |
|----------|-------------|
| Acting Driver | Personal Car Driver, Outstation Trip Driver |
| Cook | Single Meal, Full Day Cook, Party/Event Cook |
| Cleaner | Home Deep Clean, Regular Cleaning, Post-Event Cleanup |
| Plumber | Leak Repair, New Fitting, Drainage |
| Electrician | Wiring, Appliance Installation, Emergency Repair |

