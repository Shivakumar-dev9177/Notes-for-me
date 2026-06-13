# HLS + AES-128 Encryption Architecture
### Language-agnostic design + complete Node.js implementation

---

## 1. Architecture Overview

The goal is: **video content is encrypted at rest in S3, and can only be played by authenticated users inside the application. Downloading a segment URL gives you only an encrypted binary — unplayable without the key. The key is only returned to an authenticated, authorized user.**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          UPLOAD PIPELINE                                │
│                                                                         │
│  Original MP4 ──► FFmpeg ──► .ts segments (AES-128 encrypted)          │
│                      │            │                                     │
│                      │            └──► Upload to S3                    │
│                      │                   org-{orgId}/{assetId}/         │
│                      │                     seg000.ts                   │
│                      │                     seg001.ts  ...               │
│                      │                     playlist.m3u8                │
│                      │                                                  │
│                      └──► AES-128 key (hex, 32 chars)                  │
│                               Encrypted → stored in DB                  │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                          PLAYBACK FLOW                                  │
│                                                                         │
│  Browser (hls.js)                                                       │
│       │                                                                 │
│       │ 1. GET /api/lessons/{id}/hls-manifest                          │
│       │    Authorization: Bearer JWT                                    │
│       │                                                                 │
│       ▼                                                                 │
│  API Server                                                             │
│       │  a. Verify JWT + check enrollment                               │
│       │  b. Download playlist.m3u8 from S3                             │
│       │  c. Rewrite #EXT-X-KEY URI → /hls-key?token=JWT                │
│       │  d. Rewrite each .ts line → presigned S3 URL (2-hour TTL)      │
│       │  e. Return modified manifest inline                             │
│       │                                                                 │
│       │ 2. hls.js sees #EXT-X-KEY URI                                  │
│       │    GET /api/lessons/{id}/hls-key?token=JWT                     │
│       │                                                                 │
│       ▼                                                                 │
│  API Server                                                             │
│       │  a. Validate JWT from ?token= query param                      │
│       │  b. Check enrollment                                            │
│       │  c. Decrypt stored AES key → return raw 16 bytes               │
│       │                                                                 │
│       │ 3. hls.js fetches .ts from presigned S3 URL                   │
│       │    Decrypts in-browser with the 16-byte key                    │
│       │    Plays video                                                  │
│       ▼                                                                 │
│  Video plays in <video> element                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Core Concepts

### 2.1 AES-128 in HLS

- FFmpeg generates a random 16-byte AES-128 key
- Every `.ts` segment is encrypted with that key (CBC mode, random IV per segment)
- The playlist (`.m3u8`) contains a `#EXT-X-KEY` tag that tells the player WHERE to fetch the key
- That URI is the only thing you control — the rest of the decryption is done by hls.js / native HLS

```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-KEY:METHOD=AES-128,URI="https://your-api.com/hls-key?token=JWT",IV=0x00000000...
#EXTINF:6.000000,
seg000.ts
#EXTINF:6.000000,
seg001.ts
...
#EXT-X-ENDLIST
```

### 2.2 Manifest Proxy (the critical piece)

The raw `.m3u8` from S3 contains the original S3 key paths for segments (e.g., `seg000.ts`). These cannot be served directly — CloudFront requires signed URLs, and the segments themselves are encrypted.

The API **rewrites the manifest in memory**:
- `#EXT-X-KEY URI` → your API's key endpoint (with JWT embedded as ?token=)
- Each `.ts` line → a 2-hour presigned S3 URL

The rewritten manifest is returned inline. The client never sees the real S3 structure.

### 2.3 Why `?token=` instead of `Authorization: Bearer` for the key endpoint

hls.js fetches the AES key via an XHR/fetch. You CAN intercept this with `xhrSetup` callback and add a header. But the design also works without that: the token is already embedded in the manifest URI by the manifest proxy, so even native HLS (Safari) can fetch the key without custom headers.

**Both paths are supported:**
- Custom header (hls.js `xhrSetup`) → preferred in web app
- `?token=` in URI → fallback, required for native HLS, and defensive in case header is stripped

### 2.4 Blob URL Pattern (frontend)

