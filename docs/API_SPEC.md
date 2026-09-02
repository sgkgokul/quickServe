# QuickServe — REST API Specification

> Base URL: `/api/v1`
> Auth: `Authorization: Bearer <jwt-token>` (unless marked 🔓 Public)
> All responses follow the standard wrapper format.

---

## Standard Response Wrapper

### Success
```json
{
  "success": true,
  "data": { ... },
  "message": "Operation successful",
  "timestamp": "2026-04-10T14:30:00"
}
```

### Paginated Success
```json
{
  "success": true,
  "data": {
    "content": [ ... ],
    "page": 0,
    "size": 10,
    "totalElements": 42,
    "totalPages": 5
  },
  "message": "Fetched successfully",
  "timestamp": "2026-04-10T14:30:00"
}
```

### Error
```json
{
  "success": false,
  "data": null,
  "message": "Validation failed",
  "errors": {
    "email": "must be a valid email",
    "name": "must not be blank"
  },
  "timestamp": "2026-04-10T14:30:00"
}
```

### HTTP Status Codes Used

| Code | Meaning |
|------|---------|
| 200 | OK — successful read/update |
| 201 | Created — successful resource creation |
| 400 | Bad Request — validation error |
| 401 | Unauthorized — missing or invalid token |
| 403 | Forbidden — wrong role or not allowed |
| 404 | Not Found — resource doesn't exist |
| 409 | Conflict — duplicate or invalid state transition |
| 500 | Internal Server Error |

---

## 1. Auth APIs

### 1.1 Register — `POST /api/v1/auth/register` 🔓

**Request:**
```json
{
  "name": "Rahul Kumar",
  "email": "rahul@example.com",
  "password": "SecurePass123!"
}
```

**Validation:**
- `name`: required, 2–100 chars
- `email`: required, valid email format, unique
- `password`: required, min 8 chars, must contain uppercase + lowercase + digit

**Response (201):**
```json
{
  "success": true,
  "data": {
    "id": "a1b2c3d4-...",
    "name": "Rahul Kumar",
    "email": "rahul@example.com",
    "activeRole": "CUSTOMER",
    "isProvider": false,
    "createdAt": "2026-04-10T14:30:00"
  },
  "message": "Registration successful"
}
```

**Errors:**
| Code | Condition |
|------|-----------|
| 400 | Validation fails |
| 409 | Email already registered |

---

### 1.2 Login — `POST /api/v1/auth/login` 🔓

**Request:**
```json
{
  "email": "rahul@example.com",
  "password": "SecurePass123!"
}
```

**Response (200):**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOi...",
    "refreshToken": "dGhpcyBpcy...",
    "tokenType": "Bearer",
    "expiresIn": 3600,
    "user": {
      "id": "a1b2c3d4-...",
      "name": "Rahul Kumar",
      "email": "rahul@example.com",
      "activeRole": "CUSTOMER",
      "isProvider": false,
      "profilePhotoUrl": null
    }
  },
  "message": "Login successful"
}
```

**Errors:**
| Code | Condition |
|------|-----------|
| 401 | Invalid email or password |

---

### 1.3 Refresh Token — `POST /api/v1/auth/refresh` 🔓

**Request:**
```json
{
  "refreshToken": "dGhpcyBpcy..."
}
```

**Response (200):**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOi...(new)",
    "refreshToken": "dGhpcyBpcy...(new)",
    "tokenType": "Bearer",
    "expiresIn": 3600
  },
  "message": "Token refreshed"
}
```

**Errors:**
| Code | Condition |
|------|-----------|
| 401 | Invalid or expired refresh token |

---

## 2. User Profile APIs

> All require authentication.

