# Skill Bridge — Feature Tasks Document

Features implemented in the XR Dashboard / Skill Bridge LMS, ordered by complexity (hardest at top).

---

## 1. HLS Video Encryption & Streaming

**Difficulty:** 🔴 Hardest

- **Encrypted HLS Playback** — `.m3u8` manifest proxied through backend with presigned segment URLs; JWT embedded in key URI for license/auth
- **Admin HLS Preview** — `GET /admin/contenthub/lessons/{id}/hls-manifest` returns manifest URL without enrollment check (`api.js:1108`)
- **User HLS Playback** — `GET /user/lessons/{id}/hls-manifest` with enrollment validation (`api.js:1327`)
- **Authenticated Blob Stream** — Native `<video>`/`<audio>`/`<img>` can't send `Authorization` headers; blob fetched via `fetch()` with Bearer token, converted to `URL.createObjectURL()` (`api.js:1334-1343`)
- **hls.js Integration** — hls.js library handles segment fetching, decryption, and rendering (`CourseLibrary/Preview/index.jsx`)
- **External URL Resolver** — Unified resolution of YouTube/Vimeo/HLS/MP4/iframe URLs into `{ url, embedUrl, urlType }` (`api.js:1103-1106`)

---

## 2. Role-Based Access Control (RBAC)

**Difficulty:** 🔴 Very High

- **Permission Catalog** — Single source of truth defining all permissions, standard roles (admin/trainer/user), role metadata, icons, and authority flags (`access-control/catalog.js` — 373 lines)
- **3-Layer Permission Resolver** — Combines: (1) standard role permissions, (2) custom role overrides from backend, (3) per-user permission overrides (`access-control/resolver.js`)
- **`usePermission` Hook** — Client-side hook for `can('permission.key')` checks in components (`access-control/usePermission.js`)
- **Permission Context** — React Context storing custom roles and user overrides from localStorage (`context/PermissionContext.jsx`)
- **RBAC Admin UI** — `RolesTab` (list/edit templates), `PeopleTab` (assign roles to users), `PermissionMatrix` (grid of all permissions × roles), `NewRoleModal` (create custom roles) (`pages/Admin/AccessControl/`)
- **Route Guards** — `ProtectedRoute.jsx` with `AdminRoute`, `TrainerRoute`, `UserRoute`, `GuestRoute`, `AdminOrTrainerRoute` — lazy-loaded role-based route gating
- **Role Switcher** — Sidebar accordion for switching between assigned roles (`components/Shared/roleSwitcherAccordion.jsx`)
- **Backend-Driven RBAC Migration Plan** — Complete migration guide from client-side to server-side permissions with new API service patterns (`RBAC-IMPLEMENTATION-GUIDE.md`)
- **Role & UserRoleMapping CRUD APIs** — Full set of endpoints for roles and user-role assignments (`api.js:640-765`)

---

## 3. SCORM Runtime Bridge (1.2 + 2004)

**Difficulty:** 🔴 Very High

- **SCORM 1.2 API** — `window.API` exposing `LMSInitialize`, `LMSSetValue`, `LMSGetValue`, `LMSCommit`, `LMSFinish`, `LMSGetLastError`, `LMSGetErrorString`, `LMSGetDiagnostic` (`scormBridge.js`)
- **SCORM 2004 API** — `window.API_1484_11` with equivalent 2004 methods
- **Progress Tracking** — Derives completion status, score, scaled score from CMl data models; auto-syncs to backend via `apiClient.post("/scorm/progress", ...)`
- **Error Handling** — Standard SCORM error codes (0, 101, 201, 301, 401) with proper error string mapping
- **Iframe Wiring** — Bridge mounts on parent window BEFORE iframe src is set; SCO finds `window.parent.API` on boot
- **SCORM Player** — Dedicated player route with iframe hosting and API wiring (`api.js:1089-1095`)
- **SCORM/WebGL Upload** — Upload SCORM packages and WebGL content via admin endpoints (`api.js:782-798`)
- **LMS Package Reverse Proxy** — Wildcard reverse-proxy routing for multi-file SCORM/WebGL packages (`api.js:1362-1366`)
- **SCORM Progress API** — `POST /scorm/progress` saves SCO state persisted by lesson and user (`api.js:1097-1098`)

---

## 4. Real-Time Notifications via SignalR

**Difficulty:** 🔴 High

