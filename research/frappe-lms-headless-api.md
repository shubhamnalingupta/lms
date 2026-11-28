# Can Frappe LMS's API run a headless mobile/web app?

Research for issue #3. Question: can one Expo / React Native app (Android, iOS, web) use **upstream Frappe LMS, unmodified** as its backend?

## Verdict

**Yes, with one small custom Frappe app for the gaps. No fork is needed.**

- LMS's own Vue frontend is already a headless client. It calls whitelisted Python methods (`/api/method/lms.…`) and Frappe's generic `frappe.client.*` / `/api/resource` API, and nothing else. An Expo app can call the same endpoints.
- Mobile auth works today with no code: **API key/secret tokens** or **OAuth 2 Authorization Code + PKCE**, both built into Frappe.
- The real gaps are **mobile push notifications**, a **native payment return flow**, and a **stable, documented API contract**. All three can go in a separate custom Frappe app, using hooks that merge across apps. None needs a core change.
- The main risk is not capability. The LMS API is **internal and undocumented**: method names and signatures change between releases (`main` refactors often). Pin an LMS version and wrap what you use behind a facade in your custom app.

## Sources and version

- Code read: `frappe/lms` `main` at commit `adb19c73a3d884b9dc84794475ece8330275b329` (2026-11-26). File links below are relative to that tree.
- Framework code read: `frappe/frappe` branch `version-16` (`frappe/auth.py`, `frappe/oauth.py`, `frappe/integrations/oauth2.py`, `frappe/core/doctype/user/user.py`, `frappe/www/login.py`, `frappe/push_notification.py`).
- **Latest release:** tag **`v16.4.0`**, commit "chore(release): Bumped to Version 16.4.0", dated 2026-11-20 (`git ls-remote --tags`; `lms/__init__.py` at that tag says `16.4.0`). `main`'s `lms/__init__.py` says `2.45.2`, so `main` is a dev line. Pin to a release tag, not `main`.
- **Licence: AGPL-3.0.** `license.txt` is the GNU AGPL v3. `package.json` says `"license": "AGPL-3.0-or-later"`, and `hooks.py` says `app_license = "AGPL"`. An Expo app that only talks to LMS over HTTP is a separate work. But if you change LMS itself, or ship your custom Frappe app into the same server process, the AGPL network-use clause (§13) applies to what you change on the server. Get a legal read before you plan to keep server code closed.
- **Not checked:** docs.frappe.io was unreachable from this session (proxy-blocked). Everything here comes from source code. The LMS repo has no published REST/API reference of its own.

## How the API surface works

| Layer | What it is | Notes |
|---|---|---|
| Whitelisted LMS methods | `POST/GET /api/method/lms.lms.api.<fn>`, `lms.lms.utils.<fn>`, `lms.lms.doctype.<dt>.<dt>.<fn>` | About 189 `@frappe.whitelist` functions, 28 of them `allow_guest=True`. Most are in `lms/lms/api.py` (4.7k lines) and `lms/lms/utils.py`. |
| Frappe generic API | `/api/method/frappe.client.{get,get_list,insert,set_value,delete}`, `/api/resource/<DocType>`, `/api/v2/...` | Gated by DocType permissions and LMS's `has_permission` / `permission_query_conditions` hooks (`hooks.py`), so student-scoped reads are enforced on the server. |
| Realtime | Socket.IO (`frappe.publish_realtime`) | LMS emits `publish_lms_notifications` to the user when a Notification Log changes (`utils.publish_notifications`). |

**Optional lockdown, `block_endpoints`** (`lms/auth.py`, registered as `auth_hooks`). If a site sets `block_endpoints` in site config, non-System users can only reach `lms.*` methods, a fixed allow-list (login, OAuth endpoints, `frappe.client.*`, `upload_file`, `download_pdf`, notification mark-read, ...), API Server Scripts, and paths listed in the `allowed_custom_endpoints` site config. `/api/resource` is then refused for students, and v2 document routes are treated the same way. **Consequence:** call `frappe.client.*` rather than `/api/resource`, and list your custom app's endpoints in `allowed_custom_endpoints`. Both are config, not code.

