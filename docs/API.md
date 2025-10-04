# CuddleCare Childcare Booking System — API Documentation (Complete)

**Version:** 1.0
**Base URL:** `https://api.cuddlecare.com/api/v1`
**Style guide:** API JSON uses **camelCase** keys (e.g., `firstName`). Database column names use **snake_case**.

---

# 1. Overview & Design Decisions

## 1.1 Architecture & Auth recommendation

**Recommended pattern:**
**Frontend → Your API (Backend) → Keycloak → Database**

**Why:**

* Central control of flows (user profile creation, custom validation, audit logs).
* Easier to customize responses and keep DB in sync with Keycloak.
* Easier to change auth provider later without touching frontend.

**Flow examples:**

* Registration: `Frontend → POST /auth/register → Backend → Keycloak Admin API → create local profile → respond to frontend`
* Login: `Frontend → POST /auth/login → Backend (forwards form data) → Keycloak /token → tokens returned → Backend may persist session info → returns tokens to frontend`

## 1.2 Security / Headers

Protected endpoints require:

```
Authorization: Bearer <access_token>
Content-Type: application/json
```

(Backend should validate token signature and scopes/roles.)

## 1.3 Error response format (standard)

All errors follow this format:

```json
{
  "error": {
    "code": "InvalidCredentials",
    "message": "Invalid username or password",
    "details": null
  }
}
```

## 1.4 Naming consistency

* Request/response JSON: **camelCase** (e.g., `firstName`, `providerId`).
* DB columns: **snake_case** (e.g., `first_name`, `provider_id`). Map between them in DTOs.

## 1.5 Rate limits, validations, and logs

* Enforce rate limits on auth endpoints.
* Validate payloads; return `400` with error details for invalid inputs.
* Log Keycloak failures as `KeycloakConnectionError` with trace ID.

---

# 2. Authentication

## 2.1 Register User

**URL:** `/auth/register`
**Method:** `POST`
**Description:** Register a new user (parent or provider). Backend creates user in Keycloak via Admin API, then creates the profile record in your DB.

**Request Body (frontend → API):**

```json
{
  "userName": "hanan123",
  "email": "hanan@gmail.com",
  "password": "12345",
  "role": "parent"
}
```

**Success Response (201):**

```json
{
  "message": "Registration successful",
  "user": {
    "id": "b9a3c8d4-7e2b-45b7-bb61-2d42a9f20c78",
    "userName": "hanan123",
    "email": "hanan@gmail.com",
    "role": "parent"
  }
}
```

**Failure responses:**

* `400` Missing/invalid fields:

```json
{
  "error": {
    "code": "InvalidInput",
    "message": "Email or password is missing or invalid",
    "details": { "field": "email" }
  }
}
```

* `409` User already exists:

```json
{
  "error": {
    "code": "UserAlreadyExists",
    "message": "Username or email already registered",
    "details": null
  }
}
```

* `503` Keycloak connection problems:

```json
{
  "error": {
    "code": "KeycloakConnectionError",
    "message": "Unable to connect to Keycloak server",
    "details": null
  }
}
```

**Notes:**

* Backend should not return raw Keycloak internals.
* After Keycloak user creation, store any extra profile data in local DB (parents/providers tables).

---

## 2.2 Login (Frontend → API → Keycloak)

**URL:** `/auth/login`
**Method:** `POST`
**Content-Type (frontend → API):** `application/json`
**Description:** Backend will forward credentials to Keycloak token endpoint as `application/x-www-form-urlencoded` and return token(s) to frontend.

**Request Body (frontend → API):**

```json
{
  "username": "hanan123",
  "password": "12345"
}
```

**Backend will call Keycloak**:

```
POST /realms/cuddlecare/protocol/openid-connect/token
Content-Type: application/x-www-form-urlencoded

client_id=cuddlecare-frontend
grant_type=password
username=hanan123
password=12345
```

**Success Response (200):**

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5...",
  "expiresIn": 300,
  "tokenType": "Bearer",
  "scope": "profile email roles"
}
```

**Failure (Keycloak default), forwarded by backend (400/401):**

```json
{
  "error": {
    "code": "InvalidCredentials",
    "message": "Invalid username or password",
    "details": null
  }
}
```

**Notes:**

* Backend can augment tokens with additional session metadata if needed.
* Consider returning `refreshExpiresIn` and `sessionState` if you want to support session management.

---

## 2.3 Logout

**URL:** `/auth/logout`
**Method:** `POST`
**Headers:** `Authorization: Bearer <accessToken>`
**Description:** Backend calls Keycloak logout endpoint to invalidate refresh token/session.

**Backend call to Keycloak (example):**

```
POST /realms/cuddlecare/protocol/openid-connect/logout
Content-Type: application/x-www-form-urlencoded

