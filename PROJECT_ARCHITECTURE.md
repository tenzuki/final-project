# Project Architecture

## 1) API Endpoint Table

Base URL in local development:
- Backend: `http://localhost:8080`
- Frontend (via CRA proxy): `http://localhost:3000`

Auth legend:
- `Public`: no auth required
- `Auth (simple)`: valid JWT required
- `Superadmin (enhance)`: JWT required and user role must be `superadmin`

### Users

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/users` | Public | Register user (role cannot be set in request body) |
| POST | `/users/photo/:id` | Public | Upload profile image for user id |
| POST | `/users/login` | Public | Username/password login |
| POST | `/users/login/facebook` | Public | Facebook login/signup |
| POST | `/users/login/google` | Public | Google login/signup |
| POST | `/users/logout` | Auth (simple) | Logout current token |
| POST | `/users/logoutAll` | Superadmin (enhance) | Remove all active tokens for current user |
| GET | `/users` | Superadmin (enhance) | List all users |
| GET | `/users/me` | Auth (simple) | Get current user |
| GET | `/users/:id` | Superadmin (enhance) | Get user by id |
| PATCH | `/users/me` | Auth (simple) | Update own profile fields |
| PATCH | `/users/:id` | Superadmin (enhance) | Update user by id (including role) |
| DELETE | `/users/:id` | Superadmin (enhance) | Delete user by id |
| DELETE | `/users/me` | Auth (simple) | Delete own account (currently only allowed if role is `superadmin`) |

### Movies

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/movies` | Superadmin (enhance) | Create movie |
| POST | `/movies/photo/:id` | Superadmin (enhance) | Upload movie image |
| GET | `/movies` | Public | List movies |
| GET | `/movies/:id` | Public | Get movie by id |
| PUT | `/movies/:id` | Superadmin (enhance) | Update movie |
| DELETE | `/movies/:id` | Superadmin (enhance) | Delete movie |
| GET | `/movies/usermodeling/:username` | Public | Personalized movie suggestions by username |

### Cinemas

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/cinemas` | Superadmin (enhance) | Create cinema |
| POST | `/cinemas/photo/:id` | Public | Upload cinema image |
| GET | `/cinemas` | Public | List cinemas |
| GET | `/cinemas/:id` | Public | Get cinema by id |
| PATCH | `/cinemas/:id` | Superadmin (enhance) | Update cinema |
| DELETE | `/cinemas/:id` | Superadmin (enhance) | Delete cinema |
| GET | `/cinemas/usermodeling/:username` | Public | Reordered cinemas based on user reservation history |

### Showtimes

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/showtimes` | Superadmin (enhance) | Create showtime |
| GET | `/showtimes` | Public | List showtimes |
| GET | `/showtimes/:id` | Public | Get showtime by id |
| PATCH | `/showtimes/:id` | Superadmin (enhance) | Update showtime |
| DELETE | `/showtimes/:id` | Superadmin (enhance) | Delete showtime |

### Reservations

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/reservations` | Auth (simple) | Create reservation and return QR code |
| GET | `/reservations` | Auth (simple) | List all reservations |
| GET | `/reservations/:id` | Public | Get reservation by id |
| GET | `/reservations/checkin/:id` | Public | Mark reservation as checked-in |
| PATCH | `/reservations/:id` | Superadmin (enhance) | Update reservation |
| DELETE | `/reservations/:id` | Superadmin (enhance) | Delete reservation |
| GET | `/reservations/usermodeling/:username` | Public | Suggested seat preferences for username |

### Invitations

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/invitations` | Auth (simple) | Send invitation emails through Gmail transport |

---

## 2) DB Schema Table

### `User`

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | `String` | Yes | Trimmed |
| `username` | `String` | Yes | Unique, lowercase |
| `email` | `String` | Yes | Unique, lowercase, validated |
| `password` | `String` | No in schema, expected in flow | Hashed with bcrypt in pre-save hook |
| `role` | `String` | No | Enum: `guest`, `admin`, `superadmin`; default `guest` |
| `facebook` | `String` | No | Social login id |
| `google` | `String` | No | Social login id |
| `phone` | `String` | No in schema | Unique, mobile-phone validated if present |
| `imageurl` | `String` | No | Uploaded avatar URL |
| `tokens` | `[{ token: String }]` | No | Active JWT tokens |
| `createdAt`/`updatedAt` | `Date` | Auto | Added via timestamps |

### `Movie`

| Field | Type | Required | Notes |
|---|---|---|---|
| `title` | `String` | Yes | Lowercase |
| `image` | `String` | No | Poster/image URL |
| `language` | `String` | Yes | Lowercase |
| `genre` | `String` | Yes | Lowercase, comma-separated supported in modeling |
| `director` | `String` | Yes | Lowercase |
| `cast` | `String` | Yes | Lowercase |
| `description` | `String` | Yes | Lowercase |
| `duration` | `Number` | Yes | Minutes |
| `releaseDate` | `Date` | Yes | Start availability date |
| `endDate` | `Date` | Yes | End availability date |

