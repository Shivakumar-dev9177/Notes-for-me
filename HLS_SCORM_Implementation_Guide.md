# HLS + AES-128 Encryption & Secure Content Delivery — Implementation Guide

**Purpose:** Technical reference for replicating all changes made to add HLS video encryption, secure SCORM/WebGL delivery, and authenticated PDF/Office streaming.

---

## 1. Overview of What Was Built

| Feature | What it does |
|---|---|
| HLS + AES-128 video encryption | Videos are transcoded to HLS (`.m3u8` + `.ts` segments) by FFmpeg. Each segment is AES-128 encrypted. The decryption key is only returned to authenticated users |
| Manifest proxy | Backend downloads the `.m3u8` from S3, rewrites segment URLs to presigned S3 URLs (2-hour expiry), embeds the JWT in the `#EXT-X-KEY` URI, and returns the modified manifest inline |
| SCORM/WebGL token auth | Iframes cannot send `Authorization` headers. The JWT is passed as `?token=` in the iframe `src` URL. The endpoint validates it for the landing page (`index.html`) and serves sub-files anonymously |
| PDF blob streaming | PDFViewer strips auth headers for cross-origin URLs. All content (video, PDF, audio, image) is now fetched via axios (which adds Bearer token) and returned as a same-origin `blob:` URL |
| Office viewer (Excel/PPT/Word) | Backend decrypts the file in memory, uploads a temporary unencrypted copy to S3, generates a 15-minute presigned URL, and returns a Microsoft Office Online embed URL |
| Asset lookup fallback | `LessonAsset.lesson_id` is nullable on some rows. Added a fallback to `Lesson.lesson_asset_id` (the Lesson table's direct FK) so user SCORM proxy finds assets the same way the admin proxy does |

---

## 2. Prerequisites / Infrastructure

### FFmpeg (for HLS transcoding)
- **EC2 / Linux:** `sudo apt-get install -y ffmpeg`
- **Windows dev:** Install from [gyan.dev](https://www.gyan.dev/ffmpeg/builds/), get the full build, note the full path to `ffmpeg.exe`

### appsettings.json additions
```json
{
  "VideoStream": {
    "ApiBaseUrl": "https://your-api-domain.com",
    "FfmpegPath": "/usr/bin/ffmpeg"
  }
}
```
- `ApiBaseUrl` — used to build the `#EXT-X-KEY` URI embedded in the HLS manifest (must be the public API URL)
- `FfmpegPath` — absolute path to the FFmpeg binary

---

## 3. Backend Changes

### 3.1 `DTOs/Response/LessonResponseDto.cs`

Add `IsHls` to `LessonAssetSummaryDto`:

```csharp
public class LessonAssetSummaryDto
{
    // ... existing fields ...
    public bool IsHls { get; set; }   // ← ADD THIS
}
```

---

### 3.2 `Services/LessonService.cs`

#### 3.2.1 — `allowedStatuses` (TWO locations)
Anywhere the service filters assets by status (e.g., when building the lesson response), add HLS statuses so HLS-pending lessons are not rejected:

```csharp
// Before
var allowedStatuses = new[] { "Completed", "Processing" };

// After
var allowedStatuses = new[] { "Completed", "Processing", "HlsPending", "HlsProcessing" };
```

#### 3.2.2 — `BuildCdnUrl` method
Prevent CloudFront unsigned URLs being returned for video assets in any HLS state (they'd get a 403):

```csharp
private string? BuildCdnUrl(LessonAsset? asset)
{
    if (asset == null) return null;

    var isVideo = string.Equals(asset.AssetType, "video", StringComparison.OrdinalIgnoreCase)
               || (asset.MimeType ?? "").StartsWith("video/", StringComparison.OrdinalIgnoreCase);

    // Return null for HLS videos — the manifest proxy endpoint handles these
    if (isVideo && (asset.HlsPlaylistKey != null
                 || asset.Status is "HlsPending" or "HlsProcessing" or "HlsFailed"))
        return null;

    if (!string.IsNullOrWhiteSpace(asset.CdnUrl)) return asset.CdnUrl;
    return _cdn.Build(asset.S3Key);
}
```

#### 3.2.3 — Map `IsHls` in all three `LessonAssetSummaryDto` mappings
In every place you build a `LessonAssetSummaryDto` from a `LessonAsset`, add:

```csharp
IsHls = asset.HlsPlaylistKey != null,
```

---

### 3.3 `Controllers/LessonController.cs` (Admin)

> This controller is at route `api/admin/contenthub`.

#### 3.3.1 — Admin HLS manifest endpoint

```csharp
/// GET /api/admin/contenthub/lessons/{id}/hls-manifest
[HttpGet("lessons/{id}/hls-manifest")]
[Authorize(Roles = "Admin,Trainer")]
public async Task<IActionResult> GetAdminHlsManifest(int id)
{
    var lesson = await _lessonService.GetLessonByIdAsync(id);
    if (lesson == null) return NotFound();

    var asset = lesson.PrimaryAsset;
    if (asset == null || string.IsNullOrEmpty(asset.HlsPlaylistKey))
        return NotFound(new { success = false, message = "No HLS playlist found" });

    var token = Request.Headers["Authorization"].ToString().Replace("Bearer ", "").Trim();
    var keyUri = $"{Request.Scheme}://{Request.Host}/api/admin/contenthub/lessons/{id}/hls-key?token={Uri.EscapeDataString(token)}";
    return await BuildPresignedManifestAsync(asset.HlsPlaylistKey, keyUri);
}
```

#### 3.3.2 — Admin HLS key endpoint

```csharp
/// GET /api/admin/contenthub/lessons/{id}/hls-key
[HttpGet("lessons/{id}/hls-key")]
[AllowAnonymous]
public async Task<IActionResult> GetAdminHlsKey(int id, [FromQuery] string? token = null)
{
    var rawToken = token ?? Request.Headers["Authorization"].ToString().Replace("Bearer ", "").Trim();
    if (string.IsNullOrEmpty(rawToken))
        return Unauthorized();

    var jwtHelper = HttpContext.RequestServices.GetRequiredService<JwtHelper>();
    var principal = jwtHelper.ValidateToken(rawToken);
    if (principal == null) return Unauthorized();

    var role = principal.FindFirst(ClaimTypes.Role)?.Value ?? "";
    if (!role.Equals("Admin", StringComparison.OrdinalIgnoreCase) &&
        !role.Equals("Trainer", StringComparison.OrdinalIgnoreCase))
        return Forbid();

    // Fetch the asset and return the raw 16-byte AES key
    var asset = /* query LessonAssets where LessonId == id, active */;
    if (asset == null || string.IsNullOrEmpty(asset.HlsEncryptionKey))
        return NotFound();

    var enc = HttpContext.RequestServices.GetRequiredService<IEncryptionService>();
    var hexKey = enc.DecryptPassword(asset.HlsEncryptionKey);
    var keyBytes = Convert.FromHexString(hexKey);
    return File(keyBytes, "application/octet-stream");
}
```

#### 3.3.3 — Admin SCORM player page
Change to embed JWT in the SCO URL:

```csharp
[HttpGet("lessons/{id}/scorm-player.html")]
[AllowAnonymous]
public IActionResult ScormPlayerPage(int id, [FromQuery] string? token = null)
{
    var rawToken = token ?? Request.Headers["Authorization"].ToString().Replace("Bearer ", "").Trim();
    var tokenParam = string.IsNullOrEmpty(rawToken) ? "" : $"?token={Uri.EscapeDataString(rawToken)}";
    var scoUrl = $"/api/admin/contenthub/lessons/{id}/scorm/index.html{tokenParam}";
    var html = Helpers.ScormPlayerHtml.Build(id, scoUrl);

    Response.Headers["Cache-Control"] = "no-store";
    Response.Headers["Content-Security-Policy"] =
        "frame-ancestors 'self' https://*.yourdomain.com";
    return Content(html, "text/html; charset=utf-8");
}
```

#### 3.3.4 — Admin SCORM/WebGL proxy
Change `[Authorize]` to `[AllowAnonymous]`, validate token only for `index.html`:

```csharp
[AllowAnonymous]
[HttpGet("lessons/{id}/scorm/{*filePath}")]
[HttpHead("lessons/{id}/scorm/{*filePath}")]
public async Task<IActionResult> ProxyScormFile(int id, string? filePath,
    [FromQuery] string? token = null)
{
    var prelimPath = string.IsNullOrEmpty(filePath) ? "index.html" : filePath;
    bool isLanding = prelimPath.Equals("index.html", StringComparison.OrdinalIgnoreCase)
                  || string.IsNullOrWhiteSpace(prelimPath);

    if (isLanding)
    {
        var rawToken = token ?? Request.Headers["Authorization"].ToString().Replace("Bearer ", "").Trim();
        if (string.IsNullOrEmpty(rawToken)) return Unauthorized();

        var jwtHelper = HttpContext.RequestServices.GetRequiredService<JwtHelper>();
        var principal = jwtHelper.ValidateToken(rawToken);
        if (principal == null) return Unauthorized();

        var role = principal.FindFirst(ClaimTypes.Role)?.Value ?? "";
        if (!role.Equals("Admin", StringComparison.OrdinalIgnoreCase) &&
            !role.Equals("Trainer", StringComparison.OrdinalIgnoreCase))
            return Forbid();
    }

    // ... rest of existing file-serving logic unchanged ...
}
```

#### 3.3.5 — Shared `BuildPresignedManifestAsync` helper
Add to `LessonController`:

```csharp
private async Task<IActionResult> BuildPresignedManifestAsync(string playlistS3Key, string keyUri)
{
    await using var stream = await _awsS3Service.DownloadFileAsync(playlistS3Key);
    if (stream == null) return NotFound(new { success = false, message = "HLS playlist not found" });

    using var reader = new StreamReader(stream);
    var m3u8 = await reader.ReadToEndAsync();
    var lines = m3u8.Split('\n');
    var sb = new System.Text.StringBuilder();

    foreach (var raw in lines)
    {
        var line = raw.TrimEnd('\r');

        // Rewrite #EXT-X-KEY URI to our authenticated key endpoint
        if (line.StartsWith("#EXT-X-KEY:", StringComparison.OrdinalIgnoreCase))
        {
            var rewritten = System.Text.RegularExpressions.Regex.Replace(
                line,
                @"URI=""[^""]*""",
                $"URI=\"{keyUri}\"");
            sb.AppendLine(rewritten);
            continue;
        }

        // Rewrite .ts segment lines to 2-hour presigned S3 URLs
        if (!line.StartsWith("#") && line.EndsWith(".ts", StringComparison.OrdinalIgnoreCase))
        {
            var folder = playlistS3Key.Contains('/')
                ? playlistS3Key[..(playlistS3Key.LastIndexOf('/') + 1)]
                : "";
            var segKey = folder + line.TrimStart('/');
            var presigned = await _awsS3Service.GeneratePreSignedDownloadUrlAsync(segKey, 120);
            sb.AppendLine(presigned.Success && !string.IsNullOrEmpty(presigned.Url)
                ? presigned.Url
                : line);
            continue;
        }

        sb.AppendLine(line);
    }

    var bytes = System.Text.Encoding.UTF8.GetBytes(sb.ToString());
    return File(bytes, "application/vnd.apple.mpegurl");
}
```

---

### 3.4 `Controllers/UserControllers/UserLessonController.cs`

> This controller is at route `api/user`, class-level `[Authorize]`.

#### 3.4.1 — User HLS manifest endpoint

```csharp
/// GET /api/user/lessons/{lessonId}/hls-manifest
[HttpGet("lessons/{lessonId}/hls-manifest")]
[Authorize]
public async Task<IActionResult> GetHlsManifest(int lessonId)
{
    // 1. Verify user identity and enrollment (same pattern as /stream)
    var userIdClaim = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    if (!int.TryParse(userIdClaim, out int userId)) return Unauthorized();

    // 2. Access check — user must be enrolled in the curriculum
    var hasAccess = /* same CurriculumAssignments check as /stream endpoint */;
    if (!hasAccess) return NotFound();

    // 3. Find asset with HlsPlaylistKey
    var asset = await _context.LessonAssets
        .Where(a => a.LessonId == lessonId && a.IsActive && a.HlsPlaylistKey != null)
        .FirstOrDefaultAsync();
    if (asset == null) return NotFound(new { success = false, message = "No HLS content" });

    var rawToken = Request.Headers["Authorization"].ToString().Replace("Bearer ", "").Trim();
    var keyUri = $"{Request.Scheme}://{Request.Host}/api/user/lessons/{lessonId}/hls-key?token={Uri.EscapeDataString(rawToken)}";
    return await BuildPresignedManifestAsync(asset.HlsPlaylistKey, keyUri);
}
```

#### 3.4.2 — User HLS key endpoint

```csharp
/// GET /api/user/lessons/{lessonId}/hls-key
[HttpGet("lessons/{lessonId}/hls-key")]
[AllowAnonymous]
public async Task<IActionResult> GetHlsKey(int lessonId, [FromQuery] string? token = null)
{
    var rawToken = token ?? Request.Headers["Authorization"].ToString().Replace("Bearer ", "").Trim();
    if (string.IsNullOrEmpty(rawToken)) return Unauthorized();

    var jwtHelper = HttpContext.RequestServices.GetRequiredService<JwtHelper>();
    var principal = jwtHelper.ValidateToken(rawToken);
    if (principal == null) return Unauthorized();

    var userIdClaim = principal.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    if (!int.TryParse(userIdClaim, out int userId)) return Unauthorized();

    // Enrollment check (same as /stream)
    var hasAccess = /* CurriculumAssignments check */;
    if (!hasAccess) return NotFound();

    var asset = await _context.LessonAssets
        .Where(a => a.LessonId == lessonId && a.IsActive)
        .FirstOrDefaultAsync();
    if (asset == null || string.IsNullOrEmpty(asset.HlsEncryptionKey)) return NotFound();

    var enc = HttpContext.RequestServices.GetRequiredService<IEncryptionService>();
    var hexKey = enc.DecryptPassword(asset.HlsEncryptionKey);
    var keyBytes = Convert.FromHexString(hexKey);
    return File(keyBytes, "application/octet-stream");
}
```

#### 3.4.3 — User SCORM player page
Embed the JWT in the SCO URL:

```csharp
[HttpGet("lessons/{lessonId}/scorm-player.html")]
[AllowAnonymous]
public IActionResult ScormPlayerPage(int lessonId, [FromQuery] string? token = null)
{
    var rawToken = token ?? Request.Headers["Authorization"].ToString().Replace("Bearer ", "").Trim();
    var tokenParam = string.IsNullOrEmpty(rawToken) ? "" : $"?token={Uri.EscapeDataString(rawToken)}";
    var scoUrl = $"/api/user/lessons/{lessonId}/scorm/index.html{tokenParam}";
    var html = Helpers.ScormPlayerHtml.Build(lessonId, scoUrl);

    ApplyEmbedHeaders();
    Response.Headers["Cache-Control"] = "no-store";
    return Content(html, "text/html; charset=utf-8");
}
```

#### 3.4.4 — User SCORM/WebGL proxy
Change to `[AllowAnonymous]`, validate token only for `index.html`, fall back to `Lesson.lesson_asset_id` for asset lookup:

```csharp
[AllowAnonymous]
[HttpGet("lessons/{lessonId}/scorm/{*filePath}")]
[HttpHead("lessons/{lessonId}/scorm/{*filePath}")]
public async Task<IActionResult> ProxyScormSubFile(int lessonId, string? filePath,
    [FromQuery] string? token = null)
{
    var prelimPath = string.IsNullOrEmpty(filePath) ? "index.html" : filePath;
    bool isLanding = prelimPath.Equals("index.html", StringComparison.OrdinalIgnoreCase)
                  || string.IsNullOrWhiteSpace(prelimPath);

    // ── Auth only for the landing page ─────────────────────────────────────
    if (isLanding)
    {
        var rawToken = token ?? Request.Headers["Authorization"].ToString().Replace("Bearer ", "").Trim();
        if (string.IsNullOrEmpty(rawToken)) return Unauthorized();

        var jwtHelper = HttpContext.RequestServices.GetRequiredService<JwtHelper>();
        var principal = jwtHelper.ValidateToken(rawToken);
        if (principal == null) return Unauthorized();

        var userIdClaim = principal.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (!int.TryParse(userIdClaim, out int userId)) return Unauthorized();

        // Same lesson + enrollment check as /stream
        var lessonRow = await (
            from l in _context.Lessons
            join c in _context.Curricula on l.CurriculumId equals c.Id
            where l.Id == lessonId && l.IsActive && c.IsActive
            select new { Lesson = l, Curriculum = c }
        ).FirstOrDefaultAsync();
        if (lessonRow == null) return NotFound();

        var userGroupIds = await _context.UserGroupMappings
            .Where(u => u.UserId == userId && u.IsActive)
            .Select(u => u.GroupId).ToListAsync();

        var hasAccess = await _context.CurriculumAssignments.AnyAsync(ca =>
            ca.IsActive && ca.CurriculumId == lessonRow.Curriculum.Id &&
            ((ca.UserId.HasValue && ca.UserId.Value == userId) ||
             (ca.GroupId.HasValue && userGroupIds.Contains(ca.GroupId.Value))));
        if (!hasAccess) return NotFound();
    }

    // ── Asset lookup — try LessonId FK, then Lesson.lesson_asset_id FK ────
    var asset = await _context.LessonAssets
        .Where(a => a.LessonId == lessonId)
        .OrderByDescending(a => a.AssetId)
        .FirstOrDefaultAsync();

    if (asset == null)
    {
        // Some assets have LessonId = null and are linked only via Lesson.lesson_asset_id
        var lessonAssetId = await _context.Lessons
            .Where(l => l.Id == lessonId)
            .Select(l => l.LessonAssetId)
            .FirstOrDefaultAsync();

        if (lessonAssetId.HasValue)
            asset = await _context.LessonAssets.FindAsync(lessonAssetId.Value);
    }

    if (asset == null) return NotFound(new { reason = "no_asset" });

    // ... rest of existing S3 file-serving logic (status checks, baseFolder, HEAD/GET) unchanged ...
}
```

#### 3.4.5 — `GetOfficeViewerUrl` — expand accepted asset types

```csharp
// Before
if (t != "ppt" && t != "excel" && t != "word" && t != "doc")
    return BadRequest(...);

// After
var isOfficeType = t is "ppt" or "pptx" or "excel" or "xls" or "xlsx"
                      or "word" or "doc" or "docx";
if (!isOfficeType)
    return BadRequest(...);
```

#### 3.4.6 — Add `IsHls` to `GetLessonById` response

In the `GetLessonById` action, include `IsHls` in the anonymous return object:

```csharp
return Ok(new
{
    success = true,
    // ... existing fields ...
    lessonDetails.CdnUrl,
    lessonDetails.IsHls,   // ← ADD THIS
    // ...
});
```

#### 3.4.7 — Shared `BuildPresignedManifestAsync` helper
Add the **exact same** helper to `UserLessonController` as in `LessonController` (section 3.3.5). It is identical — just a private method on both controllers.

---

## 4. Frontend Changes

### 4.1 `src/utils/api.js`

#### Admin `lessonsAPI` — add token to SCORM URLs

```js
getScormUrl: (lessonId, entryPath = "index.html") => {
  const safePath = String(entryPath || "index.html").replace(/^\/+/, "");
  const base = `${apiClient.defaults.baseURL}/admin/contenthub/lessons/${lessonId}/scorm/${safePath}`;
  const token = localStorage.getItem("authToken");
  return token ? `${base}?token=${encodeURIComponent(token)}` : base;
},

getScormPlayerUrl: (lessonId) => {
  const base = `${apiClient.defaults.baseURL}/admin/contenthub/lessons/${lessonId}/scorm-player.html`;
  const token = localStorage.getItem("authToken");
  return token ? `${base}?token=${encodeURIComponent(token)}` : base;
},

// Admin HLS manifest
getHlsManifest: (lessonId) =>
  apiClient.get(`/admin/contenthub/lessons/${lessonId}/hls-manifest`),
```

#### User `userCourseLibraryAPI` — fix SCORM URL + add HLS

```js
// IMPORTANT: must point to /user/lessons/, NOT /admin/contenthub/
getScormPlayerUrl: (lessonId) => {
  const base = `${apiClient.defaults.baseURL}/user/lessons/${lessonId}/scorm-player.html`;
  const token = localStorage.getItem("authToken");
  return token ? `${base}?token=${encodeURIComponent(token)}` : base;
},

getScormUrl: (lessonId) => {
  const base = `${apiClient.defaults.baseURL}/user/lessons/${lessonId}/scorm/index.html`;
  const token = localStorage.getItem("authToken");
  return token ? `${base}?token=${encodeURIComponent(token)}` : base;
},

// HLS manifest — fetched as blob so hls.js can load it without custom headers
getHlsManifestUrl: (lessonId, config = {}) =>
  apiClient.get(`/user/lessons/${lessonId}/hls-manifest`, config),

// Stream as blob (video, PDF, audio, image — all content behind auth)
getStreamBlobUrl: async (lessonId) => {
  const token = localStorage.getItem("authToken");
  const url = `${apiClient.defaults.baseURL}/user/lessons/${lessonId}/stream`;
  const resp = await fetch(url, {
    headers: token ? { Authorization: `Bearer ${token}` } : {}
  });
  if (!resp.ok) throw new Error(`Stream fetch ${lessonId} failed: ${resp.status}`);
  const blob = await resp.blob();
  return URL.createObjectURL(blob);
},
```

---

### 4.2 `src/pages/User/CourseLibrary/Preview/index.jsx`

#### `resolveLessonUrl` — HLS video: fetch manifest as blob

```js
// For HLS videos, fetch the proxied manifest and create a blob: URL
if (ct === "video" && (detail?.isHls ?? detail?.is_hls)) {
  try {
    const res = await userCourseLibraryAPI.getHlsManifestUrl(id, { responseType: 'blob' });
    const blob = new Blob([res.data], { type: 'application/vnd.apple.mpegurl' });
    const blobUrl = URL.createObjectURL(blob);
    streamBlobUrlRef.current = blobUrl;
    return blobUrl;
  } catch (err) {
    console.error("hls-manifest fetch failed", err);
    return cdnUrl || null;
  }
}
```

#### `resolveLessonUrl` — all content types use blob fetch (removes PDF exclusion)

```js
// Remove the old `ct !== "pdf"` guard — PDF, video, audio, image all need blob
const hasLocalToken = typeof window !== "undefined" && !!window.localStorage?.getItem("authToken");

if (hasLocalToken) {
  try {
    revokeStreamBlobUrl();
    const blobUrl = await userCourseLibraryAPI.getStreamBlobUrl(id);
    streamBlobUrlRef.current = blobUrl;
    return blobUrl;
  } catch (err) {
    console.error("stream blob fetch failed", err);
  }
}
return userCourseLibraryAPI.getStreamUrl(id);
```

#### Both `setCurrentLesson` calls — add `isHls`

In BOTH places where `setCurrentLesson({...})` is called (initial load and lesson-click handler), add:

```js
setCurrentLesson({
  id: resolvedId,
  title: detail.title,
  type: detail.contentType ?? detail.content_type,
  url: resolvedUrl,
  isHls: detail.isHls ?? detail.is_hls ?? false,   // ← ADD THIS
  // ... other fields ...
});
```

---

### 4.3 `src/components/User/Course/ContentViewer.jsx`

#### HLS detection — check `lesson.isHls` (blob: URLs don't contain `.m3u8`)

```js
const isHlsUrl = isVideo && (
  lesson?.isHls ||
  (typeof lesson?.url === "string" && lesson.url.includes(".m3u8"))
);
```

#### hls.js setup effect

```js
useEffect(() => {
  if (!isHlsUrl || !videoRef.current) return;

  const token = localStorage.getItem("authToken");

  if (Hls.isSupported()) {
    const hls = new Hls({
      xhrSetup: (xhr, url) => {
        // Key requests go to our API — attach JWT (also embedded in ?token= for safety)
        if (url.includes("/hls-key")) {
          xhr.setRequestHeader("Authorization", `Bearer ${token}`);
        }
      },
    });
    hls.loadSource(lesson.url);   // lesson.url is a blob: URL containing the manifest
    hls.attachMedia(videoRef.current);
    hlsRef.current = hls;
    return () => { hls.destroy(); hlsRef.current = null; };
  } else if (videoRef.current.canPlayType("application/vnd.apple.mpegurl")) {
    videoRef.current.src = lesson.url;   // Safari native HLS
  }
}, [lesson?.url, isHlsUrl]);
```

---

### 4.4 `src/pages/Admin/ContentHub/ContentPreview/index.jsx`

#### Load ALL content types via stream blob (not raw CloudFront URL)

The admin preview should fetch content via the authenticated `/stream` endpoint for all file types (video, PDF, audio, image, PPT, Excel) because CloudFront requires signed URLs and unsigned requests get 403:

```js
useEffect(() => {
  if (!asset || !id) return;
  if (lesson?.externalUrl) return;

  const ct = (asset.assetType ?? asset.fileType ?? '').toLowerCase();

  // HLS video
  if (ct === 'video' && asset.isHls) {
    lessonsAPI.getHlsManifest(id, { responseType: 'blob' })
      .then(res => {
        const blob = new Blob([res.data], { type: 'application/vnd.apple.mpegurl' });
        setHlsManifestUrl(URL.createObjectURL(blob));
      })
      .catch(err => console.error('HLS manifest fetch failed', err));
    return;
  }

  // All other uploadable content types
  const streamableTypes = ['video', 'pdf', 'audio', 'image', 'ppt', 'pptx', 'excel'];
  if (!streamableTypes.includes(ct)) return;

  lessonsAPI.fetchStreamBlob(id)
    .then(res => {
      const mimeType = getMimeForType(ct); // e.g. 'application/pdf'
      const blob = new Blob([res.data], { type: mimeType });
      setStreamBlobUrl(URL.createObjectURL(blob));
    })
    .catch(err => console.error('Stream blob fetch failed', err));

}, [asset?.assetId, asset?.isHls, asset?.status, lesson?.externalUrl, id]);
```

---

## 5. How the Full HLS Flow Works (End-to-End)

```
Upload video
    ↓
HLS transcoding (FFmpeg) → uploads N .ts segments + playlist.m3u8 to S3
    ↓
AES-128 key generated → stored encrypted in LessonAsset.HlsEncryptionKey
    ↓
HlsPlaylistKey stored in LessonAsset.HlsPlaylistKey

Playback request (user):
    ↓
Frontend: GET /api/user/lessons/{id}/hls-manifest  (with Bearer token)
    ↓
Backend: downloads playlist.m3u8 from S3
         rewrites #EXT-X-KEY URI → /hls-key?token=JWT
         rewrites each .ts line → presigned S3 URL (2-hour expiry)
         returns modified manifest as application/vnd.apple.mpegurl
    ↓
Frontend: creates blob: URL from manifest bytes
         passes blob URL to hls.js
    ↓
hls.js: loads manifest from blob: URL (same-origin, no CORS issue)
         fetches AES key from /hls-key?token=JWT → returns raw 16 bytes
         fetches .ts segments from presigned S3 URLs
         decrypts segments in browser → plays video
```

---

## 6. How SCORM/WebGL Auth Flow Works

```
Frontend generates iframe src:
  /api/user/lessons/{id}/scorm-player.html?token=JWT
    ↓
ScormPlayerPage [AllowAnonymous]:
  reads ?token= from query
  embeds token in SCO URL:
    /api/user/lessons/{id}/scorm/index.html?token=JWT
  returns HTML page that iframes the SCO URL
    ↓
Browser loads SCO iframe:
  GET /api/user/lessons/{id}/scorm/index.html?token=JWT
    ↓
ProxyScormSubFile [AllowAnonymous]:
  isLanding = true → validates ?token= JWT, checks enrollment
  serves index.html from S3
    ↓
SCO loads sub-files (relative URLs, no token):
  GET /api/user/lessons/{id}/scorm/Build/main.js
    ↓
ProxyScormSubFile:
  isLanding = false → no auth check (browsers can't add auth headers to sub-resource requests)
  serves file from S3 directly
```

---

## 7. Asset Lookup: Why Two Paths Are Needed

The `LessonAsset.lesson_id` column is **nullable**. Some assets are linked to lessons via the `Lesson.lesson_asset_id` FK (the Lesson row holds a pointer to its primary asset), with `LessonAsset.lesson_id = NULL`.

**Always implement asset lookup as two steps:**

```csharp
// Step 1: try the asset-side FK
var asset = await _context.LessonAssets
    .Where(a => a.LessonId == lessonId)
    .OrderByDescending(a => a.AssetId)
    .FirstOrDefaultAsync();

// Step 2: fall back to the lesson-side FK
if (asset == null)
{
    var lessonAssetId = await _context.Lessons
        .Where(l => l.Id == lessonId)
        .Select(l => l.LessonAssetId)
        .FirstOrDefaultAsync();

    if (lessonAssetId.HasValue)
        asset = await _context.LessonAssets.FindAsync(lessonAssetId.Value);
}
```

The admin `LessonService.GetLessonByIdAsync` already follows this pattern internally. Any user endpoint that serves lesson content must replicate it.

---

## 8. Quick Checklist for Deployment

- [ ] Install FFmpeg on server (`sudo apt-get install -y ffmpeg`)
- [ ] Add `VideoStream.ApiBaseUrl` and `VideoStream.FfmpegPath` to `appsettings.json` (or environment secrets)
- [ ] `LessonAssetSummaryDto` — add `IsHls` field
- [ ] `LessonService.BuildCdnUrl` — return null for video in HLS states
- [ ] `LessonService` — add `"HlsPending"`, `"HlsProcessing"` to all `allowedStatuses` arrays
- [ ] `LessonController` — add `/hls-manifest`, `/hls-key` endpoints + shared `BuildPresignedManifestAsync`
- [ ] `LessonController` — `ScormPlayerPage` embeds `?token=` in SCO URL
- [ ] `LessonController` — `ProxyScormFile` change to `[AllowAnonymous]` with token check for landing page
- [ ] `UserLessonController` — add `/hls-manifest`, `/hls-key` endpoints + same `BuildPresignedManifestAsync`
- [ ] `UserLessonController` — `ScormPlayerPage` embeds `?token=` in SCO URL
- [ ] `UserLessonController` — `ProxyScormSubFile` change to `[AllowAnonymous]`, token check for landing page, two-step asset lookup
- [ ] `UserLessonController.GetLessonById` — include `IsHls` in response
- [ ] `UserLessonController.GetOfficeViewerUrl` — add `xls`, `xlsx`, `pptx`, `docx` to accepted types
- [ ] `api.js` — admin SCORM URLs include `?token=`
- [ ] `api.js` — user `getScormPlayerUrl` uses `/user/lessons/` (NOT admin URL)
- [ ] `api.js` — user `getHlsManifestUrl` added
- [ ] `Preview/index.jsx` — HLS blob fetch, PDF blob fetch (remove `ct !== "pdf"` guard), `isHls` in both `setCurrentLesson` calls
- [ ] `ContentViewer.jsx` — `isHlsUrl` checks `lesson?.isHls` in addition to URL
- [ ] `ContentPreview/index.jsx` (admin) — all content fetched via stream blob, not raw CloudFront URL
