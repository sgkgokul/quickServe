# QuickServe — Product Specification

> **Version:** 1.0
> **Last Updated:** 2026-04-10
> **Status:** Finalized (MVP Scope)

---

## 1. Overview

**QuickServe** is a responsive web application that connects people who need everyday household services with local skilled workers. It enables immediate, short-term bookings for tasks like plumbing, cooking, cleaning, driving, and electrical work within a geographic radius.

### 1.1 Problem Statement

Finding reliable, nearby gig workers for everyday household tasks is fragmented — people rely on word-of-mouth, WhatsApp groups, or unreliable directories. There's no unified platform that lets customers compare providers by rating, proximity, and price in real-time.

### 1.2 Solution

A two-sided web platform where:
- **Customers** browse service categories, find nearby available providers, compare them, and book instantly.
- **Providers** register their skills, set their own rates, go online/offline, and accept bookings at their convenience.

---

## 2. User Roles

| Role | Description |
|------|-------------|
| **Customer** | A person who needs a service (e.g., needs a plumber) |
| **Provider** | A gig worker offering their skill (e.g., a freelance plumber) |
| **Admin** | Platform operator who manages categories, users, and monitors the system |

### 2.1 Dual Role Support

A single user account can act as **both** Customer and Provider. The app provides a **toggle/switch** in the navigation to switch between Customer Mode and Provider Mode. This is because a provider (e.g., a plumber) may also need services themselves (e.g., need a cook).

- One registration, one login, one profile.
- Provider capabilities are unlocked by completing the provider onboarding flow.
- Switching modes changes the entire UI context (different dashboard, different navigation).

---

## 3. Registration & Authentication

### 3.1 Registration Flow

**Single registration for all users:**
1. User signs up with **Email + Password** (Email SSO).
2. User fills basic profile: **Name, Email, Profile Photo** (optional).
3. User lands on the **Customer dashboard** by default.

**Becoming a Provider (opt-in from settings):**
1. User clicks "Become a Provider" from their profile/settings.
2. Fills provider-specific details:
   - Select service categories (can pick multiple — e.g., Plumber + Electrician).
   - Set hourly/per-job rate for each selected service.
   - Set working radius in kilometers (e.g., "I'll travel up to 10 km").
   - Add a short bio/description of experience.
3. Provider profile is created — user can now switch to Provider Mode.

### 3.2 Authentication
- Email + Password login.
- JWT-based stateless session.
- Role context (CUSTOMER / PROVIDER) determined by the active mode toggle, not by separate accounts.

---

## 4. Service Categories (MVP)

Starting with **5 core categories**. Remaining categories are tracked in `LATER.md` for future expansion.

| Category | Sub-services |
|----------|-------------|
| **Acting Driver** | Personal car driver, Outstation trip driver |
| **Cook** | Single meal, Full day cook, Party/event cook |
| **Cleaner** | Home deep clean, Regular cleaning, Post-event cleanup |
| **Plumber** | Leak repair, New fitting, Drainage |

### 5.2 Browse Providers Screen
- User selects a category (e.g., "Plumber").
- Selects sub-service (e.g., "Leak repair").
- Enters/confirms their location (auto-detect or manual input).
- Optionally adds special instructions (text field).
- Sees a list of **available providers** sorted by proximity + rating.

### 5.3 Provider List
- Each provider card shows:
  - Profile photo
  - Name
  - Rating (★ 4.5) and number of reviews
  - Distance (e.g., "2.3 km away")
  - Rate (e.g., "₹300/hr")
- Filters available: Price range, Minimum rating.
- Tap a provider → opens Provider Detail page.

### 5.4 Provider Detail Screen
- Full profile: photo, name, bio, experience description.
- List of services offered with rates.
- Verified badge (if applicable — future).
- Past reviews and ratings from other customers.
- **[Book Now]** button.

### 5.5 Booking Screen
- Service summary (category, sub-service, special instructions).
- Provider details (name, rate).
- Estimated cost (rate × expected duration).
- **[Confirm Booking]** button.
- After confirmation: booking goes to `REQUESTED` state.

### 5.6 Active Booking Screen
- Shows current booking status:
  - `REQUESTED` → Waiting for provider to accept.
  - `ACCEPTED` → Provider has accepted, will arrive.
  - `IN_PROGRESS` → Work is ongoing.
  - `COMPLETED` → Work finished.