- **SignalR Hub Connection** — WebSocket connection to `/hubs/notifications` with `@microsoft/signalr`; automatic reconnect strategy `[0, 2s, 5s, 10s, 30s]` (`NotificationsHub.jsx`)
- **JWT Authentication** — `accessTokenFactory` reads token from localStorage for hub auth (`NotificationsHub.jsx:25-28`)
- **CustomEvent Bridge** — Incoming `"notify"` events re-broadcast as `window` CustomEvent `"app:notify"`; reconnect triggers `"app:notify-refetch"` (`NotificationsHub.jsx:34-44`)
- **Notifications Bell** — Dropdown bell icon showing unread count badge; plays `new-checkin.mp3` sound; auto-refreshes on new notification event (`NotificationsBell.jsx`)
- **Notifications List** — Full notification history with pagination, mark-as-read (single/bulk), delete (single/bulk), clear-all (`NotificationsList.jsx`)
- **Role-Scoped APIs** — Separate notification endpoints per role — `userNotificationAPI`, `trainerNotificationAPI`, `adminNotificationAPI` (`api.js:1411-1431`)
- **Role-Specific Pages** — Dedicated notification pages for User, Trainer, and Admin roles

---

## 5. Microsoft Teams Integration

**Difficulty:** 🔴 High

- **Teams Status Check** — `GET /integrations/teams/status/{orgId}` checks if org has Teams connected (`api.js:1450`)
- **Create Meeting** — `POST /integrations/teams/meetings` — create a Teams meeting linked to an event/course (`api.js:1451`)
- **Get Meetings for Event** — `GET /integrations/teams/meetings/{orgId}/{type}/{refId}` — retrieve meetings associated with a specific event/entity (`api.js:1452-1453`)
- **Cancel Meeting** — `DELETE /integrations/teams/meetings/{orgId}/{recordId}` — cancel/disconnect a Teams meeting (`api.js:1454-1455`)

---

## 6. Assessment Engine (Question Bank, Strategies, Bulk Upload)

**Difficulty:** 🔴 High

- **Assessment Hub (Admin)** — Full lifecycle: create/edit/view assessments, question bank management, reports (`pages/Admin/AssessmentsHub/` — 15 subdirectories)
- **Question Bank** — CRUD for questions with multiple types (MCQ, subjective, etc.) (`pages/Admin/AssessmentsHub/QuestionBank/`)
- **Question Strategies** — Per-subcategory strategy configuration for question selection algorithm (`pages/Admin/Preferences/QuestionStrategies/`)
- **Bulk Upload via Excel** — Upload questions/assessments using structured Excel template (`utils/bulkUpload/` — mapper, parser, template, validator; `public/templates/question-upload-template.xlsx`)
- **External Assessments** — Token-based assessment links for non-logged-in users (`pages/Admin/AssessmentsHub/ExternalUsers/`, `pages/ExternalUser/`, `api.js:1481-1507`)
- **User Assessments** — Take assessments, view attempt history, see results (`pages/User/Assessments/`, `api.js:1386-1394`)
- **Trainer Assessment Reports** — Overview and reporting on learner assessments (`pages/Trainer/Assessments/`, `api.js:1257-1266`)
- **Assessment Preview** — Preview assessment before publishing (`ContentPreview/AssessmentPreview.jsx`)

---

## 7. Certificate Management (Templates, Issuance, QR Verification)

**Difficulty:** 🟡 High-Medium

- **Certificate Templates** — Admin create/edit certificate templates with custom layout (`pages/Admin/Event&Certification/Certificats/`)
- **Issue Certificates** — Admin issue certificates to selected users (`pages/Admin/Event&Certification/Certificats/IssueCertificates/`)
- **HTML Certificate Generation** — Server-side HTML generation with `generateCertificateHTML.js`
- **Certificate Layout Component** — Reusable admin `CertificateLayout.jsx` for template design
- **QR Code Verification** — QR-based certificate preview and verification (`pages/User/Certificates/QRCertificatePreview.jsx`)
- **User Certificates** — View earned certificates with download (`pages/User/Certificates/`)
- **Trainer Certificates** — Trainer view of certificates for their assigned users (`pages/Trainer/Feedback&Certificates/Certificates/`)

---

## 8. Content Hub (CRUD, Version Control, Drag-and-Drop, Image Crop)

**Difficulty:** 🟡 High-Medium

