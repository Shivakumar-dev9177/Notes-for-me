# JWT Authentication — Complete Production Guide
# Login | Register | Logout | Cookies | Security

---

## Table of Contents
1. What is JWT
2. JWT Structure Explained
3. Access Token + Refresh Token Strategy
4. Full Flow (Register → Login → Refresh → Logout)
5. File Structure
6. All Code Files
7. Security Checklist
8. Common Mistakes to Avoid

---

## 1. What is JWT

JWT = JSON Web Token.
A way to prove who a user is without storing session data on the server.

Traditional session (old way):
  - User logs in → server stores session in DB → gives browser a session ID
  - Every request → server looks up session ID in DB → slow

JWT (modern way):
  - User logs in → server creates a signed token → gives browser the token
  - Every request → server just verifies the token math → no DB lookup → fast

---

## 2. JWT Structure

Every JWT is 3 parts separated by dots:

  eyJhbGciOiJIUzI1NiJ9.eyJpZCI6MSwibmFtZSI6IlNoaXZhIn0.abc123xyz
        HEADER                      PAYLOAD                  SIGNATURE

HEADER (base64 decoded):
  { "alg": "HS256" }
  → Just says which algorithm was used

PAYLOAD (base64 decoded):
  {
    "id": 1,
    "name": "Shivakumar",
    "email": "shiva@gmail.com",
    "iat": 1717286400,   ← issued at (timestamp)
    "exp": 1717287300    ← expires at (timestamp)
  }
  → Your actual user data
  → WARNING: This is NOT encrypted — anyone can decode it
  → Never put passwords or secrets in payload

SIGNATURE:
  HMAC_SHA256(header + "." + payload, JWT_SECRET)
  → Created using your secret key
  → If anyone changes the payload, signature breaks
  → Server rejects tampered tokens

---

## 3. Access Token + Refresh Token Strategy

NEVER use a single long-lived token in production.

Access Token:
  - Short lived (15 minutes)
  - Stored as HttpOnly cookie
  - If stolen → attacker only has 15 minutes of access
  - Verified by JWT signature → no DB lookup needed

Refresh Token:
  - Long lived (7 days)
  - Random string (NOT a JWT)
  - Stored in database AND as HttpOnly cookie
  - Can be deleted from DB anytime → instant revocation
  - Rotated on every use → stolen token gets detected

Why two tokens?
  Short access token = less risk if stolen
  Refresh token in DB = can be revoked unlike a JWT
  Best of both: speed + security

---

## 4. Full Flow

### REGISTER
  User fills form (name, email, password)
        ↓
  POST /api/auth/register
        ↓
  Check email not already in DB
        ↓
  bcrypt.hash(password, 10) → never store plain text
        ↓
  INSERT INTO users (name, email, hashedPassword)
        ↓
  Return success → frontend redirects to /login

### LOGIN
  User fills form (email, password)
        ↓
  POST /api/auth/login
        ↓
  SELECT user WHERE email = ?
        ↓
  bcrypt.compare(enteredPassword, storedHash)
        ↓
  If match:
    Create access token (JWT, 15 min)
    Create refresh token (random 64 bytes, 7 days)
    Save refresh token in DB with expiry
    Set access_token as HttpOnly cookie (15 min)
    Set refresh_token as HttpOnly cookie (7 days)
        ↓
  Frontend redirects to protected page

### EVERY REQUEST WITHIN 15 MIN WINDOW
  Browser sends access_token cookie automatically
        ↓
  Middleware reads cookie
        ↓
  jwtVerify(token, JWT_SECRET)
        ↓
  Valid? → allow request
  Invalid/expired? → send 401

### ACCESS TOKEN EXPIRES (after 15 min)
  Browser calls POST /api/auth/refresh
        ↓
  Server reads refresh_token cookie
        ↓
  SELECT from refresh_tokens WHERE token = ? AND expires_at > NOW()
        ↓
  Found?
    DELETE old refresh token from DB (rotation)
    Create new access token (15 min)
    Create new refresh token (7 days)
    Save new refresh token in DB
    Set new cookies
        ↓
  Not found / expired?
    Return 401 → frontend redirects to /login