The browser's `<video src="...">` cannot add `Authorization` headers. So the frontend:
1. Fetches the manifest via axios (which has the Bearer interceptor) → gets bytes
2. Creates `URL.createObjectURL(blob)` → `blob:https://your-app.com/uuid`
3. Passes the blob URL to `videoRef.current.src` or `hls.loadSource()`

The blob URL is same-origin — no CORS restriction. hls.js then reads the manifest from memory and makes XHR calls for the AES key and segments.

---

## 3. S3 File Structure

```
s3://your-bucket/
  org-{orgId}/
    {assetId}/
      original.mp4            ← source video (optional, can delete after HLS)
      playlist.m3u8            ← HLS manifest (segment paths are relative)
      seg000.ts               ← encrypted segment
      seg001.ts
      ...
      segNNN.ts
```

The S3 key for the playlist is stored in the DB (`lesson_assets.hls_playlist_key`).
Segment paths in the raw `.m3u8` are relative (just `seg000.ts`), so the manifest proxy
derives the segment S3 keys as: `dirname(hls_playlist_key) + "/" + segmentLine`.

---

## 4. Database Schema

```sql
-- lesson_assets table — relevant columns for HLS
ALTER TABLE lesson_assets ADD COLUMN hls_playlist_key   VARCHAR(500) NULL;
ALTER TABLE lesson_assets ADD COLUMN hls_encryption_key VARCHAR(500) NULL;  -- AES key, stored encrypted
ALTER TABLE lesson_assets ADD COLUMN status             VARCHAR(50)  NOT NULL DEFAULT 'Pending';

-- Status values:
--   Pending       → just uploaded, not yet processed
--   Processing    → FFmpeg running
--   HlsPending    → queued for HLS transcoding
--   HlsProcessing → FFmpeg + S3 upload in progress
--   HlsFailed     → transcoding failed
--   Completed     → ready for playback
```

Store the AES key **encrypted at rest** in the DB. Use your app's symmetric encryption
(e.g., AES-256 with a server-side master key from env/KMS) to encrypt the 16-byte HLS key.

---

## 5. Node.js Implementation

### 5.1 Dependencies

```bash
npm install express jsonwebtoken @aws-sdk/client-s3 @aws-sdk/s3-request-presigner \
            fluent-ffmpeg uuid crypto
```

### 5.2 Project Structure

```
src/
  hls/
    transcode.js       ← FFmpeg pipeline
    keystore.js        ← AES key generation + DB encrypt/decrypt
    manifestProxy.js   ← manifest rewrite logic
  routes/
    lessons.js         ← /hls-manifest, /hls-key endpoints
  middleware/
    auth.js            ← JWT verification
  s3.js                ← S3 client + presigned URL helper
```

---

### 5.3 `src/hls/transcode.js` — FFmpeg HLS Transcoding with AES-128