### 2.1 Get My Profile — `GET /api/v1/users/profile`

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "a1b2c3d4-...",
    "name": "Rahul Kumar",
    "email": "rahul@example.com",
    "profilePhotoUrl": "https://...",
    "activeRole": "CUSTOMER",
    "isProvider": false,
    "createdAt": "2026-04-10T14:30:00"
  }
}
```

---

### 2.2 Update My Profile — `PUT /api/v1/users/profile`

**Request:**
```json
{
  "name": "Rahul K.",
  "profilePhotoUrl": "https://cdn.quickserve.com/photos/rahul.jpg"
}
```

**Validation:**
- `name`: optional, 2–100 chars (if provided)
- `profilePhotoUrl`: optional, valid URL format

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "a1b2c3d4-...",
    "name": "Rahul K.",
    "email": "rahul@example.com",
    "profilePhotoUrl": "https://cdn.quickserve.com/photos/rahul.jpg",
    "activeRole": "CUSTOMER",
    "isProvider": false,
    "createdAt": "2026-04-10T14:30:00"
  },
  "message": "Profile updated"
}
```

---

### 2.3 Become a Provider — `POST /api/v1/users/become-provider`

> Upgrades a customer account to also have provider capabilities. One-time action.

**Request:**
```json
{
  "bio": "Experienced plumber with 5 years of work in Chennai area.",
  "address": "12, Gandhi Street, T. Nagar, Chennai",
  "latitude": 13.0407,
  "longitude": 80.2338,
  "workingRadiusKm": 15,
  "services": [
    {
      "subServiceId": "b1000000-0000-0000-0000-000000000009",
      "ratePerHour": 350.00,
      "rateType": "PER_HOUR"
    },
    {
      "subServiceId": "b1000000-0000-0000-0000-000000000010",
      "ratePerHour": 500.00,
      "rateType": "FIXED"
    }
  ]
}
```

**Validation:**
- `bio`: optional, max 2000 chars
- `address`: required, max 500 chars
- `latitude`: required, -90 to 90
- `longitude`: required, -180 to 180
- `workingRadiusKm`: optional (default 10), range 1–50
- `services`: required, at least 1 service
  - `subServiceId`: required, must exist in `sub_services` table
  - `ratePerHour`: required, > 0
  - `rateType`: optional, one of `PER_HOUR`, `FIXED` (default: `PER_HOUR`)

**Response (201):**
```json
{
  "success": true,
  "data": {
    "userId": "a1b2c3d4-...",
    "providerId": "p1p2p3p4-...",
    "bio": "Experienced plumber with 5 years of work in Chennai area.",
    "address": "12, Gandhi Street, T. Nagar, Chennai",
    "latitude": 13.0407,
    "longitude": 80.2338,
    "workingRadiusKm": 15,
    "isOnline": false,
    "avgRating": 0.0,
    "totalReviews": 0,
    "services": [
      {
        "id": "ps-001-...",
        "subServiceId": "b1000000-0000-0000-0000-000000000009",
        "subServiceName": "Leak Repair",
        "categoryName": "Plumber",
        "ratePerHour": 350.00,
        "rateType": "PER_HOUR",
        "isActive": true
      }
    ]
  },
  "message": "Provider profile created"
}
```

**Errors:**
| Code | Condition |
|------|-----------|
| 400 | Validation fails |
| 409 | User is already a provider |
| 404 | Sub-service ID doesn't exist |

---

### 2.4 Switch Active Role — `POST /api/v1/users/switch-role`

> Toggle between CUSTOMER and PROVIDER mode. Only works if `isProvider = true`.

**Request:**
```json
{
  "role": "PROVIDER"
}
```

**Validation:**
- `role`: required, one of `CUSTOMER`, `PROVIDER`