### LOGOUT
  User clicks Logout
        ↓
  POST /api/auth/logout
        ↓
  Read refresh_token cookie
        ↓
  DELETE FROM refresh_tokens WHERE token = ?
        ↓
  Clear both cookies (maxAge: 0)
        ↓
  Redirect to /login
  Even if access_token was stolen → expires in max 15 min
  Refresh token is dead → attacker cannot get new access tokens

---

## 5. File Structure

  src/
  ├── lib/
  │   ├── db.js                    ← PostgreSQL pool + auto table creation
  │   └── auth.js                  ← all JWT/token helper functions
  ├── app/
  │   ├── api/
  │   │   └── auth/
  │   │       ├── register/route.js
  │   │       ├── login/route.js
  │   │       ├── logout/route.js
  │   │       ├── refresh/route.js
  │   │       └── me/route.js
  │   ├── login/page.jsx
  │   └── register/page.jsx
  ├── components/
  │   ├── Navbar.jsx               ← async server component, reads user from cookie
  │   └── LogoutButton.jsx        ← client component
  └── middleware.js                ← protects routes before page loads

  .env.local                       ← secrets (never commit to git)

---

## 6. All Code Files

============================================================
FILE: .env.local
============================================================

  DATABASE_URL=postgresql://postgres:yourpassword%40here@localhost:5432/myapp
  JWT_SECRET=<generate with: node -e "console.log(require('crypto').randomBytes(64).toString('hex'))">

  NOTE: If password contains @ → encode it as %40 in the URL
  NOTE: JWT_SECRET must be at least 64 random characters

============================================================
FILE: src/lib/db.js
============================================================

  import { Pool } from "pg";

  const pool = new Pool({
    connectionString: process.env.DATABASE_URL,
  });

  async function initDb() {
    await pool.query(`
      CREATE TABLE IF NOT EXISTS users (
        id         SERIAL PRIMARY KEY,
        name       VARCHAR(255) NOT NULL,
        email      VARCHAR(255) UNIQUE NOT NULL,
        password   VARCHAR(255) NOT NULL,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      );

      CREATE TABLE IF NOT EXISTS refresh_tokens (
        id         SERIAL PRIMARY KEY,
        user_id    INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
        token      VARCHAR(512) UNIQUE NOT NULL,
        expires_at TIMESTAMP NOT NULL,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      );
    `);
  }

  initDb().catch(console.error);

  export default pool;

============================================================
FILE: src/lib/auth.js
============================================================

  import { SignJWT, jwtVerify } from "jose";
  import { cookies } from "next/headers";
  import crypto from "crypto";

  const ACCESS_SECRET  = new TextEncoder().encode(process.env.JWT_SECRET);
  const ACCESS_EXPIRY  = "15m";
  const REFRESH_EXPIRY = 7; // days

  // Create a short-lived access token (JWT)
  export async function signAccessToken(payload) {
    return await new SignJWT(payload)
      .setProtectedHeader({ alg: "HS256" })
      .setIssuedAt()
      .setExpirationTime(ACCESS_EXPIRY)
      .sign(ACCESS_SECRET);
  }

  // Verify an access token — returns payload or null
  export async function verifyAccessToken(token) {
    try {
      const { payload } = await jwtVerify(token, ACCESS_SECRET);
      return payload;
    } catch {
      return null;
    }
  }

  // Generate a random refresh token string
  export function generateRefreshToken() {
    return crypto.randomBytes(64).toString("hex");
  }

  // Calculate refresh token expiry date
  export function refreshTokenExpiry() {
    const date = new Date();
    date.setDate(date.getDate() + REFRESH_EXPIRY);
    return date;
  }

  // Set access token as HttpOnly cookie
  export function setAccessTokenCookie(response, token) {
    response.cookies.set("access_token", token, {
      httpOnly: true,
      secure: process.env.NODE_ENV === "production",
      sameSite: "lax",
      maxAge: 60 * 15, // 15 minutes
      path: "/",
    });
  }

  // Set refresh token as HttpOnly cookie
  export function setRefreshTokenCookie(response, token) {
    response.cookies.set("refresh_token", token, {
      httpOnly: true,
      secure: process.env.NODE_ENV === "production",
      sameSite: "lax",
      maxAge: 60 * 60 * 24 * 7, // 7 days
      path: "/",
    });
  }

  // Get current logged-in user from access token cookie
  export async function getUser() {
    const cookieStore = await cookies();
    const token = cookieStore.get("access_token")?.value;
    if (!token) return null;
    return await verifyAccessToken(token);
  }