- Option to view provider details.

### 5.7 Post-Service Screen
- Rate the provider (1–5 stars).
- Write a text review.
- Submit rating → booking moves to `RATED` state.

---

## 6. Provider Flows

### 6.1 Provider Dashboard (Home)
- **Online / Offline toggle** — prominent switch at the top.
  - When **Online**: provider is discoverable by customers in their radius.
  - When **Offline**: provider is hidden from search results.
- Current/active bookings.
- Recent booking history.

### 6.2 Incoming Booking Requests
- When a customer books this provider, a new booking request appears on the dashboard.
- Request card shows:
  - Service type and sub-service.
  - Customer name.
  - Distance from provider's current location.
  - Estimated duration and pay.
- **No timer / no auto-decline** — provider accepts at their convenience.
- **[Accept]** and **[Decline]** buttons.

### 6.3 Active Job Screen
- Customer address (text — no live map in MVP).
- Status update buttons that the provider clicks in sequence:
  - **[Accept]** → **[Start Job]** → **[Complete Job]**
- Each status change updates the booking state and notifies the customer.

### 6.4 Provider Profile Management
- Edit bio, profile photo.
- Update rate card per service.
- Update working radius (km).
- Add or remove service categories.

### 6.5 Ratings & Reviews
- View overall rating.
- View individual reviews from customers.

---

## 7. Booking Lifecycle

### 7.1 States

```
REQUESTED → ACCEPTED → IN_PROGRESS → COMPLETED → RATED
              ↓
           DECLINED
```

| State | Triggered By | Description |
|-------|-------------|-------------|
| `REQUESTED` | Customer confirms booking | Booking created, waiting for provider |
| `ACCEPTED` | Provider clicks Accept | Provider has agreed to do the job |
| `DECLINED` | Provider clicks Decline | Provider rejected; customer can book another |
| `IN_PROGRESS` | Provider clicks Start Job | Work has begun |
| `COMPLETED` | Provider clicks Complete Job | Work is done |
| `RATED` | Customer submits rating | Customer has reviewed the service |

### 7.2 Rules
- A provider can have **multiple active bookings** (they manage their own schedule).
- If a provider declines, the customer is notified and can choose another provider.
- No auto-reassignment in MVP.
- No cancellation flow in MVP.

---

## 8. Notification System (MVP)

- **In-app only** — a notification bell icon in the navigation bar.
- Notification count badge (unread count).
- Clicking opens a notification dropdown/page with a list.

### 8.1 Notification Triggers

| Event | Notified User | Message Example |
|-------|--------------|-----------------|
| Booking created | Provider | "New booking request for Plumbing from Rahul" |
| Booking accepted | Customer | "Your plumbing request was accepted by Suresh" |
| Booking declined | Customer | "Suresh declined your plumbing request" |
| Job started | Customer | "Suresh has started working on your plumbing request" |
| Job completed | Customer | "Your plumbing job is complete! Please rate Suresh" |
| Rating received | Provider | "Rahul rated you ★4.5 for the plumbing job" |

---

## 9. Admin Panel (Web)

Basic admin capabilities for MVP:
- **User Management:** View all users, their roles (customer/provider/both).
- **Category Management:** Add/edit/disable service categories and sub-services.
- **Booking Overview:** View all bookings, filter by status/date/category.
- **Provider List:** View all providers, their services, ratings.

---

## 10. UI Layout — Responsive Web

### 10.1 Shared Layout
- **Top Navigation Bar:**
  - Logo (left).
  - Role switch toggle: "Customer | Provider" (center or right).
  - Notification bell with unread badge (right).
  - Profile avatar dropdown (right) — Settings, Become a Provider, Logout.
- **Footer:** Minimal — links to About, Contact.

### 10.2 Responsive Behavior
- **Desktop (>1024px):** Side-by-side layouts, provider cards in grid (3–4 per row).
- **Tablet (768–1024px):** 2 cards per row, collapsible navigation.
- **Mobile (<768px):** Single column, bottom navigation bar, hamburger menu.

---

## 11. Out of Scope (MVP)

All items listed in `LATER.md`, including:
- Phone/OTP login
- Aadhaar/document verification
- In-app chat or phone calls
- Payment processing
- Cancellation policy
- Push/SMS notifications
- Auto-cascade booking
- Schedule-for-later bookings
- Mobile native apps
- Live location tracking