**Response (200):**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOi...(new token with updated role)",
    "refreshToken": "dGhpcyBpcy...(new)",
    "tokenType": "Bearer",
    "expiresIn": 3600,
    "activeRole": "PROVIDER"
  },
  "message": "Switched to PROVIDER mode"
}
```

**Errors:**
| Code | Condition |
|------|-----------|
| 400 | Invalid role value |
| 403 | User is not a provider (cannot switch to PROVIDER) |

---

## 3. Provider APIs

> All require authentication + PROVIDER active role.

### 3.1 Get My Provider Profile — `GET /api/v1/provider/profile`

**Response (200):**
```json
{
  "success": true,
  "data": {
    "providerId": "p1p2p3p4-...",
    "userId": "a1b2c3d4-...",
    "name": "Rahul Kumar",
    "email": "rahul@example.com",
    "profilePhotoUrl": "https://...",
    "bio": "Experienced plumber...",
    "address": "12, Gandhi Street...",
    "latitude": 13.0407,
    "longitude": 80.2338,
    "workingRadiusKm": 15,
    "isOnline": false,
    "avgRating": 4.50,
    "totalReviews": 12,
    "services": [
      {
        "id": "ps-001-...",
        "subServiceId": "b1000000-...",
        "subServiceName": "Leak Repair",
        "categoryName": "Plumber",
        "ratePerHour": 350.00,
        "rateType": "PER_HOUR",
        "isActive": true
      }
    ],
    "createdAt": "2026-04-10T14:30:00"
  }
}
```

---

### 3.2 Update Provider Profile — `PUT /api/v1/provider/profile`

**Request:**
```json
{
  "bio": "Updated bio text...",
  "address": "New address...",
  "latitude": 13.05,
  "longitude": 80.24,
  "workingRadiusKm": 20
}
```

**Validation:** Same as become-provider (except `services` not included here — managed separately)

**Response (200):** Same shape as 3.1 response with updated values.

---

### 3.3 Toggle Availability — `PUT /api/v1/provider/availability`

> Go online or offline. When online, provider is discoverable in search. Publishes to Kafka `provider-status-events`.

**Request:**
```json
{
  "isOnline": true,
  "latitude": 13.0407,
  "longitude": 80.2338
}
```

**Validation:**
- `isOnline`: required, boolean
- `latitude`: required if `isOnline=true` (provider must share location when going online)
- `longitude`: required if `isOnline=true`

**Response (200):**
```json
{
  "success": true,
  "data": {
    "providerId": "p1p2p3p4-...",
    "isOnline": true,
    "latitude": 13.0407,
    "longitude": 80.2338
  },
  "message": "You are now online"
}
```

---

### 3.4 Add Service — `POST /api/v1/provider/services`

**Request:**
```json
{
  "subServiceId": "b1000000-0000-0000-0000-000000000011",
  "ratePerHour": 400.00,
  "rateType": "PER_HOUR"
}
```

**Validation:**
- `subServiceId`: required, must exist, must not already be added by this provider
- `ratePerHour`: required, > 0
- `rateType`: optional, one of `PER_HOUR`, `FIXED` (default: `PER_HOUR`)

**Response (201):**
```json
{
  "success": true,
  "data": {
    "id": "ps-002-...",
    "subServiceId": "b1000000-0000-0000-0000-000000000011",
    "subServiceName": "Drainage",
    "categoryName": "Plumber",
    "ratePerHour": 400.00,
    "rateType": "PER_HOUR",
    "isActive": true
  },
  "message": "Service added"
}
```

**Errors:**
| Code | Condition |
|------|-----------|
| 404 | Sub-service doesn't exist |
| 409 | Provider already offers this sub-service |

---

### 3.5 Update Service Rate — `PUT /api/v1/provider/services/{serviceId}`

**Request:**
```json
{
  "ratePerHour": 450.00,
  "rateType": "FIXED",
  "isActive": true
}
```

**Response (200):** Updated service object (same shape as 3.4 response).

**Errors:**
| Code | Condition |
|------|-----------|
| 404 | Service not found or doesn't belong to this provider |

---

### 3.6 Remove Service — `DELETE /api/v1/provider/services/{serviceId}`

**Response (200):**
```json
{
  "success": true,
  "data": null,
  "message": "Service removed"
}
```

**Errors:**
| Code | Condition |
|------|-----------|
| 404 | Service not found or doesn't belong to this provider |
| 409 | Cannot remove — active bookings exist for this service |

---

### 3.7 List My Services — `GET /api/v1/provider/services`

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": "ps-001-...",
      "subServiceId": "b1000000-...",
      "subServiceName": "Leak Repair",
      "categoryName": "Plumber",
      "ratePerHour": 350.00,
      "rateType": "PER_HOUR",
      "isActive": true
    }
  ]
}
```

