## Chai aur backend series - Youtube

---

## Table of contents
- Project overview
- Tech stack
- Features
- Repo layout
- API: endpoints, auth, request/response examples
  - /api/v1/users
    - POST /register
    - POST /login
    - POST /logout
    - POST /refresh-token
- Data models (User, Video)
- Middleware and utilities
- Environment variables
- Local development (setup & run)
- File uploads & Cloudinary
- Security notes & cookies
- Troubleshooting & common pitfalls
- 
---

## Project overview
This repository is an Express-based backend that implements user registration, login, token issuance (access + refresh), logout and token refresh flows, plus utilities for file uploads (multer), Cloudinary integration, and JWT verification middleware.

The project stores uploaded files temporarily under `public/temp` before uploading to Cloudinary. It stores refresh tokens on the user model and sets tokens as HTTP-only cookies.

---

## Tech stack
- Node.js (ES modules)
- Express
- MongoDB (mongoose)
- Cloudinary (for media uploads)
- multer (file upload middleware)
- jsonwebtoken (JWTs)
- bcrypt (password hashing)
- mongoose-aggregate-paginate-v2 (used by the Video model)

---

## Repo layout (important files)
- `src/app.js` — Express app, CORS and middleware, route registration (mounts `/api/v1/users`).
- `src/index.js` — app entry: loads env, connects to MongoDB and starts server.
- `src/routes/user.routes.js` — user-related routes.
- `src/controllers/user.controller.js` — register, login, logout, refresh token handlers.
- `src/models/user.model.js` — User mongoose model, methods for generating tokens & checking password.
- `src/models/video.model.js` — Video mongoose model (used for video-related data).
- `src/middlewares/auth.middleware.js` — `verifyJWT` to protect routes.
- `src/middlewares/multer.middleware.js` — multer disk storage to `./public/temp`.
- `src/utils/cloudinary.js` — upload to Cloudinary and cleanup local temp file.
- `src/utils/ApiResponse.js` & `src/utils/ApiError.js` — unified response/error wrappers.
- `src/utils/asyncHandler.js` — wrapper to handle async controllers.

---

## Features
- User registration with avatar (required) and optional cover image (file uploads → Cloudinary).
- Login with username OR email; sets access and refresh tokens in HTTP-only cookies and returns tokens in JSON.
- Logout: clears accessToken and refreshToken cookies; removes refreshToken from DB for the user.
- Refresh token endpoint: accepts refresh token from cookie or request body and issues new tokens.
- Middleware to verify JWT either from cookie (`accessToken`) or `Authorization: Bearer <token>`.
- Local disk temp storage for uploaded files and removal after successfully uploading to Cloudinary.

---

## API

Base path: `/api/v1/users`

All responses use the ApiResponse wrapper object:
- shape (success): { stausCode, data, message, success }
- errors and failure responses use ApiError (includes statusCode, message, errors array, success=false)

Note: there are a few naming/typos in the source (e.g., `stausCode` in ApiResponse and `refreshAcessToken` controller name). The routes and behavior below reflect the code's actual endpoints.

### POST /api/v1/users/register
Register a new user (multipart/form-data — file upload required).

- Auth: Public
- Content-Type: multipart/form-data
- Form fields:
  - `fullName` (string) — required
  - `email` (string) — required, unique
  - `username` (string) — required, unique
  - `password` (string) — required
  - `avatar` (file) — required, maxCount: 1 (uploaded to Cloudinary)
  - `coverImage` (file) — optional, maxCount: 1 (uploaded to Cloudinary)

- Notes:
  - Avatar is required; `coverImage` is optional.
  - Files are first stored in `./public/temp` and then uploaded to Cloudinary; local temp files are deleted after upload.

- Successful response:
  - HTTP 201 (controller uses `res.status(201)`) — JSON ApiResponse with created user (password and refreshToken excluded).
  - Example data: { user: { _id, username, email, fullName, avatar, coverImage, createdAt, updatedAt } }

- Example curl:
  curl -v -X POST http://localhost:8000/api/v1/users/register \
    -F "fullName=Amar Kumar" \
    -F "email=amar@example.com" \
    -F "username=amar295" \
    -F "password=secret123" \
    -F "avatar=@/path/to/avatar.jpg" \
    -F "coverImage=@/path/to/cover.jpg"