```js
const ffmpeg = require("fluent-ffmpeg");
const crypto = require("crypto");
const fs = require("fs");
const os = require("os");
const path = require("path");
const { uploadFileToS3 } = require("../s3");

/**
 * Transcodes a video file to AES-128 encrypted HLS and uploads to S3.
 *
 * @param {Object} opts
 * @param {string} opts.inputPath   - local path to source video
 * @param {string} opts.s3Prefix    - S3 folder prefix, e.g. "org-5/asset-99"
 * @returns {{ playlistKey: string, hexKey: string }}
 *   playlistKey = S3 key of the .m3u8 file
 *   hexKey      = raw AES-128 key as 32-char hex string (store this encrypted in DB)
 */
async function transcodeToHls({ inputPath, s3Prefix }) {
  const workDir = path.join(os.tmpdir(), `hls-${Date.now()}`);
  fs.mkdirSync(workDir, { recursive: true });

  // 1. Generate AES-128 key (16 bytes = 128 bits)
  const aesKeyBytes = crypto.randomBytes(16);
  const hexKey = aesKeyBytes.toString("hex"); // store this in DB
  const keyFile = path.join(workDir, "enc.key");
  fs.writeFileSync(keyFile, aesKeyBytes); // raw bytes for FFmpeg

  // 2. Create key info file (FFmpeg needs this for HLS encryption)
  //    Format: <key URI>\n<key file path>\n[<IV>]
  //    We use a placeholder URI — the manifest proxy will rewrite it anyway.
  const keyInfoFile = path.join(workDir, "enc.keyinfo");
  fs.writeFileSync(keyInfoFile, [
    "https://placeholder/hls-key", // ← rewritten by manifest proxy at request time
    keyFile,
  ].join("\n"));

  const playlistFile = path.join(workDir, "playlist.m3u8");

  // 3. Run FFmpeg
  await new Promise((resolve, reject) => {
    ffmpeg(inputPath)
      .outputOptions([
        "-c:v copy",           // no re-encode (fast) — or use "libx264" to re-encode
        "-c:a aac",
        "-hls_time 6",         // 6-second segments
        "-hls_playlist_type vod",
        "-hls_segment_filename", path.join(workDir, "seg%03d.ts"),
        "-hls_key_info_file", keyInfoFile,
        "-hls_flags delete_segments", // don't keep old segments on disk
      ])
      .output(playlistFile)
      .on("end", resolve)
      .on("error", reject)
      .run();
  });

  // 4. Upload all files to S3
  const files = fs.readdirSync(workDir).filter(f => f.endsWith(".ts") || f.endsWith(".m3u8"));

  for (const file of files) {
    const s3Key = `${s3Prefix}/${file}`;
    await uploadFileToS3({
      key: s3Key,
      body: fs.createReadStream(path.join(workDir, file)),
      contentType: file.endsWith(".m3u8") ? "application/vnd.apple.mpegurl" : "video/mp2t",
    });
  }

  // 5. Clean up temp dir
  fs.rmSync(workDir, { recursive: true, force: true });

  return {
    playlistKey: `${s3Prefix}/playlist.m3u8`,
    hexKey,
  };
}

module.exports = { transcodeToHls };
```

---

### 5.4 `src/hls/keystore.js` — AES Key Encryption for DB Storage

```js
const crypto = require("crypto");

// MASTER_KEY must be 32 bytes (256 bits) — store in env, never in code
// Generate once: node -e "console.log(crypto.randomBytes(32).toString('hex'))"
const MASTER_KEY = Buffer.from(process.env.HLS_MASTER_KEY, "hex");
const ALGORITHM = "aes-256-gcm";

/**
 * Encrypt the raw HLS key before storing in the database.
 * Returns a base64 string: iv(12):authTag(16):ciphertext
 */
function encryptHlsKey(hexKey) {
  const iv = crypto.randomBytes(12);
  const cipher = crypto.createCipheriv(ALGORITHM, MASTER_KEY, iv);
  const enc = Buffer.concat([cipher.update(hexKey, "utf8"), cipher.final()]);
  const tag = cipher.getAuthTag();
  return Buffer.concat([iv, tag, enc]).toString("base64");
}

/**
 * Decrypt the stored DB value back to the raw hex key.
 */
function decryptHlsKey(stored) {
  const buf = Buffer.from(stored, "base64");
  const iv = buf.subarray(0, 12);
  const tag = buf.subarray(12, 28);
  const enc = buf.subarray(28);
  const decipher = crypto.createDecipheriv(ALGORITHM, MASTER_KEY, iv);
  decipher.setAuthTag(tag);
  return Buffer.concat([decipher.update(enc), decipher.final()]).toString("utf8");
}

module.exports = { encryptHlsKey, decryptHlsKey };
```

---

### 5.5 `src/s3.js` — S3 Client + Presigned URL Helper

```js
const { S3Client, GetObjectCommand, PutObjectCommand } = require("@aws-sdk/client-s3");
const { getSignedUrl } = require("@aws-sdk/s3-request-presigner");

const s3 = new S3Client({
  region: process.env.AWS_REGION,
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
  },
});

const BUCKET = process.env.S3_BUCKET_NAME;

async function downloadS3Object(key) {
  const cmd = new GetObjectCommand({ Bucket: BUCKET, Key: key });
  const res = await s3.send(cmd);
  const chunks = [];
  for await (const chunk of res.Body) chunks.push(chunk);
  return Buffer.concat(chunks).toString("utf8");
}

async function uploadFileToS3({ key, body, contentType }) {
  await s3.send(new PutObjectCommand({
    Bucket: BUCKET,
    Key: key,
    Body: body,
    ContentType: contentType,
  }));
}

async function getPresignedUrl(key, expiresInSeconds = 7200) {
  const cmd = new GetObjectCommand({ Bucket: BUCKET, Key: key });
  return getSignedUrl(s3, cmd, { expiresIn: expiresInSeconds });
}

module.exports = { downloadS3Object, uploadFileToS3, getPresignedUrl };
```