---

## 4. Catalog APIs

> 🔓 Public — no authentication required.

### 4.1 List Categories — `GET /api/v1/categories` 🔓

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": "a1000000-0000-0000-0000-000000000001",
      "name": "Acting Driver",
      "icon": "driver",
      "description": "Professional drivers for personal or outstation trips",
      "displayOrder": 1,
      "subServiceCount": 2
    },
    {
      "id": "a1000000-0000-0000-0000-000000000004",
      "name": "Plumber",
      "icon": "plumber",
      "description": "Plumbing repairs, fittings, and drainage work",
      "displayOrder": 4,
      "subServiceCount": 3
    }
  ]
}
```

---

### 4.2 List Sub-Services by Category — `GET /api/v1/categories/{categoryId}/sub-services` 🔓

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": "b1000000-0000-0000-0000-000000000009",
      "name": "Leak Repair",
      "description": "Fix leaking pipes, taps, and faucets",
      "categoryId": "a1000000-0000-0000-0000-000000000004",
      "categoryName": "Plumber"
    },
    {
      "id": "b1000000-0000-0000-0000-000000000010",
      "name": "New Fitting",
      "description": "Install new plumbing fixtures and fittings",
      "categoryId": "a1000000-0000-0000-0000-000000000004",
      "categoryName": "Plumber"
    }
  ]
}
```

**Errors:**
| Code | Condition |
|------|-----------|
| 404 | Category not found |

---

## 5. Search APIs

> Requires authentication (CUSTOMER role).

### 5.1 Search Providers — `GET /api/v1/search/providers`

**Query Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `categoryId` | UUID | Yes | Service category to search in |
| `subServiceId` | UUID | No | Specific sub-service (narrows results) |
| `lat` | Double | Yes | Customer's latitude |
| `lng` | Double | Yes | Customer's longitude |
| `radiusKm` | Integer | No | Search radius in km (default: 10, max: 50) |
| `sortBy` | String | No | `distance` (default), `rating`, `price_low`, `price_high` |
| `page` | Integer | No | Page number (default: 0) |
| `size` | Integer | No | Page size (default: 10, max: 20) |

**Example:** `GET /api/v1/search/providers?categoryId=a100...004&lat=13.04&lng=80.23&radiusKm=15&sortBy=rating`

**Response (200):**
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "providerId": "p1p2p3p4-...",
        "userId": "a1b2c3d4-...",
        "name": "Suresh Kumar",
        "profilePhotoUrl": "https://...",
        "bio": "Expert plumber...",
        "distanceKm": 3.2,
        "avgRating": 4.50,
        "totalReviews": 12,
        "isOnline": true,
        "services": [
          {
            "subServiceId": "b1000000-...",
            "subServiceName": "Leak Repair",
            "ratePerHour": 350.00,
            "rateType": "PER_HOUR"
          }
        ]
      }
    ],
    "page": 0,
    "size": 10,
    "totalElements": 5,
    "totalPages": 1
  }
}
```

**Logic:**
1. Filter providers who are **online** (Redis check).
2. Filter providers within **radiusKm** using PostGIS Haversine query.
3. Filter providers who offer the requested **category** (and optionally **sub-service**).
4. Sort by the requested `sortBy` parameter.
5. Return paginated results.

---

### 5.2 Get Provider Public Profile — `GET /api/v1/search/providers/{providerId}`

> Customer views a specific provider's full public profile before booking.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "providerId": "p1p2p3p4-...",
    "name": "Suresh Kumar",
    "profilePhotoUrl": "https://...",
    "bio": "Expert plumber with 5+ years...",
    "avgRating": 4.50,
    "totalReviews": 12,
    "isOnline": true,
    "services": [
      {
        "subServiceId": "b1000000-...",
        "subServiceName": "Leak Repair",
        "categoryName": "Plumber",
        "ratePerHour": 350.00,
        "rateType": "PER_HOUR"
      }
    ],
    "recentRatings": [
      {
        "customerName": "Rahul K.",
        "stars": 5,
        "reviewText": "Excellent work, fixed the leak quickly!",
        "createdAt": "2026-04-05T10:00:00"
      }
    ]
  }
}
```