- **Content Hub (Admin)** — Full CRUD for categories, sub-categories, curriculum/modules, lessons (`pages/Admin/ContentHub/`, `pages/Admin/ContentHub-new/`)
- **Content Types** — Video, PDF, PPT, SCORM, WebGL, Excel, HTML, EXE, APK (`data/mock-content-hub.js`)
- **Drag-and-Drop Reordering** — Lesson/module reordering via `@dnd-kit` (`content-hub-new/ContentDataTable.jsx`)
- **Version Control** — Version management modal for uploaded content (`content-hub-new/VersionControlModal.jsx`)
- **Image Cropping** — Crop images during upload using `react-easy-crop` (`content-hub-new/ImageCropModal.jsx`)
- **Content Preview** — Preview lessons in admin panel (`ContentPreview/`, `ContentPreviewModal.jsx`)
- **Upload Progress** — Upload with real-time progress tracking (`content-hub-new/UploadProgressCard.jsx`)
- **Admin HLS Preview** — `GET /admin/contenthub/lessons/{id}/hls-manifest` (no enrollment check)

---

## 9. Analytics Dashboards (Admin, Trainer, User)

**Difficulty:** 🟡 Medium

- **Admin Dashboard** — Quick stats, training hours trend, zone/cluster analytics, completion rates, target vs achievement, recent activity (`pages/Admin/Dashboard/`, `useAdminAnalytics.js`)
- **Trainer Dashboard** — Quick stats, completion trends, pass/fail rates, attendance vs enrollment, learner progress (`pages/Trainer/Dashboard/`, `useTrainerAnalytics.js`)
- **User Dashboard** — Skills overview, course progress tracking, performance analytics (`pages/User/Dashboard/`, `useUserAnalytics.js`)
- **Recharts Integration** — Interactive charts (bar, line, pie, radial) across all dashboards
- **KPI Cards** — Reusable `KpiCard` component with trend indicators and comparison periods

---

## 10. Authentication & Session Management

**Difficulty:** 🟡 Medium

- **Login (Password + OTP)** — Email/password login with org-code routing; OTP-based login flow (`LoginForm.jsx`, `LoginWithOtp.jsx`)
- **Signup** — User registration with validation (`SignupForm.jsx`)
- **Forgot/Reset Password** — Password recovery with email link (`ForgotPassword.jsx`, `ResetPassword.jsx`)
- **MFA Settings** — Multi-factor authentication configuration (`pages/User/MFASettings.jsx`)
- **Token Refresh** — Axios interceptor that auto-refreshes expired tokens (`api.js:1-115`)
- **Session Expiry Modal** — Modal warning when session is about to expire, with auto-logout
- **Org-Based Routing** — Routes prefixed by org code (e.g., `/login/schneider`) (`routes/index.jsx`)

---

## 11. Event & Calendar Management

**Difficulty:** 🟡 Medium

- **Event CRUD (Admin)** — Create, edit, delete events with scheduling (`pages/Admin/Event&Certification/Events/`)
- **Trainer Events** — Event calendar with attendance tracking (`pages/Trainer/Events&Attendance/`)
- **User Events** — View event calendar and upcoming events (`pages/User/Events/`)
- **Calendar Widget** — `react-day-picker` based calendar (`components/ui/CalenderWidget.jsx`)
- **Event Notifications** — Send email notifications for events (`api.js:1249-1250`)
- **Teams Meeting Linking** — Link Teams meetings to events for online sessions

---

## 12. User & Employee Management

**Difficulty:** 🟡 Medium

- **Employee Management** — CRUD for employees with departments, designations (`EmployeeList.jsx`, `EmployeeForm.jsx`, `EmployeeView.jsx`)
- **User Management** — CRUD for platform users (`pages/Admin/UserManagement/Users/`)
- **Trainer Management** — CRUD for trainers with assigned users (`pages/Admin/UserManagement/Trainers/`)
- **Assigned Users** — View users assigned to specific trainers (`pages/Admin/UserManagement/AssignedUsers/`)
- **Bulk Delete** — Soft and permanent bulk delete with confirmation dialog (`api.js:630-637`)
- **Export** — Export user data to Excel

---

## 13. Master Data Management

**Difficulty:** 🟢 Medium-Low