**Which DocTypes students can reach directly.** `LMS Student` has no DocPerm on `LMS Course` or `Course Lesson`. The catalogue and lesson content are served only through guest-allowed whitelisted methods (`get_courses`, `get_course_details`, `get_course_outline`, `get_lesson`). Students have read+write+create on `LMS Enrollment` and `LMS Assignment Submission`, and read on `LMS Course Progress`, `LMS Quiz Submission`, `LMS Certificate`, `LMS Batch` and `LMS Live Class` (from each DocType's JSON). New users get `LMS Student` automatically (`hooks.py` `User.before_insert` → `lms.lms.user.add_lms_student_role`).

## Feature by feature

Legend: **Ready** = works now with no code. **Custom app** = gap fillable in a separate Frappe app. **Core** = needs an LMS/Frappe change.

### 1. Authentication — Ready (with custom app for nicer UX)

| Method | Status | Evidence |
|---|---|---|
| Username/password → session cookie | Ready | `POST /api/method/login` (on the `block_endpoints` allow-list). Cookie `sid`. Works on native, where you manage the cookie yourself. On Expo web cross-origin it needs Frappe's `allow_cors` site config plus credentialed requests. |
| API key + secret (`Authorization: token key:secret`) | Ready, but key issuance is admin-only | `frappe/auth.py` `validate_auth_via_api_keys`. `generate_keys` calls `frappe.only_for("System Manager")` (`user.py`), so a student cannot mint their own. **Custom app:** a whitelisted "login → return per-user key/secret" endpoint is a common pattern. It is less safe than OAuth (no expiry, no scopes). |
| **OAuth 2 Authorization Code + PKCE** (Frappe as OAuth provider) | **Ready. Recommended.** | `frappe/integrations/oauth2.py` advertises `code_challenge_methods_supported=["S256"]`; `frappe/oauth.py` stores and checks `code_challenge`. `authorize`, `get_token`, `revoke_token`, `openid_profile` and `register_client` are all on LMS's allow-list in `lms/auth.py`. Bearer tokens are accepted by `validate_oauth`. Register an `OAuth Client` with your app's redirect URI and use `expo-auth-session`. **Verify:** that `OAuth Client` accepts a custom-scheme or `exp://` redirect URI. |
| Social login (Google, GitHub, Facebook, Office 365, Keycloak, custom) | Ready via the OAuth flow above | `frappe.www.login.login_via_*` are redirect callbacks that set a browser session. They are not usable as token APIs from native code. Instead, the user picks "Sign in with Google" on Frappe's login page inside the OAuth PKCE browser sheet, and the app receives a Frappe bearer token. A native Google SDK token → Frappe exchange would need a **custom app** endpoint. |
| OTP | Partial | Frappe's OTP is **two-factor on top of password** (`frappe/twofactor.py`, `enable_two_factor_auth`). It works inside the OAuth browser sheet. "Login with email link" (`frappe.www.login.send_login_link`, System Settings `login_with_email_link`) is a magic link, not a code. **Passwordless SMS/email OTP as the primary login is missing → custom app** (whitelisted `send_otp` / `verify_otp` that creates a session or an OAuth token). |
| Sign-up | Ready | `lms.lms.user.sign_up(email, full_name, verify_terms, user_category)`, guest-allowed and rate-limited. The password-setup step is an emailed web link. |

### 2. Courses, chapters, lessons — Ready

- Catalogue: `lms.lms.utils.get_courses`, `get_course_count`, `get_course_categories`, `get_course_details`, `get_related_courses`, `get_reviews` (all guest-allowed).
- Structure: `lms.lms.utils.get_course_outline`.
- Lesson: `lms.lms.utils.get_lesson(course, chapter, lesson)`, addressed by chapter and lesson **index**. It applies `can_access_course`, lesson locking (`locked` / `redirect_to`), preview rules (`no_preview`), and hides `instructor_content` / `instructor_notes` from students (`utils.py`).
- Enrol in a free course: `frappe.client.insert({doctype: "LMS Enrollment", course, member})`, exactly as `frontend/src/components/CourseCardOverlay.vue` does. Paid courses are refused by server hooks, so they go through payments (section 8).
- My learning: `lms.lms.api.get_my_courses`, `get_course_to_continue`.
- **Rendering cost (the main client-side work):** lesson `content` is **Editor.js JSON** (`frontend/package.json` `@editorjs/*`), with custom block types: `upload` (video/audio/PDF/file), `embed` (YouTube, Vimeo, Cloudflare Stream, Bunny Stream, Aparat), `quiz`, `assignment`, `program`, `markdown`, `codeBox`, `table`, `list`, `image` (`frontend/src/utils/index.js`). Older lessons use `body` (Markdown with macros such as `{{ YouTubeVideo(...) }}`, `hooks.py` `lms_markdown_macro_renderers`). You need a React Native renderer for these blocks. That is client work only.
- SCORM lessons return `is_scorm_package` and are played through `course_chapter.get_scorm_playback`. Use a WebView.

### 3. Video lessons — Ready

- Private uploaded media is served through `lms.lms.doctype.course_lesson.course_lesson.serve_resource(file_url)`. `get_lesson` rewrites embedded URLs to point at it. It is guest-allowed but access-gated by `can_access_lesson`, rate-limited to 20,000/hour, and delegates to Frappe's `send_private_file` (nginx `X-Accel-Redirect` in production, which handles byte ranges for iOS AVPlayer). The code comments explicitly expect one request per video byte-range.
- The player must send auth on media requests: `expo-video` source `headers` with the bearer token, or a cookie.
- Third-party embeds (YouTube, Vimeo, Bunny, Cloudflare) are just URLs, so play them with a native player or a WebView.
- Watch tracking: `lms.lms.api.track_video_watch_duration(lesson, videos)`.
- **No DRM, signed-URL, or HLS transcoding** is in LMS. If you need that, use Bunny or Cloudflare Stream embeds, which are already supported.

### 4. Quizzes and assignments — Ready

- Quiz: `lms.lms.utils.get_quiz_with_questions(quiz)` (question content is withheld outside the schedule window), `lms.lms.doctype.lms_quiz.lms_quiz.check_answer`, `submit_quiz(quiz, results, violation_count, submission_reason, violation_events)` (POST).
- Assignment: `lms.lms.utils.get_assignment`. Submit with `frappe.client.insert` / `set_value` on `LMS Assignment Submission` (`answer`, `assignment_attachment`), as `frontend/src/components/Assignment.vue` does. Files go to `/api/method/upload_file` first. Own submission: `lms.lms.api.get_own_assignment_submission`.
- Programming exercises: `get_programming_exercise`, `create_programming_exercise_submission`, `evaluate_programming_exercise`.

### 5. Progress tracking — Ready

- `lms.lms.doctype.course_lesson.course_lesson.save_progress(lesson, course)` (POST), or `lms.lms.api.mark_lesson_progress(course, chapter_number, lesson_number)`, which returns whether the lesson is now complete. Both are no-ops when the user is not enrolled or lesson requirements are unmet.
- Reads: `LMS Enrollment.progress` and `current_lesson`, `get_course_assessment_progress`, `get_streak_info`, `get_heatmap_data`.
- **Offline:** no offline-sync or batch endpoint exists. Queue calls on the device and replay them. A batch endpoint could go in a **custom app** if needed.

### 6. Certificates — Ready

- List: `lms.lms.api.get_user_certificates(username)`, `get_certification_details`. Students can read `LMS Certificate`.
- PDF: `GET /api/method/frappe.utils.print_format.download_pdf?doctype=LMS+Certificate&name=<id>&format=<print format>`, as the frontend does (`CertificationLinks.vue`). It is on the allow-list. Download with the auth header and open with `expo-sharing` or a PDF viewer.
- Public verify page: `/courses/<course>/<certificate_id>` (`website_route_rules`).
- Evaluation-based certificates: `LMS Certificate Request` / `course_evaluator.get_schedule`.

### 7. Batches and live classes — Ready

- Batches: `lms.lms.utils.get_batches`, `get_batch_details`, `get_batch_courses`, `enroll_in_batch`. `lms.lms.api.get_my_batches`. Timetable: `lms.lms.doctype.lms_batch.lms_batch.get_batch_timetable`.
- Live classes: `lms.lms.api.get_my_live_classes` returns `join_url`. Providers are **Zoom and Google Meet** (`LMS Live Class.conferencing_provider`). The app opens the `join_url` in the Zoom or Meet app. No in-app video SDK is involved.
- Attendance is synced hourly on the server (`lms_live_class.update_attendance`).
- Announcements: `get_announcements`.

### 8. Payments — Works, but web-redirect only (custom app for a native flow)

- `lms.lms.payments.get_payment_link(doctype, docname, address, payment_for_certificate, coupon_code)` returns either a **hosted checkout URL** from the `frappe/payments` app (a required app, per `hooks.py` `required_apps`), or the LMS web route straight away if the order is free or already paid.
- The gateway calls back `on_payment_authorized` on `LMS Course` / `LMS Batch`, which records payment and enrols the user.
- Order summary and coupons: `lms.lms.utils.get_order_summary`.
- **Gaps:**
  - `redirect_to` is always an LMS **web** URL (`payments.get_redirect_url`), so an in-app browser will not auto-close back into the app. Workaround with no code: open the URL in `WebBrowser`, then poll `LMS Enrollment` when it closes. Cleaner: a **custom app** endpoint that creates the payment with an app deep link or universal link as `redirect_to`, or native Razorpay/Stripe SDK order creation plus webhook verification that calls the same enrolment.
  - **Store rules (not verified here, check them):** Apple and Google generally require in-app purchase for digital content consumed in the app. Selling courses inside the iOS/Android app through Razorpay or Stripe may be rejected. That would need IAP plus server-side receipt validation in a **custom app**. Expo web is unaffected.

### 9. Notifications — In-app Ready; push = custom app

- In-app: `lms.lms.api.get_notifications(filters)` (always scoped to the session user). Mark read with `frappe.desk.doctype.notification_log.notification_log.mark_as_read` / `mark_all_as_read` (allow-listed). Live updates come from the Socket.IO event `publish_lms_notifications`.
- Email: reminders are already scheduled on the server (batch start, live class, payment reminder, day-1 email, in `hooks.py` `scheduler_events`).
- **Mobile push: missing.** Frappe's `PushNotification` (`frappe/push_notification.py`) is a Frappe-Cloud relay keyed by **FCM** tokens and aimed at Frappe's own PWAs. It is not Expo/APNs. **Custom app:** a `Device Token` DocType, a whitelisted `register_device(expo_push_token)`, and `doc_events = {"Notification Log": {"after_insert": "myapp.push.send"}}` that calls the Expo Push API. Frappe merges `doc_events` across installed apps, so LMS's own handler keeps working.

## What the custom Frappe app would hold (no fork)

1. **Push**: device-token DocType, register/unregister endpoints, and a `Notification Log` `after_insert` hook that sends through Expo Push.
2. **Auth extras (optional)**: passwordless OTP login and/or native Google ID-token exchange that returns an OAuth bearer token.
3. **Payments for the app**: create-order endpoint with an app deep link `redirect_to` or a native SDK flow, plus IAP receipt validation if the stores require it.
4. **API facade (recommended)**: thin `myapp.api.v1.*` wrappers over the LMS methods the app uses, so LMS refactors break one server file instead of shipped mobile binaries.
5. Site config, not code: `allow_cors` for Expo web, `allowed_custom_endpoints` if `block_endpoints` is on, and an `OAuth Client` record.

Frappe supports all of this through standard extension points: `@frappe.whitelist`, `doc_events`, `override_whitelisted_methods`, and `has_permission` hooks. LMS itself shows the pattern (`hooks.py`). No change to `frappe/lms` or `frappe/frappe` code was found to be necessary.

## Risks

- **Unstable API contract.** The LMS endpoints exist for the Vue SPA. They are undocumented and unversioned, and `main` is heavily refactored. Pin a release tag (currently `v16.4.0`), keep a facade, and add contract tests.
- **Editor.js rendering** in React Native is the largest client-side effort.
- **AGPL** obligations for any server-side modifications.
- **Store payment rules** for selling digital courses in native apps.