**Errors:**
| Code | Condition |
|------|-----------|
| 404 | Provider not found |

---

## 6. Booking APIs

### 6.1 Create Booking — `POST /api/v1/bookings`

> Requires CUSTOMER active role. Publishes `BOOKING_REQUESTED` to Kafka `booking-events`.

**Request:**
```json
{
  "providerId": "p1p2p3p4-...",
  "subServiceId": "b1000000-0000-0000-0000-000000000009",
  "description": "Kitchen tap is leaking, need urgent repair",
  "estimatedDurationHrs": 2,
  "customerAddress": "45, Anna Nagar, Chennai 600040",
  "customerLat": 13.085,
  "customerLng": 80.2101
}
```

**Validation:**
- `providerId`: required, must exist and be online
- `subServiceId`: required, must be offered by this provider
- `description`: optional, max 2000 chars
- `estimatedDurationHrs`: optional, 1–24
- `customerAddress`: required, max 500 chars
- `customerLat`: required
- `customerLng`: required

**Response (201):**
```json
{
  "success": true,
  "data": {
    "id": "bk-001-...",
    "status": "REQUESTED",
    "customer": {
      "id": "a1b2c3d4-...",
      "name": "Rahul Kumar"
    },
    "provider": {
      "id": "p1p2p3p4-...",
      "name": "Suresh Kumar"
    },
    "subService": {
      "id": "b1000000-...",
      "name": "Leak Repair",
      "categoryName": "Plumber"
    },
    "description": "Kitchen tap is leaking...",
    "estimatedCost": 700.00,
    "estimatedDurationHrs": 2,
    "customerAddress": "45, Anna Nagar...",
    "bookedAt": "2026-04-10T15:00:00",
    "createdAt": "2026-04-10T15:00:00"
  },
  "message": "Booking created — waiting for provider to accept"
}
```

**Errors:**
| Code | Condition |
|------|-----------|
| 400 | Validation fails |
| 404 | Provider or sub-service not found |
| 409 | Provider is offline |
| 409 | Provider doesn't offer this sub-service |

---

### 6.2 Accept Booking — `PUT /api/v1/bookings/{bookingId}/accept`

> Requires PROVIDER active role. Only the assigned provider can accept. Publishes `BOOKING_ACCEPTED`.

**Request:** No body needed.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "bk-001-...",
    "status": "ACCEPTED",
    "acceptedAt": "2026-04-10T15:05:00"
  },
  "message": "Booking accepted"
}
```

**Errors:**
| Code | Condition |
|------|-----------|
| 403 | Not the assigned provider |
| 404 | Booking not found |
| 409 | Booking is not in REQUESTED status |

---

### 6.3 Decline Booking — `PUT /api/v1/bookings/{bookingId}/decline`

> Requires PROVIDER active role. Publishes `BOOKING_DECLINED`.

**Request:** No body needed.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "bk-001-...",
    "status": "DECLINED"
  },
  "message": "Booking declined"
}
```

**Errors:** Same as 6.2.

---

### 6.4 Start Job — `PUT /api/v1/bookings/{bookingId}/start`

> Requires PROVIDER active role. Only if status is ACCEPTED. Publishes `BOOKING_STARTED`.

**Request:** No body needed.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "bk-001-...",
    "status": "IN_PROGRESS",
    "startedAt": "2026-04-10T15:30:00"
  },
  "message": "Job started"
}
```

**Errors:**
| Code | Condition |
|------|-----------|
| 403 | Not the assigned provider |
| 404 | Booking not found |
| 409 | Booking is not in ACCEPTED status |


---

### 6.5 Complete Job — `PUT /api/v1/bookings/{bookingId}/complete`

> Requires PROVIDER active role. Only if status is IN_PROGRESS. Publishes `BOOKING_COMPLETED`.

**Request:** No body needed.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "bk-001-...",
    "status": "COMPLETED",
    "completedAt": "2026-04-10T17:30:00"
  },
  "message": "Job completed"
}
```