### POST /api/v1/users/login
Login with username or email and password.

- Auth: Public
- Content-Type: application/json
- Body:
  - `username` (string) OR `email` (string) — at least one required
  - `password` (string) — required

- Response:
  - HTTP 200 — sets HTTP-only cookies `accessToken` and `refreshToken` and returns JSON ApiResponse:
    data includes `{ user, accessToken, refreshToken }` where `user` has password and refreshToken excluded.

- Example curl:
  curl -v -X POST http://localhost:8000/api/v1/users/login \
    -H "Content-Type: application/json" \
    -d '{"username":"amar295","password":"secret123"}'

- Notes:
  - The code sets cookies with options:
    { httpOnly: true, secure: true }
  - For local development, `secure: true` requires HTTPS. If running over HTTP in development, you may want to relax that (or run via HTTPS).

### POST /api/v1/users/logout
Log the user out by clearing cookies and un-setting refresh token in DB.

- Auth: Protected — uses `verifyJWT`
- How auth is read:
  - `verifyJWT` checks `req.cookies?.accessToken` OR `Authorization` header (`Bearer <token>`)
- Behavior:
  - Removes refresh token from the DB for the logged-in user (sets to undefined).
  - Clears cookies `accessToken` and `refreshToken` (with same cookie options) and returns success.

- Example curl (using bearer token):
  curl -v -X POST http://localhost:8000/api/v1/users/logout \
    -H "Authorization: Bearer <accessToken>"

- Example curl (using cookie):
  curl -v -X POST http://localhost:8000/api/v1/users/logout \
    --cookie "accessToken=<accessToken>; refreshToken=<refreshToken>"

### POST /api/v1/users/refresh-token
Refresh access token using refresh token (cookie or body). Note: in code the controller function is named `refreshAcessToken` (typo), route path is `/refresh-token`.

- Auth: Public (but requires a valid refresh token)
- Accepts:
  - Cookie: `refreshToken` OR
  - Request body field: `refreshAcessToken` (note the spelling used in code)
- Behavior:
  - Verifies the refresh token JWT using `process.env.REFRESH_TOKEN_SECRET`
  - Confirms the user exists and matches stored refresh token
  - Generates new `accessToken` and `refreshToken`, sets cookies, and returns them in JSON

- Example curl (using cookie):
  curl -v -X POST http://localhost:8000/api/v1/users/refresh-token \
    --cookie "refreshToken=<your_refresh_token>"