---

### 5.6 `src/hls/manifestProxy.js` — Manifest Rewrite Logic

```js
const { downloadS3Object, getPresignedUrl } = require("../s3");

/**
 * Downloads the raw .m3u8 from S3, rewrites:
 *   - #EXT-X-KEY URI → the API's /hls-key endpoint (with JWT in ?token=)
 *   - each .ts line → a 2-hour presigned S3 URL
 *
 * Returns the rewritten manifest as a string.
 *
 * @param {string} playlistKey  - S3 key of the .m3u8 file
 * @param {string} keyUri       - full URL of the key endpoint (with token embedded)
 */
async function buildPresignedManifest(playlistKey, keyUri) {
  const raw = await downloadS3Object(playlistKey);
  const lines = raw.split("\n");
  const folder = playlistKey.substring(0, playlistKey.lastIndexOf("/") + 1); // "org-5/asset-99/"

  const rewritten = await Promise.all(
    lines.map(async (rawLine) => {
      const line = rawLine.trimEnd();

      // Rewrite #EXT-X-KEY URI
      if (line.startsWith("#EXT-X-KEY:")) {
        return line.replace(/URI="[^"]*"/, `URI="${keyUri}"`);
      }

      // Rewrite .ts segment lines
      if (!line.startsWith("#") && line.endsWith(".ts")) {
        const segKey = folder + line.replace(/^\/+/, "");
        const presigned = await getPresignedUrl(segKey, 7200); // 2-hour TTL
        return presigned;
      }

      return line;
    })
  );

  return rewritten.join("\n");
}

module.exports = { buildPresignedManifest };
```

---

### 5.7 `src/middleware/auth.js` — JWT Middleware

```js
const jwt = require("jsonwebtoken");

const JWT_SECRET = process.env.JWT_SECRET;

/**
 * Standard middleware — sets req.user or returns 401.
 * Use this on routes that require Authorization: Bearer header.
 */
function requireAuth(req, res, next) {
  const header = req.headers["authorization"] ?? "";
  const token = header.startsWith("Bearer ") ? header.slice(7) : null;
  if (!token) return res.status(401).json({ error: "Missing token" });

  try {
    req.user = jwt.verify(token, JWT_SECRET);
    next();
  } catch {
    res.status(401).json({ error: "Invalid or expired token" });
  }
}

/**
 * Flexible token extractor — checks Authorization header OR ?token= query param.
 * Used for endpoints that hls.js / iframes call (can't always set headers).
 */
function extractToken(req) {
  const header = req.headers["authorization"] ?? "";
  if (header.startsWith("Bearer ")) return header.slice(7);
  return req.query.token ?? null;
}

/**
 * Validates a raw token string. Returns the decoded payload or null.
 */
function validateToken(rawToken) {
  try {
    return jwt.verify(rawToken, JWT_SECRET);
  } catch {
    return null;
  }
}

module.exports = { requireAuth, extractToken, validateToken };
```

---

### 5.8 `src/routes/lessons.js` — HLS Manifest + Key Endpoints