**Errors:**
| Code | Condition |
|------|-----------|
| 403 | Not the assigned provider |
| 404 | Booking not found |
| 409 | Booking is not in IN_PROGRESS status |

---

### 6.6 List My Bookings — `GET /api/v1/bookings`

> Returns bookings based on active role. CUSTOMER sees their bookings, PROVIDER sees bookings assigned to them.

**Query Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `status` | String | No | Filter: `REQUESTED`, `ACCEPTED`, `IN_PROGRESS`, `COMPLETED`, `DECLINED`, `RATED` |
| `page` | Integer | No | Page number (default: 0) |
| `size` | Integer | No | Page size (default: 10, max: 20) |

**Response (200):**
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "bk-001-...",
        "status": "COMPLETED",
        "customer": {
          "id": "a1b2c3d4-...",
          "name": "Rahul Kumar"
        },
        "provider": {
          "id": "p1p2p3p4-...",
          "name": "Suresh Kumar"
        },
        "subService": {
          "id": "b1000000-...",
          "name": "Leak Repair",
          "categoryName": "Plumber"
        },
        "estimatedCost": 700.00,
        "customerAddress": "45, Anna Nagar...",
        "bookedAt": "2026-04-10T15:00:00",
        "completedAt": "2026-04-10T17:30:00"
      }
    ],
    "page": 0,
    "size": 10,
    "totalElements": 15,
    "totalPages": 2
  }
}
```

---

### 6.7 Get Booking Details — `GET /api/v1/bookings/{bookingId}`

> Returns full booking details. Only accessible by the customer or provider involved.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "bk-001-...",
    "status": "COMPLETED",
    "customer": {
      "id": "a1b2c3d4-...",
      "name": "Rahul Kumar",
      "profilePhotoUrl": "https://..."
    },
    "provider": {
      "id": "p1p2p3p4-...",
      "name": "Suresh Kumar",
      "profilePhotoUrl": "https://...",
      "avgRating": 4.50
    },
    "subService": {
      "id": "b1000000-...",
      "name": "Leak Repair",
      "categoryName": "Plumber"
    },
    "description": "Kitchen tap is leaking...",
    "estimatedCost": 700.00,
    "estimatedDurationHrs": 2,
    "customerAddress": "45, Anna Nagar, Chennai 600040",
    "customerLat": 13.085,
    "customerLng": 80.2101,
    "bookedAt": "2026-04-10T15:00:00",
    "acceptedAt": "2026-04-10T15:05:00",
    "startedAt": "2026-04-10T15:30:00",
    "completedAt": "2026-04-10T17:30:00",
    "rating": {
      "stars": 5,
      "reviewText": "Great work!",
      "createdAt": "2026-04-10T18:00:00"
    }
  }
}
```

**Errors:**
| Code | Condition |
|------|-----------|
| 403 | Not involved in this booking |
| 404 | Booking not found |

---

## 7. Rating APIs

### 7.1 Submit Rating — `POST /api/v1/bookings/{bookingId}/rating`

> Requires CUSTOMER active role. Only for COMPLETED bookings. Booking moves to RATED status. Publishes `BOOKING_RATED`.

**Request:**
```json
{
  "stars": 5,
  "reviewText": "Excellent work! Fixed the leak quickly and cleanly."
}
```

**Validation:**
- `stars`: required, integer 1–5
- `reviewText`: optional, max 1000 chars

**Response (201):**
```json
{
  "success": true,
  "data": {
    "id": "rt-001-...",
    "bookingId": "bk-001-...",
    "stars": 5,
    "reviewText": "Excellent work! Fixed the leak quickly and cleanly.",
    "customerName": "Rahul Kumar",
    "providerName": "Suresh Kumar",
    "createdAt": "2026-04-10T18:00:00"
  },
  "message": "Rating submitted — thank you!"
}
```

**Errors:**
| Code | Condition |
|------|-----------|
| 400 | Validation fails |
| 403 | Not the customer of this booking |
| 404 | Booking not found |
| 409 | Booking is not in COMPLETED status |
| 409 | Rating already submitted for this booking |