- Example curl (using body param — note parameter name matches code's spelling):
  curl -v -X POST http://localhost:8000/api/v1/users/refresh-token \
    -H "Content-Type: application/json" \
    -d '{"refreshAcessToken":"<your_refresh_token>"}'

- Successful response:
  - HTTP 200 — JSON ApiResponse with `{ accessToken, refreshToken }` and cookies set.

---

## Data Models

### User (src/models/user.model.js)
Important fields:
- username: String, required, unique, lowercase, indexed
- email: String, required, unique, lowercase
- fullName: String, required
- avatar: String (Cloudinary URL), required
- coverImage: String (Cloudinary URL), optional
- watchHistory: [ObjectId -> Video]
- password: String (encrypted via bcrypt)
- refreshToken: stored token string (used for refresh flow)
- timestamps: createdAt, updatedAt

User model also contains instance methods used to:
- generate access token (likely `generateAccessToken()`)
- generate refresh token (likely `generateRefreshToken()`)
- isPasswordCorrect(password) for password verification

(See src/models/user.model.js for exact implementation.)

### Video (src/models/video.model.js)
Fields:
- videoFile: String (Cloudinary URL)
- thumbnaail: String
- owner: ObjectId (ref to User)
- title, description, duration, views, isPublished
- uses mongooseAggregatePaginate for pagination

Although the Video model exists, there are no public video routes in the current code snapshot (they may be added later).

---

## Middleware & utilities

- verifyJWT (src/middlewares/auth.middleware.js)
  - Reads token from `req.cookies.accessToken` or `Authorization: Bearer <token>`
  - Verifies with `process.env.ACCESS_TOKEN_SECRET`
  - On success attaches `req.user` (user doc without password or refreshToken)
  - Throws ApiError(401, ...) for missing/expired/invalid token

- multer configuration (src/middlewares/multer.middleware.js)
  - Disk storage to `./public/temp`
  - Filenames preserved (uses original name)
  - Exported as `upload` used in register route:
    upload.fields([{ name: "avatar", maxCount: 1 }, { name: "coverImage", maxCount: 1 }])

- cloudinary util (src/utils/cloudinary.js)
  - Configured using env vars: CLOUDINARY_CLOUD_NAME, CLOUDINARY_API_KEY, CLOUDINARY_API_SECRET
  - uploadOnCloudinary(localFilePath) uploads and deletes local file on success; deletes on failure

- ApiResponse & ApiError
  - ApiResponse constructs consistent success responses (note `stausCode` property spelling).
  - ApiError extends Error and contains `statusCode`, `data`, `message`, `success=false`, `errors`.

- asyncHandler
  - Wraps async controllers to forward errors to Express error handler

---

## Environment variables
Create a `.env` file in project root with the following keys (used directly in code):

- PORT (optional, default `8000`)
- MONGODB_URL — base MongoDB URL (eg: mongodb+srv://user:pass@cluster.mongodb.net)
- DB_NAME — repository defines `DB_NAME = "612amar"` in `src/constants.js`, but environment may override connection path
- ACCESS_TOKEN_SECRET — JWT secret for access tokens
- REFRESH_TOKEN_SECRET — JWT secret for refresh tokens
- CORS_ORIGIN — allowed origin for CORS (app uses this)
- CLOUDINARY_CLOUD_NAME
- CLOUDINARY_API_KEY
- CLOUDINARY_API_SECRET

Example .env:
PORT=8000
MONGODB_URL="mongodb+srv://<user>:<pass>@cluster0.mongodb.net"
ACCESS_TOKEN_SECRET="some_access_secret"
REFRESH_TOKEN_SECRET="some_refresh_secret"
CLOUDINARY_CLOUD_NAME="..."
CLOUDINARY_API_KEY="..."
CLOUDINARY_API_SECRET="..."
CORS_ORIGIN="http://localhost:3000"

---

## Local development — setup & run

1. Clone repository:
   git clone https://github.com/amar-295/backend.git
   cd backend

2. Install:
   npm install
   (project uses ESM imports — use Node >= 14+)

3. Create `.env` as described above.

4. Start server:
   npm start
   or
   node src/index.js

5. Server will try to connect to MongoDB at `${process.env.MONGODB_URL}/${DB_NAME}` where DB_NAME is `612amar` (see `src/constants.js`) and then start listening on `process.env.PORT` (or `8000`).

---

## File uploads & Cloudinary details
- multer stores uploads to `./public/temp` using original filename.
- After successful Cloudinary upload, local temp file is deleted (via fs.unlinkSync).
- Cloudinary config is read from env vars.
- If upload fails, local temp file is also deleted (cleanup).

---

## Security notes & cookies
- Access and refresh tokens are set as cookies with:
  { httpOnly: true, secure: true }
  - `secure: true` requires HTTPS. For local HTTP development, you may need to conditionally set `secure` to `false` or run over HTTPS.
- `verifyJWT` reads token from cookie OR `Authorization` header: flexible for API clients or browsers.
- Refresh token is persisted on the user record in DB; refresh endpoint validates that stored value matches the incoming token.
- Make sure secrets are strong and not committed to source.

---

## Troubleshooting & common pitfalls
- Cookies not being set in development:
  - Browser blocks cross-site cookies without proper CORS and SameSite settings. app.js config uses `origin: process.env.CORS_ORIGIN, credentials: true`. Ensure front-end sends credentials and the origin matches.
- secure cookie over HTTP:
  - `secure: true` blocks cookie over plain HTTP. For local testing, either set `secure` false or use HTTPS.
- Typo/field name mismatch:
  - The refresh endpoint accepts `refreshAcessToken` in request body (note spelling). Prefer sending refresh token as cookie `refreshToken`.
  - ApiResponse has `stausCode` spelled incorrectly in its property. Be aware when parsing the response object programmatically.
- DB connection:
  - Connection string is `${MONGODB_URL}/${DB_NAME}`; ensure MONGODB_URL does not already include a database name or duplicate names may occur.

---