```js
const express = require("express");
const router = express.Router();
const { requireAuth, extractToken, validateToken } = require("../middleware/auth");
const { buildPresignedManifest } = require("../hls/manifestProxy");
const { decryptHlsKey } = require("../hls/keystore");
const db = require("../db"); // your DB client

const API_BASE_URL = process.env.API_BASE_URL; // e.g. "https://api.yourdomain.com"

// ─── Helper: check if a user is enrolled in the lesson's curriculum ──────────
async function isEnrolled(userId, lessonId) {
  // Adapt this query to your schema
  const row = await db.query(`
    SELECT 1
    FROM   curriculum_assignments ca
    JOIN   lessons l ON l.curriculum_id = ca.curriculum_id
    WHERE  l.id = $1
      AND  ca.is_active = true
      AND  (ca.user_id = $2 OR ca.group_id IN (
              SELECT group_id FROM user_group_mappings
              WHERE  user_id = $2 AND is_active = true
           ))
    LIMIT 1
  `, [lessonId, userId]);
  return row.rows.length > 0;
}

// ─── Helper: get the lesson asset (with HLS info) ────────────────────────────
async function getLessonAsset(lessonId) {
  // Two-step lookup: LessonAsset.lesson_id FK first, then Lesson.lesson_asset_id fallback
  let row = await db.query(
    `SELECT * FROM lesson_assets WHERE lesson_id = $1 ORDER BY asset_id DESC LIMIT 1`,
    [lessonId]
  );
  if (row.rows.length) return row.rows[0];

  // Fallback — some assets have lesson_id = NULL, linked via lessons.lesson_asset_id
  const lesson = await db.query(
    `SELECT lesson_asset_id FROM lessons WHERE id = $1`,
    [lessonId]
  );
  if (!lesson.rows[0]?.lesson_asset_id) return null;
  const asset = await db.query(
    `SELECT * FROM lesson_assets WHERE asset_id = $1`,
    [lesson.rows[0].lesson_asset_id]
  );
  return asset.rows[0] ?? null;
}


// ─── GET /api/lessons/:lessonId/hls-manifest ─────────────────────────────────
// Requires: Authorization: Bearer JWT
// Returns:  Modified .m3u8 with presigned segment URLs + key URI
router.get("/:lessonId/hls-manifest", requireAuth, async (req, res) => {
  const { lessonId } = req.params;
  const { userId } = req.user;

  const enrolled = await isEnrolled(userId, lessonId);
  if (!enrolled) return res.status(404).json({ error: "Lesson not found or no access" });

  const asset = await getLessonAsset(lessonId);
  if (!asset?.hls_playlist_key)
    return res.status(404).json({ error: "No HLS content for this lesson" });

  // Embed the current JWT in the key URI so hls.js / Safari can fetch it without custom headers
  const rawToken = req.headers["authorization"].slice(7);
  const keyUri = `${API_BASE_URL}/api/lessons/${lessonId}/hls-key?token=${encodeURIComponent(rawToken)}`;

  const manifest = await buildPresignedManifest(asset.hls_playlist_key, keyUri);

  res.setHeader("Content-Type", "application/vnd.apple.mpegurl");
  res.setHeader("Cache-Control", "no-store");
  res.send(manifest);
});


// ─── GET /api/lessons/:lessonId/hls-key ──────────────────────────────────────
// UNAUTHENTICATED route (no middleware) — manual JWT validation from ?token=
// Returns: raw 16 bytes (AES-128 key) as application/octet-stream
router.get("/:lessonId/hls-key", async (req, res) => {
  const { lessonId } = req.params;

  // Token from ?token= OR Authorization header
  const rawToken = extractToken(req);
  if (!rawToken) return res.status(401).json({ error: "Missing token" });

  const payload = validateToken(rawToken);
  if (!payload) return res.status(401).json({ error: "Invalid or expired token" });

  const enrolled = await isEnrolled(payload.userId, lessonId);
  if (!enrolled) return res.status(404).json({ error: "No access" });

  const asset = await getLessonAsset(lessonId);
  if (!asset?.hls_encryption_key)
    return res.status(404).json({ error: "No encryption key found" });

  const hexKey = decryptHlsKey(asset.hls_encryption_key);
  const keyBytes = Buffer.from(hexKey, "hex"); // 16 bytes

  res.setHeader("Content-Type", "application/octet-stream");
  res.setHeader("Cache-Control", "no-store");
  res.send(keyBytes);
});


// ─── GET /api/lessons/:lessonId/stream ───────────────────────────────────────
// For non-HLS content (PDF, audio, image, Word, Excel, PPT)
// Returns: proxied file bytes so the frontend can create a blob: URL
router.get("/:lessonId/stream", requireAuth, async (req, res) => {
  const { lessonId } = req.params;
  const { userId } = req.user;

  const enrolled = await isEnrolled(userId, lessonId);
  if (!enrolled) return res.status(404).json({ error: "No access" });

  const asset = await getLessonAsset(lessonId);
  if (!asset) return res.status(404).json({ error: "Asset not found" });

  const presigned = await getPresignedUrl(asset.s3_key, 300); // 5-minute URL
  // Redirect to presigned URL — browser follows it transparently
  res.redirect(302, presigned);
});


module.exports = router;
```