---

### 7.2 List Provider Ratings — `GET /api/v1/providers/{providerId}/ratings`

> 🔓 Public endpoint — anyone can view a provider's reviews.

**Query Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `page` | Integer | No | Page number (default: 0) |
| `size` | Integer | No | Page size (default: 10, max: 20) |

**Response (200):**
```json
{
  "success": true,
  "data": {
    "summary": {
      "avgRating": 4.50,
      "totalReviews": 12
    },
    "reviews": {
      "content": [
        {
          "id": "rt-001-...",
          "stars": 5,
          "reviewText": "Excellent work!",
          "customerName": "Rahul K.",
          "serviceName": "Leak Repair",
          "createdAt": "2026-04-10T18:00:00"
        }
      ],
      "page": 0,
      "size": 10,
      "totalElements": 12,
      "totalPages": 2
    }
  }
}
```

**Errors:**
| Code | Condition |
|------|-----------|
| 404 | Provider not found |

---

## 8. Notification APIs

> All require authentication.

### 8.1 List My Notifications — `GET /api/v1/notifications`

**Query Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `page` | Integer | No | Page number (default: 0) |
| `size` | Integer | No | Page size (default: 15, max: 30) |

**Response (200):**
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "nt-001-...",
        "type": "BOOKING_ACCEPTED",
        "title": "Booking Accepted",
        "message": "Your plumbing request was accepted by Suresh Kumar",
        "bookingId": "bk-001-...",
        "isRead": false,
        "createdAt": "2026-04-10T15:05:00"
      }
    ],
    "page": 0,
    "size": 15,
    "totalElements": 8,
    "totalPages": 1
  }
}
```

---

### 8.2 Get Unread Count — `GET /api/v1/notifications/unread-count`

**Response (200):**
```json
{
  "success": true,
  "data": {
    "unreadCount": 3
  }
}
```

---

### 8.3 Mark as Read — `PUT /api/v1/notifications/{notificationId}/read`

**Request:** No body needed.

**Response (200):**
```json
{
  "success": true,
  "data": null,
  "message": "Notification marked as read"
}
```

**Errors:**
| Code | Condition |
|------|-----------|
| 404 | Notification not found or doesn't belong to user |

---

### 8.4 Mark All as Read — `PUT /api/v1/notifications/read-all`

**Request:** No body needed.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "markedCount": 3
  },
  "message": "All notifications marked as read"
}
```

---

## 9. Admin APIs (Future — MVP Placeholder)

> Requires ADMIN role. These endpoints are documented for completeness but will be implemented post-MVP.

| # | Endpoint | Method | Description |
|---|----------|--------|-------------|
| 9.1 | `/api/v1/admin/users` | GET | List all users (paginated, filterable) |
| 9.2 | `/api/v1/admin/users/{id}` | GET | View user details |
| 9.3 | `/api/v1/admin/providers` | GET | List all providers with ratings |
| 9.4 | `/api/v1/admin/bookings` | GET | List all bookings (filter by status, date, category) |
| 9.5 | `/api/v1/admin/categories` | POST | Add a new service category |
| 9.6 | `/api/v1/admin/categories/{id}` | PUT | Update category |
| 9.7 | `/api/v1/admin/categories/{id}/sub-services` | POST | Add sub-service to category |
| 9.8 | `/api/v1/admin/dashboard/stats` | GET | Dashboard stats (total users, bookings today, revenue) |

---

## 10. Endpoint Summary

### Total: 28 Endpoints

