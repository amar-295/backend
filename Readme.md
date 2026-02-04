# ☕ Chai aur Backend — Express Authentication Backend

Production-style backend built as part of the **Chai aur Backend** YouTube series.  
This project demonstrates real-world authentication, file uploads, token handling, and security patterns using Node.js and Express.

---

## 📌 Project Overview

This repository contains an **Express-based backend** that supports:

- User registration with avatar & cover image uploads
- Login using **username or email**
- JWT-based authentication (Access + Refresh tokens)
- Secure logout & refresh-token rotation
- File uploads using **multer + Cloudinary**
- MongoDB integration with Mongoose models

Uploaded files are stored temporarily in `public/temp`, uploaded to Cloudinary, and then removed from local storage.

---

## 🛠 Tech Stack

- **Node.js** (ES Modules)
- **Express**
- **MongoDB + Mongoose**
- **Cloudinary** (media uploads)
- **multer** (file handling)
- **jsonwebtoken** (JWT)
- **bcrypt** (password hashing)
- **mongoose-aggregate-paginate-v2**

---

## 📂 Repository Structure

src/
├── app.js # Express app setup & middleware
├── index.js # Entry point (DB connect + server start)
├── routes/
│ └── user.routes.js # User routes
├── controllers/
│ └── user.controller.js # Register, login, logout, refresh
├── models/
│ ├── user.model.js # User schema & methods
│ └── video.model.js # Video schema
├── middlewares/
│ ├── auth.middleware.js # verifyJWT
│ └── multer.middleware.js # multer config
├── utils/
│ ├── cloudinary.js
│ ├── ApiResponse.js
│ ├── ApiError.js
│ └── asyncHandler.js
└── constants.js # DB_NAME and constants


---

## ✨ Features

- ✅ Register with **required avatar** & optional cover image
- ✅ Login using **email OR username**
- ✅ Access & refresh tokens via **HTTP-only cookies**
- ✅ Secure logout (DB + cookie cleanup)
- ✅ Refresh-token rotation with DB validation
- ✅ JWT verification from **cookies or Authorization header**
- ✅ Automatic cleanup of temp files after upload

---

## 🔗 API Overview

**Base URL**

/api/v1/users


All responses follow a unified structure using `ApiResponse`:

```json
{
  "stausCode": 200,
  "data": {},
  "message": "Success",
  "success": true
}
⚠️ Note: stausCode is intentionally misspelled in the source.

🧑‍💻 User Routes
➕ POST /register
Register a new user (multipart/form-data)

Fields

fullName (string) — required

email (string) — required, unique

username (string) — required, unique

password (string) — required

avatar (file) — required

coverImage (file) — optional

Example

curl -X POST http://localhost:8000/api/v1/users/register \
-F "fullName=Amar Kumar" \
-F "email=amar@example.com" \
-F "username=amar295" \
-F "password=secret123" \
-F "avatar=@avatar.jpg"
🔐 POST /login
Login using username or email.

Request Body

{
  "username": "amar295",
  "password": "secret123"
}
Behavior

Sets accessToken & refreshToken as HTTP-only cookies

Returns user + tokens in response

🚪 POST /logout
Protected route.

Clears cookies

Removes refresh token from DB

Header

Authorization: Bearer <accessToken>
🔁 POST /refresh-token
Generates new tokens using refresh token.

Accepts

Cookie: refreshToken

OR body field (code typo):

{
  "refreshAcessToken": "<token>"
}
Returns new access & refresh tokens and updates cookies.

🧱 Data Models
👤 User Model
Key fields:

username, email, fullName

avatar, coverImage

password (bcrypt hashed)

refreshToken

watchHistory

Instance methods:

generateAccessToken()

generateRefreshToken()

isPasswordCorrect()

🎥 Video Model
Includes:

videoFile, thumbnail

owner

views, duration, isPublished

Uses aggregation pagination

No public video routes yet.

🔐 Authentication Middleware
verifyJWT
Reads token from:

req.cookies.accessToken

OR Authorization: Bearer <token>

Attaches sanitized req.user

Throws 401 on invalid or expired token

☁️ File Uploads & Cloudinary
multer stores files in public/temp

Files are uploaded to Cloudinary

Local temp files are deleted after success or failure

Environment Variables

CLOUDINARY_CLOUD_NAME
CLOUDINARY_API_KEY
CLOUDINARY_API_SECRET
🌱 Environment Variables
Create a .env file in the project root:

PORT=8000
MONGODB_URL=mongodb+srv://<user>:<pass>@cluster.mongodb.net
ACCESS_TOKEN_SECRET=your_access_secret
REFRESH_TOKEN_SECRET=your_refresh_secret
CLOUDINARY_CLOUD_NAME=xxxx
CLOUDINARY_API_KEY=xxxx
CLOUDINARY_API_SECRET=xxxx
CORS_ORIGIN=http://localhost:3000
▶️ Local Development
git clone https://github.com/amar-295/backend.git
cd backend
npm install
npm start
Server runs at:

http://localhost:8000
🔒 Security Notes
Cookies are configured as:

{ httpOnly: true, secure: true }
⚠️ For local HTTP development:

Set secure: false, or

Run backend over HTTPS

Refresh tokens are validated against the database to prevent reuse and token theft.

🐛 Common Pitfalls
Cookies not being set

Enable credentials: true on frontend

Ensure CORS_ORIGIN matches frontend URL

Secure cookies on HTTP

Cookies won’t be stored — disable secure locally

Refresh token typo

Body field name is refreshAcessToken (matches source code)

MongoDB connection issues

Ensure DB name is not duplicated in MONGODB_URL

📺 About the Series
This project is part of the Chai aur Backend YouTube series, focused on teaching backend development using real-world patterns, clean architecture, and production-ready practices.