============================================================
FILE: src/app/api/auth/register/route.js
============================================================

  import { NextResponse } from "next/server";
  import bcrypt from "bcryptjs";
  import pool from "@/lib/db";

  export async function POST(request) {
    try {
      const { name, email, password } = await request.json();

      if (!name || !email || !password)
        return NextResponse.json({ error: "All fields are required" }, { status: 400 });

      if (password.length < 6)
        return NextResponse.json({ error: "Password must be at least 6 characters" }, { status: 400 });

      const existing = await pool.query("SELECT id FROM users WHERE email = $1", [email]);
      if (existing.rows.length > 0)
        return NextResponse.json({ error: "Email already registered" }, { status: 400 });

      const hashedPassword = await bcrypt.hash(password, 10);

      const result = await pool.query(
        "INSERT INTO users (name, email, password) VALUES ($1, $2, $3) RETURNING id, name, email",
        [name, email, hashedPassword]
      );

      return NextResponse.json(result.rows[0], { status: 201 });
    } catch (err) {
      console.error("Register error:", err.message);
      return NextResponse.json({ error: err.message }, { status: 500 });
    }
  }

============================================================
FILE: src/app/api/auth/login/route.js
============================================================

  import { NextResponse } from "next/server";
  import bcrypt from "bcryptjs";
  import pool from "@/lib/db";
  import {
    signAccessToken,
    generateRefreshToken,
    refreshTokenExpiry,
    setAccessTokenCookie,
    setRefreshTokenCookie,
  } from "@/lib/auth";

  export async function POST(request) {
    try {
      const { email, password } = await request.json();

      if (!email || !password)
        return NextResponse.json({ error: "All fields are required" }, { status: 400 });

      const result = await pool.query("SELECT * FROM users WHERE email = $1", [email]);
      const user = result.rows[0];

      if (!user)
        return NextResponse.json({ error: "Invalid email or password" }, { status: 401 });

      const passwordMatch = await bcrypt.compare(password, user.password);
      if (!passwordMatch)
        return NextResponse.json({ error: "Invalid email or password" }, { status: 401 });

      const accessToken = await signAccessToken({
        id: user.id,
        name: user.name,
        email: user.email,
      });

      const refreshToken = generateRefreshToken();
      const expiresAt = refreshTokenExpiry();

      await pool.query(
        "INSERT INTO refresh_tokens (user_id, token, expires_at) VALUES ($1, $2, $3)",
        [user.id, refreshToken, expiresAt]
      );

      const response = NextResponse.json({ id: user.id, name: user.name, email: user.email });
      setAccessTokenCookie(response, accessToken);
      setRefreshTokenCookie(response, refreshToken);

      return response;
    } catch (err) {
      console.error("Login error:", err.message);
      return NextResponse.json({ error: err.message }, { status: 500 });
    }
  }

============================================================
FILE: src/app/api/auth/refresh/route.js
============================================================

  import { NextResponse } from "next/server";
  import { cookies } from "next/headers";
  import pool from "@/lib/db";
  import {
    signAccessToken,
    generateRefreshToken,
    refreshTokenExpiry,
    setAccessTokenCookie,
    setRefreshTokenCookie,
  } from "@/lib/auth";

  export async function POST() {
    try {
      const cookieStore = await cookies();
      const refreshToken = cookieStore.get("refresh_token")?.value;

      if (!refreshToken)
        return NextResponse.json({ error: "No refresh token" }, { status: 401 });

      const result = await pool.query(
        `SELECT rt.*, u.name, u.email
         FROM refresh_tokens rt
         JOIN users u ON u.id = rt.user_id
         WHERE rt.token = $1 AND rt.expires_at > NOW()`,
        [refreshToken]
      );

      const stored = result.rows[0];

      if (!stored)
        return NextResponse.json({ error: "Invalid or expired refresh token" }, { status: 401 });

      // Delete old token (rotation — old one can never be reused)
      await pool.query("DELETE FROM refresh_tokens WHERE token = $1", [refreshToken]);

      // Issue new access token
      const newAccessToken = await signAccessToken({
        id: stored.user_id,
        name: stored.name,
        email: stored.email,
      });

      // Issue new refresh token
      const newRefreshToken = generateRefreshToken();
      await pool.query(
        "INSERT INTO refresh_tokens (user_id, token, expires_at) VALUES ($1, $2, $3)",
        [stored.user_id, newRefreshToken, refreshTokenExpiry()]
      );

      const response = NextResponse.json({ success: true });
      setAccessTokenCookie(response, newAccessToken);
      setRefreshTokenCookie(response, newRefreshToken);

      return response;
    } catch (err) {
      console.error("Refresh error:", err.message);
      return NextResponse.json({ error: err.message }, { status: 500 });
    }
  }