| # | Method | Endpoint | Auth | Role |
|---|--------|----------|------|------|
| 1 | POST | `/api/v1/auth/register` | 🔓 | — |
| 2 | POST | `/api/v1/auth/login` | 🔓 | — |
| 3 | POST | `/api/v1/auth/refresh` | 🔓 | — |
| 4 | GET | `/api/v1/users/profile` | 🔒 | Any |
| 5 | PUT | `/api/v1/users/profile` | 🔒 | Any |
| 6 | POST | `/api/v1/users/become-provider` | 🔒 | CUSTOMER |
| 7 | POST | `/api/v1/users/switch-role` | 🔒 | Any |
| 8 | GET | `/api/v1/provider/profile` | 🔒 | PROVIDER |
| 9 | PUT | `/api/v1/provider/profile` | 🔒 | PROVIDER |
| 10 | PUT | `/api/v1/provider/availability` | 🔒 | PROVIDER |
| 11 | POST | `/api/v1/provider/services` | 🔒 | PROVIDER |
| 12 | PUT | `/api/v1/provider/services/{id}` | 🔒 | PROVIDER |
| 13 | DELETE | `/api/v1/provider/services/{id}` | 🔒 | PROVIDER |
| 14 | GET | `/api/v1/provider/services` | 🔒 | PROVIDER |
| 15 | GET | `/api/v1/categories` | 🔓 | — |
| 16 | GET | `/api/v1/categories/{id}/sub-services` | 🔓 | — |
| 17 | GET | `/api/v1/search/providers` | 🔒 | CUSTOMER |
| 18 | GET | `/api/v1/search/providers/{id}` | 🔒 | CUSTOMER |
| 19 | POST | `/api/v1/bookings` | 🔒 | CUSTOMER |
| 20 | PUT | `/api/v1/bookings/{id}/accept` | 🔒 | PROVIDER |
| 21 | PUT | `/api/v1/bookings/{id}/decline` | 🔒 | PROVIDER |
| 22 | PUT | `/api/v1/bookings/{id}/start` | 🔒 | PROVIDER |
| 23 | PUT | `/api/v1/bookings/{id}/complete` | 🔒 | PROVIDER |
| 24 | GET | `/api/v1/bookings` | 🔒 | Any |
| 25 | GET | `/api/v1/bookings/{id}` | 🔒 | Any (owner) |
| 26 | POST | `/api/v1/bookings/{id}/rating` | 🔒 | CUSTOMER |
| 27 | GET | `/api/v1/providers/{id}/ratings` | 🔓 | — |
| 28 | GET | `/api/v1/notifications` | 🔒 | Any |
| 29 | GET | `/api/v1/notifications/unread-count` | 🔒 | Any |
| 30 | PUT | `/api/v1/notifications/{id}/read` | 🔒 | Any |
| 31 | PUT | `/api/v1/notifications/read-all` | 🔒 | Any |

---

## 11. Booking State Transition Rules

```
REQUESTED ──► ACCEPTED ──► IN_PROGRESS ──► COMPLETED ──► RATED
    │
    └──► DECLINED
```

| Current State | Allowed Transitions | Who Can Trigger |
|---------------|--------------------|----|
| `REQUESTED` | `ACCEPTED`, `DECLINED` | Provider |
| `ACCEPTED` | `IN_PROGRESS` | Provider |
| `IN_PROGRESS` | `COMPLETED` | Provider |
| `COMPLETED` | `RATED` | Customer (via rating submission) |
| `DECLINED` | — (terminal) | — |
| `RATED` | — (terminal) | — |

---

## 12. Kafka Events Reference

| Event | Topic | Producer | Consumer | Trigger |
|-------|-------|----------|----------|---------|
| `BOOKING_REQUESTED` | `booking-events` | Booking Module | Notification Module | Customer creates booking |
| `BOOKING_ACCEPTED` | `booking-events` | Booking Module | Notification Module | Provider accepts |
| `BOOKING_DECLINED` | `booking-events` | Booking Module | Notification Module | Provider declines |
| `BOOKING_STARTED` | `booking-events` | Booking Module | Notification Module | Provider starts job |
| `BOOKING_COMPLETED` | `booking-events` | Booking Module | Notification Module | Provider completes job |
| `BOOKING_RATED` | `booking-events` | Rating Module | Notification Module | Customer submits rating |
| `PROVIDER_ONLINE` | `provider-status-events` | User Module | Search Module | Provider goes online |
| `PROVIDER_OFFLINE` | `provider-status-events` | User Module | Search Module | Provider goes offline |