- **Offer Type** — CRUD for training/offer types (`pages/Admin/Master/OffereType/`)
- **Category / SubCategory** — Hierarchical category management (`pages/Admin/Master/Category/`, `SubCategory/`)
- **Partner** — Partner organization management (`pages/Admin/Master/Partner/`)
- **Department** — Department CRUD (`pages/Admin/Master/Department/`)
- **Zone / Cluster / Country** — Geographic hierarchy management (`pages/Admin/Master/Zone/`, `Cluster/`, `Country/`)
- **ContentMain Master** — Content categorization master (`pages/Admin/Master/ContentMaster/`)
- **Group Master** — Group management with user-course assignment (`pages/Admin/Master/GroupMaster/`)

---

## 14. Feedback System

**Difficulty:** 🟢 Medium-Low

- **User Feedback** — Submit feedback with optional file attachments (`pages/User/Feedback/`, `api.js:1378-1384`)
- **Admin Feedback Management** — View/search/export all submitted feedback (`pages/Admin/FeedbackManagement/`, `api.js:1126-1133`)
- **Trainer Feedback** — View feedback for trainer's assigned users (`pages/Trainer/Feedback&Certificates/Feedback/`, `api.js:1278-1286`)
- **Partner Feedback** — Partner-specific feedback collection (`pages/Admin/FeedbackManagement/PartnerFeedback.jsx`)

---

## 15. Groups & Course Assignment

**Difficulty:** 🟢 Medium-Low

- **Group Master** — Group CRUD with user assignment and course linking (`api.js:703-720`)
- **Trainer Groups** — Trainer-specific group management with course content assignment (`components/Trainer/groups/`, `api.js:1198-1217`)
- **User Group Assignment** — Assign users and groups to courses (`api.js:1185-1196`)

---

## 16. Settings & Preferences

**Difficulty:** 🟢 Low

- **Admin Settings** — Organization auth settings, security config (`pages/Admin/Preferences/Settings/`)
- **Trainer Settings** — Profile, password, notification prefs, language/timezone (`pages/Trainer/Settings/`)
- **User Settings** — Profile, password, notification preferences (`pages/User/Settings.jsx`)
- **Dashboard Card Preferences** — Per-role configurable dashboard cards saved to server (`CardPreferenceContext.jsx`, `api.js:1518-1525`)
- **Subscriber Management** — API subscribers with secret keys, access rules, audit logs (`pages/Admin/Preferences/Subscribers/`, `api.js:1527-1539`)

---

## 17. Theme System & Translation (i18n)

**Difficulty:** 🟢 Low

- **Theme Switching** — Dark/light theme via Tailwind CSS with `ThemeContext.jsx`; persistent user preference
- **Translation / i18n** — Multi-language support framework via `TranslationContext.jsx`; translatable UI labels

---

## 18. File Handling (Excel, PDF, S3 Upload)

**Difficulty:** 🟢 Low

- **Excel Import/Export** — Read/write `.xlsx` using `xlsx` library; bulk upload questions, export user data
- **PDF Generation** — Generate PDF reports using `jspdf` + `jspdf-autotable`; certificate PDFs
- **S3 Upload** — Presigned URL-based uploads and deletes (`api.js:1509-1512`)

---

## 19. UI Component Library (Shadcn-style Radix)

**Difficulty:** 🟢 Low

- **31 Radix UI Components** — accordion, avatar, badge, button, calendar, card, checkbox, collapsible, command (Cmd+K palette), dialog, drawer (Vaul), dropdown-menu, input-otp, input, label, multiSelect, popover, progress, radio-group, scroll-area, select, separator, sonner (toasts), switch, table, tabs, textarea, tooltip (`components/ui/`)
- **Command Palette** — Cmd+K global search with `cmdk` (`components/ui/command.jsx`)
- **Toast System** — Sonner toast notifications for success/error/info
- **Drag-and-Drop** — `@dnd-kit` for lesson/module reordering

---

## 20. Error Handling & Network Status

**Difficulty:** 🟢 Low

- **Error Boundaries** — Global `GlobalErrorBoundary` and route-level `ErrorBoundary` with fallback UI (`components/error-boundary/`)
- **Network Status** — Online/offline detection, server error card, timeout card, retry logic (`components/networkError/`)
- **Session Expired Modal** — Auto-detect 401 responses and show session expiry modal
- **Catch Error Formatter** — Unified error message formatting utility (`utils/catchErrorFormatter.js`)

---

## 21. Chat Bot (using LLM/Data)

**Difficulty:** 🟢 Not yet implemented — planned/envisioned

- AI-powered chatbot for learner assistance, course recommendations, FAQ answering
- Integration with course content data, assessment results, and user progress
- Can be built on top of the existing API layer and data models