### `Cinema`

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | `String` | Yes | Cinema name |
| `ticketPrice` | `Number` | Yes | Per-seat price |
| `city` | `String` | Yes | Lowercase |
| `seats` | `Array<Mixed>` | Yes | 2D seat matrix used by UI |
| `seatsAvailable` | `Number` | Yes | Capacity-like value |
| `image` | `String` | No | Image URL |

### `Showtime`

| Field | Type | Required | Notes |
|---|---|---|---|
| `startAt` | `String` | Yes | Time string |
| `startDate` | `Date` | Yes | Start date |
| `endDate` | `Date` | Yes | End date |
| `movieId` | `ObjectId` | Yes | Ref: `Movie` |
| `cinemaId` | `ObjectId` | Yes | Ref: `Cinema` |

### `Reservation`

| Field | Type | Required | Notes |
|---|---|---|---|
| `date` | `Date` | Yes | Reserved show date |
| `startAt` | `String` | Yes | Show time string |
| `seats` | `Array<Mixed>` | Yes | Seat coordinates like `[row, col]` |
| `ticketPrice` | `Number` | Yes | Snapshot value at booking time |
| `total` | `Number` | Yes | Total amount |
| `movieId` | `ObjectId` | Yes | Ref: `Movie` |
| `cinemaId` | `ObjectId` | Yes | Ref: `Cinema` |
| `username` | `String` | Yes | Denormalized username |
| `phone` | `String` | Yes | Denormalized phone |
| `checkin` | `Boolean` | No | Default `false` |

---

## 3) Role/Permission Matrix (Current Behavior)

Roles used by app:
- Visitor: not logged in
- Guest: logged in user with role `guest`
- Admin: logged in user with role `admin`
- Superadmin: logged in user with role `superadmin`

Important implementation detail:
- Backend `auth.enhance` currently allows only `superadmin`.  
- So users with role `admin` can open admin UI pages but many write operations fail server-side.

| Capability | Visitor | Guest | Admin | Superadmin |
|---|---|---|---|---|
| Browse movies/cinemas/showtimes | Yes | Yes | Yes | Yes |
| Register/login | Yes | N/A | N/A | N/A |
| Create reservation | No | Yes | Yes | Yes |
| Send invitations | No | Yes | Yes | Yes |
| View all reservations (`GET /reservations`) | No | Yes | Yes | Yes |
| Create/update/delete movies | No | No | No (blocked by backend) | Yes |
| Create/update/delete cinemas | No | No | No (blocked by backend) | Yes |
| Create/update/delete showtimes | No | No | No (blocked by backend) | Yes |
| Update/delete reservations by id | No | No | No (blocked by backend) | Yes |
| List/manage all users | No | No | No | Yes |

---

## 4) Known Gaps and Hardening Checklist

### High-priority security gaps

- [ ] Move JWT secret out of code (`'mySecret'`) into environment variable, rotate old tokens.
- [ ] Protect `POST /users/photo/:id` and `POST /cinemas/photo/:id` with auth and ownership/role checks.
- [ ] Protect `GET /reservations/:id` and `GET /reservations/checkin/:id` (currently public and state-changing via GET).
- [ ] Restrict `GET /reservations` to scoped user data unless requester is privileged.
- [ ] Add authorization checks for user-modeling endpoints to prevent data enumeration by username.

### Authorization/role model consistency

- [ ] Decide intended privileges for `admin` and update middleware accordingly (currently effectively superadmin-only for protected CRUD).
- [ ] Align frontend route guard with role checks (not only `isAuthenticated`) for admin pages.
- [ ] Fix `/users/me` delete logic if self-delete should be allowed for normal users.

### Data integrity and booking correctness

- [ ] Enforce unique seat booking atomically in backend (current booking validates in UI, not with DB transaction/lock).
- [ ] Validate that reservation `movieId`, `cinemaId`, `date`, and `startAt` match a valid showtime.
- [ ] Avoid trusting client-sent `ticketPrice` and `total`; compute on server.
- [ ] Normalize and validate seat matrix shape and seat coordinate bounds.

### API and platform hygiene

- [ ] Replace permissive CORS (`*`) with explicit allow-list.
- [ ] Add centralized error middleware and consistent error response format.
- [ ] Version API paths (for example `/api/v1/...`) for future compatibility.
- [ ] Add request validation layer (Joi/Zod/express-validator).
- [ ] Add rate limiting and basic abuse protection on auth and invitation endpoints.

### Secrets and email operations

- [ ] Ensure `.env` values for MongoDB and Gmail are documented and managed securely.
- [ ] For Gmail sending, use app passwords/OAuth2; avoid plain credentials patterns.

### Frontend/session reliability

- [ ] Call `loadUser()` at app startup to restore session user data from JWT.
- [ ] Handle token expiry and forced logout cleanly.
- [ ] Avoid exposing privileged UI actions when role is insufficient.

### Testing and observability

- [ ] Add API integration tests for auth, role access, reservations, check-in, and invitations.
- [ ] Add unit tests for user-modeling helpers.
- [ ] Add logging/monitoring for failed auth, booking conflicts, and mail delivery errors.
- [ ] Add CI checks for lint/test/build (current workflow is CodeQL-only).