---

### 5.9 Upload Route — Trigger HLS Transcoding After Upload

```js
const { transcodeToHls } = require("../hls/transcode");
const { encryptHlsKey } = require("../hls/keystore");

// Called after file is saved to S3 (or temp disk)
async function processVideoAsset(assetId, lessonId, orgId, localFilePath) {
  const s3Prefix = `org-${orgId}/asset-${assetId}`;

  // Mark as processing
  await db.query(
    `UPDATE lesson_assets SET status = 'HlsProcessing' WHERE asset_id = $1`,
    [assetId]
  );

  try {
    const { playlistKey, hexKey } = await transcodeToHls({
      inputPath: localFilePath,
      s3Prefix,
    });

    const encryptedKey = encryptHlsKey(hexKey);

    await db.query(`
      UPDATE lesson_assets
      SET    status             = 'Completed',
             hls_playlist_key  = $1,
             hls_encryption_key = $2,
             is_hls            = true
      WHERE  asset_id = $3
    `, [playlistKey, encryptedKey, assetId]);

  } catch (err) {
    await db.query(
      `UPDATE lesson_assets SET status = 'HlsFailed' WHERE asset_id = $1`,
      [assetId]
    );
    throw err;
  }
}
```

---

### 5.10 Frontend — Fetch Manifest as Blob, Pass to hls.js

```js
import Hls from "hls.js";

async function loadHlsVideo(lessonId, videoEl) {
  const token = localStorage.getItem("authToken");

  // 1. Fetch manifest via axios (adds Authorization header automatically)
  const res = await fetch(`/api/lessons/${lessonId}/hls-manifest`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  if (!res.ok) throw new Error(`Manifest fetch failed: ${res.status}`);

  // 2. Create blob: URL from the manifest bytes
  const blob = await res.blob();
  const blobUrl = URL.createObjectURL(blob);

  // 3. Load into hls.js
  if (Hls.isSupported()) {
    const hls = new Hls({
      xhrSetup: (xhr, url) => {
        // Add auth header to key requests (defense-in-depth; ?token= is already in the URI)
        if (url.includes("/hls-key")) {
          xhr.setRequestHeader("Authorization", `Bearer ${token}`);
        }
      },
    });
    hls.loadSource(blobUrl);
    hls.attachMedia(videoEl);
  } else if (videoEl.canPlayType("application/vnd.apple.mpegurl")) {
    // Safari — native HLS, fetches the key using the ?token= in the manifest URI
    videoEl.src = blobUrl;
  }
}
```

---

## 6. SCORM / WebGL Token Auth (Node.js)

Same architecture as the C# version. The problem: iframes cannot add `Authorization` headers.

### 6.1 Generate the player HTML page

```js
// GET /api/lessons/:lessonId/scorm-player.html
// [No auth middleware] — token is read from ?token= query param
router.get("/:lessonId/scorm-player.html", (req, res) => {
  const { lessonId } = req.params;
  const token = req.query.token ?? "";
  const tokenParam = token ? `?token=${encodeURIComponent(token)}` : "";
  const scoUrl = `/api/lessons/${lessonId}/scorm/index.html${tokenParam}`;

  const html = buildScormPlayerHtml(lessonId, scoUrl); // see 6.2 below
  res.setHeader("Content-Type", "text/html; charset=utf-8");
  res.setHeader("Cache-Control", "no-store");
  res.setHeader("X-Frame-Options", "SAMEORIGIN");
  res.send(html);
});
```

### 6.2 SCORM player HTML builder

```js
function buildScormPlayerHtml(lessonId, scoUrl) {
  return `<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>SCORM Player</title>
  <style>body,html{margin:0;height:100%;overflow:hidden} iframe{width:100%;height:100%;border:none}</style>