============================================================
FILE: src/app/api/auth/logout/route.js
============================================================

  import { NextResponse } from "next/server";
  import { cookies } from "next/headers";
  import pool from "@/lib/db";

  export async function POST() {
    try {
      const cookieStore = await cookies();
      const refreshToken = cookieStore.get("refresh_token")?.value;

      if (refreshToken)
        await pool.query("DELETE FROM refresh_tokens WHERE token = $1", [refreshToken]);

      const response = NextResponse.json({ message: "Logged out successfully" });
      response.cookies.set("access_token",  "", { httpOnly: true, maxAge: 0, path: "/" });
      response.cookies.set("refresh_token", "", { httpOnly: true, maxAge: 0, path: "/" });

      return response;
    } catch (err) {
      console.error("Logout error:", err.message);
      return NextResponse.json({ error: err.message }, { status: 500 });
    }
  }

============================================================
FILE: src/app/api/auth/me/route.js
============================================================

  import { NextResponse } from "next/server";
  import { getUser } from "@/lib/auth";

  export async function GET() {
    const user = await getUser();
    if (!user)
      return NextResponse.json({ error: "Not authenticated" }, { status: 401 });
    return NextResponse.json(user);
  }

============================================================
FILE: middleware.js  (at project root, not inside src/)
============================================================

  import { NextResponse } from "next/server";
  import { jwtVerify } from "jose";

  const SECRET = new TextEncoder().encode(process.env.JWT_SECRET);

  async function verifyToken(token) {
    try {
      const { payload } = await jwtVerify(token, SECRET);
      return payload;
    } catch {
      return null;
    }
  }

  const protectedRoutes = ["/notes", "/dashboard", "/profile"];
  const authRoutes      = ["/login", "/register"];

  export async function middleware(request) {
    const token = request.cookies.get("access_token")?.value;
    const { pathname } = request.nextUrl;

    const isProtected = protectedRoutes.some(r => pathname.startsWith(r));
    const isAuthRoute = authRoutes.some(r => pathname.startsWith(r));

    if (isProtected) {
      if (!token) return NextResponse.redirect(new URL("/login", request.url));
      const user = await verifyToken(token);
      if (!user) return NextResponse.redirect(new URL("/login", request.url));
    }

    if (isAuthRoute && token) {
      const user = await verifyToken(token);
      if (user) return NextResponse.redirect(new URL("/notes", request.url));
    }

    return NextResponse.next();
  }

  export const config = {
    matcher: ["/notes/:path*", "/dashboard/:path*", "/profile/:path*", "/login", "/register"],
  };

============================================================
FILE: src/components/Navbar.jsx
============================================================

  import Link from "next/link";
  import { getUser } from "@/lib/auth";
  import LogoutButton from "@/components/LogoutButton";

  export default async function Navbar() {
    const user = await getUser();

    return (
      <nav className="bg-white border-b border-gray-200 px-8 py-4 flex items-center gap-8">
        <Link href="/" className="text-xl font-bold text-black">MyApp</Link>
        <Link href="/">Home</Link>
        <Link href="/notes">Notes</Link>

        <div className="ml-auto flex items-center gap-4">
          {user ? (
            <>
              <span className="text-sm text-gray-500">
                Hi, <span className="font-semibold">{user.name}</span>
              </span>
              <LogoutButton />
            </>
          ) : (
            <>
              <Link href="/login">Login</Link>
              <Link href="/register">Register</Link>
            </>
          )}
        </div>
      </nav>
    );
  }