client_id=cuddlecare-frontend
refresh_token=<refreshToken>
```

**Success Response (200):**

```json
{
  "message": "Logout successful"
}
```

**Failure:**
Return `400`/`503` with `KeycloakConnectionError` or other relevant error described earlier.

---

# 3. Parents

## 3.1 Add Parent Information

**URL:** `/parents/info`
**Method:** `POST`
**Auth:** `Bearer` required
**Description:** Save extended parent profile data linked to an authenticated user.

**Request Body:**

```json
{
  "firstName": "John",
  "lastName": "Doe",
  "phoneNo1": "+25198765432",
  "phoneNo2": "+25196359862",
  "emergencyName": "Anna",
  "emergencyPhone": "+251965893256",
  "address": "Addis Ababa"
}
```

**Success Response (201):**

```json
{
  "message": "Information added successfully",
  "parentId": 123
}
```

---

## 3.2 Get Parent Profile

**URL:** `/parents/{parentId}`
**Method:** `GET`
**Auth:** `Bearer` (only owner/admin)
**Response:**

```json
{
  "parentId": 1,
  "firstName": "John",
  "lastName": "Doe",
  "email": "john@example.com",
  "phoneNo1": "+251912345678",
  "phoneNo2": "+251912345679",
  "address": "Addis Ababa",
  "emergencyName": "Anna",
  "emergencyPhone": "+251965893256",
  "children": [
    {
      "childId": 2,
      "fullName": "Sara Fatih",
      "age": 4,
      "dob": "2021-03-15",
      "gender": "female"
    }
  ]
}
```

---

## 3.3 Update Parent Profile

**URL:** `/parents/{parentId}`
**Method:** `PUT`
**Auth:** `Bearer` (owner/admin)
**Request Body (partial allowed):**

```json
{
  "phoneNo1": "+251962358964",
  "address": "Addis Ababa"
}
```

**Success Response:**

```json
{
  "message": "Parent profile updated successfully"
}
```

---

## 3.4 Delete Parent Profile

**URL:** `/parents/{parentId}`
**Method:** `DELETE`
**Auth:** `Bearer` (admin or owner with restrictions)
**Response:**

```json
{
  "message": "Parent profile deleted successfully"
}
```

**Note:** Consider soft-deletes (mark `isDeleted`) for auditability.

---

# 4. Children

## 4.1 Add Child

**URL:** `/children`
**Method:** `POST`
**Auth:** `Bearer` (parent)
**Request Body:**

```json
{
  "parentId": 1,
  "fullName": "Zara John",
  "age": 5,
  "dob": "2020-03-15",
  "gender": "female",
  "allergies": "none",
  "specialNeeds": "none"
}
```

**Success (201):**

```json
{
  "childId": 2,
  "message": "Child added successfully"
}
```

---

## 4.2 Get All Children By Parent

**URL:** `/children/parent/{parentId}`
**Method:** `GET`
**Auth:** `Bearer` (owner/admin)
**Response:**

```json
[
  {
    "childId": 2,
    "fullName": "Zara John",
    "age": 5,
    "dob": "2020-03-15",
    "gender": "female",
    "allergies": "none",
    "specialNeeds": "none"
  }
]
```

---

## 4.3 Update Child

**URL:** `/children/{childId}`
**Method:** `PUT`
**Auth:** `Bearer` (parent/admin)
**Request Body (example):**

```json
{
  "allergies": "Peanut Allergies"
}
```

**Success:**

```json
{
  "message": "Child info updated successfully"
}
```

---

## 4.4 Delete Child

**URL:** `/children/{childId}`
**Method:** `DELETE`
**Auth:** `Bearer` (parent/admin)
**Response:**

```json
{
  "message": "Child info deleted"
}
```

---

# 5. Providers

## 5.1 Register Provider

**URL:** `/providers/register`
**Method:** `POST`
**Auth:** `Bearer` required (or registration flows that create Keycloak user + provider record)
**Request Body:**

```json
{
  "userId": 45,
  "companyName": "Little Steps Daycare",
  "address": "Bole, Addis Ababa",
  "licenseNumber": "123adn",
  "services": ["daycare", "tutoring"],
  "capacity": 10,
  "approvalStatus": "Pending"
}
```

**Success (201):**

```json
{
  "providerId": 10,
  "message": "Provider registered successfully"
}
```

---

## 5.2 Get All Providers

**URL:** `/providers`
**Method:** `GET`
**Query params (optional):** `?location=Bole&service=daycare&minRating=4`
**Response:**

```json
[
  {
    "providerId": 10,
    "companyName": "Little Steps Daycare",
    "rating": 4.8,
    "location": "Bole, Addis Ababa",
    "services": ["daycare", "tutoring"],
    "capacity": 12,
    "approvalStatus": "Approved"
  }
]
```

---

## 5.3 Update Provider

**URL:** `/providers/{providerId}`
**Method:** `PUT`
**Auth:** `Bearer` (provider owner/admin)
**Request Body (example):**

```json
{
  "capacity": 15,
  "rating": 5
}
```

**Success:**

```json
{
  "message": "Provider updated successfully"
}
```

---

## 5.4 Delete Provider

**URL:** `/providers/{providerId}`
**Method:** `DELETE`
**Auth:** `Bearer` (admin)
**Response:**

```json
{
  "message": "Provider deleted successfully"
}
```

---

# 6. Bookings

## 6.1 Create Booking

**URL:** `/bookings`
**Method:** `POST`
**Auth:** `Bearer` (parent)
**Request Body:**

```json
{
  "parentId": 1,
  "childId": 2,
  "providerId": 10,
  "startDate": "2025-10-05",
  "endDate": "2025-10-05",
  "duration": "half-day",
  "amount": 500,
  "notes": "Please feed child lunch at noon"
}
```

**Success (201):**

```json
{
  "bookingId": 23,
  "status": "pending",
  "message": "Booking request sent to provider"
}
```

---

## 6.2 Get Bookings By Parent

**URL:** `/bookings/parent/{parentId}`
**Method:** `GET`
**Auth:** `Bearer` (owner/admin)
**Response:**

```json
[
  {
    "bookingId": 23,
    "childId": 2,
    "providerId": 10,
    "startDate": "2025-10-05",
    "endDate": "2025-10-05",
    "amount": 500,
    "status": "pending"
  }
]
```

---

## 6.3 Get Bookings By Provider

**URL:** `/bookings/provider/{providerId}`
**Method:** `GET`
**Auth:** `Bearer` (provider/admin)
**Response similar to above.**

---

## 6.4 Update Booking Status

**URL:** `/bookings/{bookingId}/status`
**Method:** `PUT`
**Auth:** `Bearer` (provider or system)
**Request Body:**

```json
{
  "status": "confirmed"  // pending | confirmed | cancelled | completed
}
```

**Success:**

```json
{
  "message": "Booking status updated to confirmed"
}
```

**Notes:**

* Booking status can be updated by provider, or automatically by system after check-ins/check-outs.
* Emit notifications on status changes.

---

# 7. Payments

## 7.1 Initiate Payment

**URL:** `/payments/initiate`
**Method:** `POST`
**Auth:** `Bearer` (parent)
**Description:** Initiate payment using chosen gateway (the API may call a payment provider).

**Request Body:**

```json
{
  "bookingId": 23,
  "payerId": 1,
  "payeeId": 10,
  "amount": 500,
  "method": "card",                 // card | cash | mobile_money
  "paymentType": "ParentToProvider" // ParentToProvider | ProviderToPlatform
}
```

**Success (201) - example (if async):**

```json
{
  "paymentId": 1001,
  "status": "processing",
  "paymentGatewayUrl": "https://payments.example.com/txn/abcd"
}
```

**If synchronous / gateway confirms:**

```json
{
  "paymentId": 1001,
  "status": "paid",
  "message": "Payment successful",
  "paidAt": "2025-10-05T12:45:00Z"
}
```

---

## 7.2 Payment Status

**URL:** `/payments/status/{bookingId}`
**Method:** `GET`
**Auth:** `Bearer` (owner/admin)
**Response:**

```json
{
  "paymentId": 1001,
  "bookingId": 23,
  "status": "paid",
  "amount": 500,
  "paidAt": "2025-10-05T12:45:00Z"
}
```

---

## 7.3 Payment History

**URL:** `/payments/history/parent/{parentId}`
**Method:** `GET`
**Auth:** `Bearer` (owner/admin)
**Response array of payments.**

---

# 8. Activities / Updates (Provider -> Parent)

## 8.1 Add Activity

**URL:** `/activities`
**Method:** `POST`
**Auth:** `Bearer` (provider)
**Request Body:**

```json
{
  "childId": 2,
  "providerId": 10,
  "activityType": "nap", // nap | meal | play | assignment | fun_moment
  "description": "Child slept for 1 hour",
  "photoUrl": "https://cdn.cuddlecare.com/activities/abc123.jpg",
  "timestamp": "2025-10-05T10:30:00Z"
}
```

**Success (201):**

```json
{
  "activityId": 501,
  "message": "Activity added"
}
```

---

## 8.2 Get Activities By Child

**URL:** `/activities/child/{childId}`
**Method:** `GET`
**Auth:** `Bearer` (parent/provider/admin)
**Response:** array of activities.

---

# 9. Notifications & Messages

## 9.1 Send Notification (system/admin)

**URL:** `/notifications/send`
**Method:** `POST`
**Auth:** `Bearer` (system/admin)
**Request Body:**

```json
{
  "receiverId": 1,
  "senderId": 10,
  "title": "Booking Approved",
  "message": "Your booking with Little Steps Daycare has been approved.",
  "type": "alert" // info | alert | chat
}
```

**Success:**

```json
{
  "notificationId": 5,
  "message": "Notification sent"
}
```

---

## 9.2 Get Notifications for Parent

**URL:** `/notifications/parent/{parentId}`
**Method:** `GET`
**Auth:** `Bearer` (owner/admin)
**Response:**

```json
[
  {
    "notificationId": 5,
    "title": "Booking Approved",
    "message": "Your booking with Little Steps Daycare has been approved.",
    "type": "alert",
    "status": "unread",
    "createdAt": "2025-10-05T12:00:00Z"
  }
]
```

---

# 10. Admin

## 10.1 Admin Dashboard

**URL:** `/admin/dashboard`
**Method:** `GET`
**Auth:** `Bearer` (admin)
**Response:**

```json
{
  "totalParents": 150,
  "totalProviders": 80,
  "activeBookings": 65,
  "completedBookings": 120,
  "pendingProviders": 4
}
```

---

## 10.2 Suspend/Unsuspend User

**URL:** `/admin/user/{userId}/suspend`
**Method:** `PUT`
**Auth:** `Bearer` (admin)
**Request Body:**

```json
{
  "suspended": true,
  "reason": "Violation of terms"
}
```

**Success:**

```json
{
  "message": "User account suspended successfully"
}
```

---

## 10.3 Approve Provider

**URL:** `/admin/providers/{providerId}/approve`
**Method:** `PUT`
**Auth:** `Bearer` (admin)
**Request Body:**

```json
{
  "approvalStatus": "Approved"
}
```

**Success:**

```json
{
  "message": "Provider approved successfully"
}
```

---

# 11. File / Folder Structure (Backend)

```
/src
 ├── controllers/
 ├── services/
 ├── repositories/
 ├── models/
 ├── dtos/
 ├── config/
 │    └── keycloak.js (or KeycloakConfig)
 ├── routes/
 ├── middlewares/
 │    └── authMiddleware.js (validates tokens)
 └── utils/