</head>
<body>
  <iframe src="${scoUrl}" allow="fullscreen"></iframe>
</body>
</html>`;
}
```

### 6.3 SCORM file proxy (landing page auth + anonymous sub-files)

```js
// GET /api/lessons/:lessonId/scorm/*filepath
router.get("/:lessonId/scorm/*", async (req, res) => {
  const { lessonId } = req.params;
  const filePath = req.params[0] || "index.html";
  const isLanding = filePath === "index.html" || filePath === "" || filePath === "/";

  if (isLanding) {
    // Validate JWT only for the landing page
    const rawToken = req.query.token ?? (req.headers["authorization"] ?? "").replace("Bearer ", "");
    if (!rawToken) return res.status(401).json({ error: "Authentication required" });

    const payload = validateToken(rawToken);
    if (!payload) return res.status(401).json({ error: "Invalid token" });

    const enrolled = await isEnrolled(payload.userId, lessonId);
    if (!enrolled) return res.status(404).json({ error: "No access" });
  }
  // Sub-files (JS, CSS, wasm, images, fonts) — no auth check
  // Browsers cannot add Authorization headers to these relative sub-resource requests

  // Look up the asset S3 folder
  const asset = await getLessonAsset(lessonId);
  if (!asset) return res.status(404).json({ reason: "no_asset" });

  const s3Folder = asset.s3_key.replace(/\/[^/]+$/, ""); // strip filename to get folder
  const s3Key = `${s3Folder}/${filePath}`;

  const presigned = await getPresignedUrl(s3Key, 300);
  res.redirect(302, presigned);
});
```

---

## 7. Security Model Summary

| Threat | Mitigation |
|---|---|
| Download segment URL (`.ts`) | Encrypted binary — no key = unplayable |
| Guess/share segment URL | Presigned URL expires in 2 hours; no value without the key |
| Steal the AES key URL | Key endpoint validates JWT — must be an enrolled user |
| Share JWT with another user | JWT expiry is your standard session TTL (typically 15 min–24 h) |
| Download manifest and replay | Manifest is generated fresh per request; segment presigned URLs have TTL |
| Access SCORM bundle directly from S3 | S3 bucket is not public; all access via API proxy |
| Guess SCORM S3 path | Path includes org-ID + UUID asset-ID — not guessable |
| Sub-file served without auth | Content is training material, not sensitive; landing-page gate already verified identity |

### What this does NOT protect against
- **Screen recording** — browser-side, no server-side solution exists
- **A user with a valid JWT sharing it with another user** — standard session management problem
- **A user with a valid JWT downloading and decrypting segments themselves** — they have legitimate access; you cannot prevent cryptographic access by an authorized user

---

## 8. Environment Variables

```bash
# AWS
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
S3_BUCKET_NAME=your-bucket

# App
API_BASE_URL=https://api.yourdomain.com
JWT_SECRET=your-jwt-secret-min-256-bits

# HLS key encryption (generate once: node -e "require('crypto').randomBytes(32).toString('hex')")
HLS_MASTER_KEY=64-char-hex-string
```

---

## 9. Quick-Start Checklist

- [ ] Install FFmpeg on server (`apt-get install -y ffmpeg` or Homebrew on macOS)
- [ ] Add `HLS_MASTER_KEY`, `API_BASE_URL`, `JWT_SECRET` to environment
- [ ] Add `hls_playlist_key`, `hls_encryption_key`, `status`, `is_hls` columns to `lesson_assets` table
- [ ] Copy `transcode.js`, `keystore.js`, `manifestProxy.js`, `s3.js` into your project
- [ ] Wire upload handler to call `processVideoAsset()` after file is saved
- [ ] Add `/hls-manifest` and `/hls-key` routes to your lesson router
- [ ] Add `/scorm-player.html` and `/scorm/*` routes
- [ ] Frontend: replace `<video src="cdn-url">` with the blob-URL pattern
- [ ] Frontend: pass `isHls` field from API response to the player component
- [ ] Test: confirm downloading a `.ts` presigned URL returns an unplayable encrypted binary
- [ ] Test: confirm `/hls-key` with an invalid JWT returns 401
- [ ] Test: confirm a non-enrolled user cannot load the manifest or key