============================================================
FILE: src/components/LogoutButton.jsx
============================================================

  "use client";
  import { useRouter } from "next/navigation";

  export default function LogoutButton() {
    const router = useRouter();

    async function handleLogout() {
      await fetch("/api/auth/logout", { method: "POST" });
      router.push("/login");
      router.refresh();
    }

    return (
      <button onClick={handleLogout} className="text-sm text-red-500 hover:text-red-700">
        Logout
      </button>
    );
  }

---

## 7. Security Checklist

  SECRETS
  [x] JWT_SECRET is at least 64 random characters
  [x] JWT_SECRET generated with: node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
  [x] .env.local is in .gitignore — never committed to git
  [x] Different JWT_SECRET for development and production

  PASSWORDS
  [x] Passwords hashed with bcrypt (cost factor 10+)
  [x] Plain text password never stored or logged
  [x] Minimum password length enforced (6+ characters)

  TOKENS
  [x] Access token is short lived (15 minutes)
  [x] Refresh token is random bytes (not a JWT)
  [x] Refresh token stored in database (can be revoked)
  [x] Refresh token rotation (new token on every refresh)
  [x] Logout deletes refresh token from database

  COOKIES
  [x] httpOnly: true (JavaScript cannot read the cookie)
  [x] secure: true in production (HTTPS only)
  [x] sameSite: "lax" (prevents CSRF attacks)
  [x] Correct maxAge set on both cookies

  MIDDLEWARE
  [x] Protected routes defined in middleware.js
  [x] Auth routes redirect logged-in users away
  [x] Token verified on every request to protected routes

  API ROUTES
  [x] All routes wrapped in try/catch
  [x] Errors return JSON (not empty body)
  [x] Input validation before DB queries
  [x] Parameterized queries ($1, $2) — no SQL injection possible

---

## 8. Common Mistakes to Avoid

MISTAKE 1 — Storing JWT in localStorage
  localStorage can be read by JavaScript → XSS attack steals your token
  FIX: Always use HttpOnly cookies

MISTAKE 2 — Long-lived single token (7 days JWT)
  If stolen → attacker has full access for 7 days, no way to revoke
  FIX: Short access token (15 min) + revocable refresh token in DB

MISTAKE 3 — Weak JWT_SECRET
  Short or dictionary-based secrets can be brute forced
  FIX: 64+ random bytes generated with crypto.randomBytes()

MISTAKE 4 — Storing plain text password
  If DB is stolen → all user passwords exposed
  FIX: Always bcrypt.hash() before saving, bcrypt.compare() to verify

MISTAKE 5 — Not using parameterized queries
  string concatenation in SQL = SQL injection vulnerability
  WRONG: "SELECT * FROM users WHERE email = '" + email + "'"
  RIGHT: "SELECT * FROM users WHERE email = $1", [email]

MISTAKE 6 — Putting secrets in payload
  JWT payload is base64 encoded — not encrypted — anyone can read it
  FIX: Only put non-sensitive data in payload (id, name, email)

MISTAKE 7 — Not rotating refresh tokens
  Reusing refresh tokens → stolen token works forever until expiry
  FIX: Delete old refresh token and issue new one on every refresh call

MISTAKE 8 — Committing .env.local to git
  Anyone with repo access gets your DB password and JWT secret
  FIX: Always have .env.local in .gitignore

---

## 9. npm Packages Used

  npm install pg bcryptjs jose

  pg        → PostgreSQL client for Node.js
  bcryptjs  → Password hashing (pure JS, no native bindings needed)
  jose      → JWT creation and verification (works in Next.js Edge runtime)

---

## 10. Database Tables Required

  -- Run once in your PostgreSQL database
  CREATE TABLE IF NOT EXISTS users (
    id         SERIAL PRIMARY KEY,
    name       VARCHAR(255) NOT NULL,
    email      VARCHAR(255) UNIQUE NOT NULL,
    password   VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  );

  CREATE TABLE IF NOT EXISTS refresh_tokens (
    id         SERIAL PRIMARY KEY,
    user_id    INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token      VARCHAR(512) UNIQUE NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  );

  -- ON DELETE CASCADE means: if a user is deleted,
  -- all their refresh tokens are automatically deleted too

---

Last updated: June 2026
Project: Next.js 16 + PostgreSQL + jose + bcryptjs