```

* Keep Keycloak logic in `services/keycloakService.js` for reusability.

---

# 14. Additional Recommended Enhancements (concrete)

1. **Consistent naming convention**: switch to camelCase across all API JSON. (Done in this doc.)
2. **Validation & OpenAPI**: write OpenAPI 3.0 spec (YAML) and generate client/server stubs.
3. **Standardized Error Codes**: create a codes list (e.g., `InvalidInput`, `UserAlreadyExists`, `Unauthorized`, `NotFound`, `KeycloakConnectionError`).
4. **Soft deletes** for sensitive deletes (users, children) to preserve audit logs.
5. **Role-based middleware**: centralize RBAC (roles: parent, provider, admin).
6. **Event-driven notifications**: use message queue for heavy tasks (payment webhooks, activity photo processing).
7. **Payment idempotency**: support `Idempotency-Key` header for payment attempts.
8. **Rate limiting & brute force protection**: throttle auth endpoints.
9. **Data sync tasks**: scheduled job to reconcile Keycloak users with local DB (handle deleted/suspended users).
10. **Monitoring & health checks**: endpoints: `/health`, `/metrics`. Monitor Keycloak connectivity.
11. **API versioning**: prefix with `/api/v1/`. Plan for `/api/v2/` breaking changes.
12. **Documentation**: export this to `API_Documentation.md` and generate Swagger UI from OpenAPI.

---

# 15. Example cURL snippets

## Register (frontend → backend)

```bash
curl -X POST "https://api.cuddlecare.com/api/v1/auth/register" \
 -H "Content-Type: application/json" \
 -d '{
   "userName":"hanan123",
   "email":"hanan@gmail.com",
   "password":"12345",
   "role":"parent"
 }'
```

## Login (frontend → backend)

```bash
curl -X POST "https://api.cuddlecare.com/api/v1/auth/login" \
 -H "Content-Type: application/json" \
 -d '{
   "username":"hanan123",
   "password":"12345"
 }'
```

(Backend will call Keycloak token endpoint.)

---

# 16. Next steps I can do for you (pick any)

* Generate an **OpenAPI (Swagger) YAML** from this doc (ready to import into Swagger UI / Postman).
* Create `API_Documentation.md` file content and provide it formatted for Git commit.
* Provide **Spring Boot** example code for `/auth/register` and `/auth/login` integrating with Keycloak Admin REST.
* Generate Postman collection JSON.

---