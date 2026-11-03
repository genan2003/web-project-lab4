# Attendance app review: tickets

Problems found in `teamwork-attendance` at `gen` (23cb667) on 2026-11-02, one section per HelpDesk ticket. File paths are inside `teamwork-attendance/`.
25 tickets: 3 high, 10 normal, 12 low. 7 are fixed in this pull request, each on its own branch merged into `fixes/all`; the rest are open and need a decision from the team.

The backend was read in full and the test suite run; the frontend (`app/static`) was read and each finding traced through the code.
Frontend findings were not reproduced in a browser.

| # | Ticket | Priority | Status |
| --- | --- | --- | --- |
| 1 | [QR code on the board works for only one student per refresh](#1-qr-code-on-the-board-works-for-only-one-student-per-refresh) | High | Fixed, `fix/qr-whole-class` |
| 2 | [Attendance board freezes silently when the 30-minute sign-in expires](#2-attendance-board-freezes-silently-when-the-30-minute-sign-in-expires) | High | Fixed, `fix/board-signin-expired` |
| 3 | [Scan sheet: correcting a dropdown leaves the wrong student ticked and saves them as present](#3-scan-sheet-correcting-a-dropdown-leaves-the-wrong-student-ticked-and-saves-them-as-present) | High | Fixed, `fix/scan-dropdown-unticks` |
| 4 | [Scan: names like Marin, Iulian or Decebal are read as the sheet date](#4-scan-names-like-marin-iulian-or-decebal-are-read-as-the-sheet-date) | Normal | Fixed, `fix/scan-date-from-names` |
| 5 | [Scan: a damaged or huge photo returns 500 and can use hundreds of MB](#5-scan-a-damaged-or-huge-photo-returns-500-and-can-use-hundreds-of-mb) | Normal | Fixed, `fix/scan-unreadable-image` |
| 6 | [uv.lock is out of date: uv sync --frozen installs an environment without Pillow](#6-uvlock-is-out-of-date-uv-sync---frozen-installs-an-environment-without-pillow) | Normal | Fixed, `chore/update-uv-lock` |
| 7 | [Back-dated scanned sessions never count anyone as absent](#7-back-dated-scanned-sessions-never-count-anyone-as-absent) | Normal | Open |
| 8 | [Re-adding a removed student counts the classes in between as absences](#8-re-adding-a-removed-student-counts-the-classes-in-between-as-absences) | Normal | Open |
| 9 | [Ten wrong passwords lock any account for 15 minutes, even with the right password](#9-ten-wrong-passwords-lock-any-account-for-15-minutes-even-with-the-right-password) | Normal | Open |
| 10 | [Per-IP attempt limit is shared by the whole classroom behind one NAT or proxy](#10-per-ip-attempt-limit-is-shared-by-the-whole-classroom-behind-one-nat-or-proxy) | Normal | Open |
| 11 | [Opening a QR link while signed out fails with an expired QR](#11-opening-a-qr-link-while-signed-out-fails-with-an-expired-qr) | Normal | Open |
| 12 | [Student page stops updating after one failed request](#12-student-page-stops-updating-after-one-failed-request) | Normal | Open |
| 13 | [Accessibility: the whole app is one aria-live region](#13-accessibility-the-whole-app-is-one-aria-live-region) | Normal | Open |
| 14 | [Compiled Python files are tracked in git](#14-compiled-python-files-are-tracked-in-git) | Low | Fixed, `chore/untrack-pycache` |
| 15 | [Creating a class from All students stays on All students; double click makes two classes](#15-creating-a-class-from-all-students-stays-on-all-students-double-click-makes-two-classes) | Low | Open |
| 16 | [Roster status dropdown shows the new value when saving failed](#16-roster-status-dropdown-shows-the-new-value-when-saving-failed) | Low | Open |
| 17 | [Notifications popover can open twice and one copy cannot be closed](#17-notifications-popover-can-open-twice-and-one-copy-cannot-be-closed) | Low | Open |
| 18 | [QR link edge cases: malformed link gives a blank page, professor tab keeps a stale QR](#18-qr-link-edge-cases-malformed-link-gives-a-blank-page-professor-tab-keeps-a-stale-qr) | Low | Open |
| 19 | [Scanned sessions show a made-up 02:00 time and the raw method name "scan"](#19-scanned-sessions-show-a-made-up-0200-time-and-the-raw-method-name-scan) | Low | Open |
| 20 | [Per-class attendance line counts excused classes, the overall figure does not](#20-per-class-attendance-line-counts-excused-classes-the-overall-figure-does-not) | Low | Open |
| 21 | [Attendance shows 0% when nothing counts yet or every class was excused](#21-attendance-shows-0-when-nothing-counts-yet-or-every-class-was-excused) | Low | Open |
| 22 | [Roster import merges two students with the same name into one account](#22-roster-import-merges-two-students-with-the-same-name-into-one-account) | Low | Open |
| 23 | [Scanning the same sheet twice creates duplicate sessions](#23-scanning-the-same-sheet-twice-creates-duplicate-sessions) | Low | Open |
| 24 | [Registering the same email twice at the same moment returns 500](#24-registering-the-same-email-twice-at-the-same-moment-returns-500) | Low | Open |
| 25 | [PROBLEMS.md describes a different version of the project](#25-problemsmd-describes-a-different-version-of-the-project) | Low | Open |

## 1. QR code on the board works for only one student per refresh

Priority: high. Status: fixed in this pull request (`fix/qr-whole-class`). Tags: backend, qr, fix-branch.

**Where:** `app/main.py` `confirm_attendance()`, unique index ux_attendance_records_qr_nonce (`app/models.py`, `app/db.py`)

**What happens:** the board shows one QR and refreshes it every 20 s, but each QR token is single-use. The first student who scans it is marked present. Everyone else who scans the same QR gets 409 "QR code has already been used", and that is logged as a failed "invalid" attempt. After 5 failures a student is locked out of the session for 10 minutes, and after 30 failures from one IP (campus Wi-Fi NAT) the whole room is locked out, including students typing the correct code.

**Steps:** open a session, let two enrolled students scan the same QR within 20 s. The second one gets 409. The UI even tells them to wait for the next QR.

**Fix:** in this pull request, branch `fix/qr-whole-class`. A token works for every enrolled student until it expires (30 s); each student still confirms once per session. The nonce stays on the record, the unique index is dropped (also on existing databases). Tests and docs updated.

## 2. Attendance board freezes silently when the 30-minute sign-in expires

Priority: high. Status: fixed in this pull request (`fix/board-signin-expired`). Tags: frontend, board, fix-branch.

**Where:** `app/static/app.js` `api()` 401 handler and `openBoard()`

**What happens:** the sign-in cookie lasts 30 minutes (ACCESS_TOKEN_MINUTES) and a session can run for up to 120. The board is a full-screen overlay outside #app. On the first 401 the app draws the sign-in form under the board and stops all timers. The countdown freezes, the QR expires 30 s later so every scan fails, new names stop appearing, and New code / End attendance throw unhandled 401s. No message is shown.

**Steps:** sign in as professor, wait about 20 minutes, start a 30-minute session. About 10 minutes later the board stops updating.

**Fix:** in this pull request, branch `fix/board-signin-expired`. The board registers as an overlay; the 401 handler and Sign out take overlays (and the notifications popover) down, so the professor sees "Your sign-in expired", signs in and reopens the board with Show attendance board. Still open: the token cannot be renewed, so a long session always needs a second sign-in. A refresh endpoint, or a token lifetime longer than the longest session, would remove that.

## 3. Scan sheet: correcting a dropdown leaves the wrong student ticked and saves them as present

Priority: high. Status: fixed in this pull request (`fix/scan-dropdown-unticks`). Tags: frontend, scan, fix-branch.

**Where:** `app/static/scan.js` `scanSession()`, change handler on the review dropdowns

**What happens:** rows the app is unsure about come with the best guess preselected, and that student is ticked. The change handler only ever ticks the new pick. Switching the dropdown to the right student, or to Nobody, leaves the first guess ticked, and both are sent to /attendance-scans/confirm.

**Steps:** a sheet line "Popescu Ana" lands in review with "Popescu Ioana" preselected. Switch to Ana, Approve and save. Both are recorded present.

**Fix:** in this pull request, branch `fix/scan-dropdown-unticks`. The previous pick is unticked unless the sheet read that student on another row or another dropdown still picks them.

## 4. Scan: names like Marin, Iulian or Decebal are read as the sheet date

Priority: normal. Status: fixed in this pull request (`fix/scan-date-from-names`). Tags: backend, scan, ocr, fix-branch.

**Where:** `app/recognition.py` `find_date()`

**What happens:** WORD_DATE matches a number followed by any word, then looks up the first three letters as a month. "3) Marin Andrei" becomes 3 March, "5) Iulian Pop" 5 July, "7) Decebal Ion" 7 December. On a sheet whose date is missing or unreadable, that date is offered for the scanned session. An impossible numeric date (31.02.2026) also stopped the search instead of trying the next line.

**Fix:** in this pull request, branch `fix/scan-date-from-names`. Month words must be a whole month name or abbreviation (Romanian or English); invalid numeric dates are skipped. Tests added.

## 5. Scan: a damaged or huge photo returns 500 and can use hundreds of MB

Priority: normal. Status: fixed in this pull request (`fix/scan-unreadable-image`). Tags: backend, scan, fix-branch.

**Where:** `app/attendance_scan.py` `preview_scan()`, `app/recognition.py` `_upscaled_png()`

**What happens:** the upload check only looks at the first bytes. A PNG/JPEG signature followed by a damaged body raises inside Pillow/OCR and returns 500. A small PNG can declare a canvas of 150 megapixels: Pillow only warns and decodes it (about 450 MB RGB), then the OCR decodes it again. Over about 179 megapixels Pillow raises DecompressionBombError, again a 500.

**Fix:** in this pull request, branch `fix/scan-unreadable-image`. The photo is opened and decoded before OCR: damaged files get 415, photos over 50 megapixels get 413. Tests now use a real PNG.

## 6. uv.lock is out of date: uv sync --frozen installs an environment without Pillow

Priority: normal. Status: fixed in this pull request (`chore/update-uv-lock`). Tags: repo, dependencies, fix-branch.

**Where:** `uv.lock`, `pyproject.toml`

**What happens:** rapidocr, onnxruntime, rapidfuzz and pillow were added to `pyproject.toml` without running uv lock. uv sync --frozen and uv run --frozen install from the old lock, so Pillow is missing and every test fails to import app.main (ModuleNotFoundError: No module named 'PIL'). uv lock --check fails. The Dockerfile installs from `pyproject.toml` with pip, so the image still builds, but with unpinned versions.

**Fix:** in this pull request, branch `chore/update-uv-lock`. The lock is regenerated and the 188 tests pass with uv run --frozen. Suggested follow-up: build the image from the lock (uv sync --frozen in the Dockerfile) and add uv lock --check to CI.

## 7. Back-dated scanned sessions never count anyone as absent

Priority: normal. Status: open. Tags: backend, scan, statistics.

**Where:** `app/models.py` Enrollment.enrolled_at (default now_utc), `app/attendance_scan.py` `confirm_scan()`, `enrolled_at_start()` in `app/main.py`

**What happens:** statistics count a student for a session only if enrolled_at is before the session start. enrolled_at is set to the moment the student was added in the app (UI enroll and app.roster import alike). Scanned sheets are usually from earlier days and get starts_at = midnight UTC of the sheet date, so every student added after that date is left out. Only the students ticked on the sheet are counted: a back-dated session always shows 100% and nobody gets an absence.

**Steps:** enroll two students today, scan a sheet dated last week with one name, save. Class statistics show 1 of 1 present; the other student's statistics do not include the session.

**Suggested fix:** decide what enrolled_at means. For the roster import it is the official list from the start of the semester, so enrolled_at = NULL ("from the start") fits. For scans, either let the professor confirm that everyone currently in the class was in it on that date, or store an enrollment start date the professor can set. Related: imported sections belong to an inactive placeholder professor, so no real professor can scan into them without editing the database.

## 8. Re-adding a removed student counts the classes in between as absences

Priority: normal. Status: open. Tags: backend, enrollment, statistics.

**Where:** `app/main.py` `enroll_student()`, branch for existing non-active enrollments

**What happens:** re-adding a removed student sets status=active and ended_at=NULL but keeps the original enrolled_at. `enrolled_at_start()` then treats the student as enrolled for every session held while they were out of the class. They could not confirm those (confirmation needs an active enrollment), so they all show as absent.

**Steps:** enroll a student, remove them, hold and close a session, add them back. Their statistics show one absence for the session they were not part of.

**Suggested fix:** one enrollment row has room for one period only. Either store enrollment periods (a separate table, or a new row per period without the (section_id, student_id) unique constraint), or on re-add move enrolled_at to now and accept that sessions before the removal stop counting.

## 9. Ten wrong passwords lock any account for 15 minutes, even with the right password

Priority: normal. Status: open. Tags: backend, auth, security.

**Where:** `app/main.py` `authenticate()`

**What happens:** failed sign-ins are counted per email only. Anyone who knows a professor's email can send 10 wrong passwords and keep that professor locked out by repeating it every 15 minutes, for example right before a class. The real user is refused even with the correct password. A successful sign-in does not reset the count.

**Suggested fix:** count failures per (email, client IP) and per IP, or add a growing delay instead of a hard lock, and keep the per-email limit much higher as a last resort. Unknown emails also skip the password hash, so the response time shows which emails have accounts; verify against a dummy hash in that case.

## 10. Per-IP attempt limit is shared by the whole classroom behind one NAT or proxy

Priority: normal. Status: open. Tags: backend, rate-limit.

**Where:** `app/main.py` `attempt_limit_reached()`, `client_ip()`; `app/config.py` attempt_limit_per_ip = 30

**What happens:** failed confirmations are limited to 30 per IP per session in 10 minutes. Students on campus Wi-Fi share one public IP, and behind a reverse proxy request.client.host is the proxy's address for everyone. 30 typos in a large class block every student on that network, including those with the right code, because the limit is checked before the code. The QR bug (separate ticket) made this much easier to hit.

**Suggested fix:** students are signed in, so the per-student limit already covers guessing. Raise the per-IP limit a lot (or drop it for authenticated students), and when deploying behind a proxy run uvicorn with --proxy-headers and --forwarded-allow-ips set to the proxy.

## 11. Opening a QR link while signed out fails with an expired QR

Priority: normal. Status: open. Tags: frontend, qr.

**Where:** `app/static/app.js` pendingQr handling (around lines 241-271) and `confirmFromQr()`

**What happens:** a scanned QR link has 10 to 30 seconds left. Signed out, the student is asked to sign in first; typing email and password takes longer, so confirmFromQr posts an expired token, gets 400 "That QR has expired", and the attempt is logged as a failure (1 of the student's 5).

**Suggested fix:** sessionIdFromQr already decodes the token; also read exp. If it has passed, do not post it. Tell the student they are signed in and should scan again or type the code, and focus that class's code field.

## 12. Student page stops updating after one failed request

Priority: normal. Status: open. Tags: frontend, error-handling.

**Where:** `app/static/app.js` `renderStudent()` (around 725-729), sign-in submit (around 320), `renderProfessor()` and `renderSection()`

**What happens:** the polling intervals are registered after the first `loadLive()`/`loadStats()`. If either rejects (Wi-Fi drop, a 500 from /students/me/statistics), polling never starts, so attendance opened later never appears although the page says it appears automatically. After sign-in, `renderHome()` is not awaited: if /sections fails, the form stays with its button disabled and no message. Professor views show an empty main area on failure.

**Suggested fix:** register the intervals before the first await (the poll already catches errors), wrap first loads in try/catch with an inline error and a Retry button, and await `renderHome()` inside the sign-in try.

## 13. Accessibility: the whole app is one aria-live region

Priority: normal. Status: open. Tags: frontend, accessibility.

**Where:** `app/static/index.html` <div id="app" aria-live="polite">, countdown text in app.js

**What happens:** every view change replaces #app, the countdown text is rewritten every second and the live cards every 15 s. Screen readers (NVDA, VoiceOver) re-read the countdown and cards over and over and read the whole page on every navigation.

**Suggested fix:** remove aria-live from #app, announce through the existing #toast region or move focus to the view heading, and write the countdown only when its text changes.

## 14. Compiled Python files are tracked in git

Priority: low. Status: fixed in this pull request (`chore/untrack-pycache`). Tags: repo, fix-branch.

**Where:** `app/__pycache__/*.cpython-314.pyc`

**What happens:** seven .pyc files were committed before `.gitignore` listed __pycache__. Running the app or tests with Python 3.14 rewrites them, so they show as modified in every checkout.

**Fix:** in this pull request, branch `chore/untrack-pycache`. Removes them from the index.

## 15. Creating a class from All students stays on All students; double click makes two classes

Priority: low. Status: open. Tags: frontend.

**Where:** `app/static/app.js` `create()` in the professor sidebar (around 1063-1071), `signOut()`

**What happens:** `create()` sets the new class and tab but leaves state.allStudents true, so the All students page stays up while the toast says "Add your students next" and both entries look selected. The button is not disabled during the request, so a double click creates two classes; failures show nothing. `signOut()` does not reset allStudents or tab, so the next professor on the same browser lands on the old view.

**Suggested fix:** set state.allStudents = false in `create()`, disable the button while the request runs, add try/catch with a toast, reset allStudents and tab on sign-out and on 401.

## 16. Roster status dropdown shows the new value when saving failed

Priority: low. Status: open. Tags: frontend.

**Where:** `app/static/app.js` roster status select (around 1273-1283)

**What happens:** when the PUT fails, the catch re-enables the select but keeps the new value. The professor sees Present for a student who is still Absent.

**Suggested fix:** set select.value = row.status in the catch.

## 17. Notifications popover can open twice and one copy cannot be closed

Priority: low. Status: open. Tags: frontend.

**Where:** `app/static/app.js` notifications button (around 434-467)

**What happens:** the panel variable is assigned only after await api("/notifications"). Two quick clicks build two popovers and the first can never be closed. (The popover left over the sign-in form after a 401 is fixed on branch `fix/board-signin-expired`.)

**Suggested fix:** set a loading flag or create the panel before the await.

## 18. QR link edge cases: malformed link gives a blank page, professor tab keeps a stale QR

Priority: low. Status: open. Tags: frontend, qr.

**Where:** `app/static/app.js` qrFromHash (around 244) and `start()` (around 260-271)

**What happens:** decodeURIComponent throws on a bad link (for example /#qr=%E0, or a link cut short by a QR reader). In `start()` that call is outside the try, so nothing renders. When a signed-in professor opens a QR link, `start()` keeps pendingQr without a message; the next student who signs in on that tab gets it fired as a confirmation.

**Suggested fix:** wrap the decode in try/catch, and in `start()` use the same professor branch as the hashchange handler (clear pendingQr and show a toast).

## 19. Scanned sessions show a made-up 02:00 time and the raw method name "scan"

Priority: low. Status: open. Tags: frontend, scan.

**Where:** `app/attendance_scan.py` `confirm_scan()` (starts_at and confirmed_at = midnight UTC), `app/static/app.js` METHOD_LABEL

**What happens:** in Romania the session list and roster show scanned sessions at 02:00 or 03:00, and roster rows read "02:00, scan" because METHOD_LABEL has no entry for scan. In time zones behind UTC the previous day is shown.

**Suggested fix:** add scan: "from paper sheet" to METHOD_LABEL and show only session_date for scanned sessions, or let the professor enter the class time when saving the scan.

## 20. Per-class attendance line counts excused classes, the overall figure does not

Priority: low. Status: open. Tags: frontend, statistics.

**Where:** `app/static/app.js` per-class line (around 682-686) vs hero figure (around 528-532)

**What happens:** a student with 2 present and 1 excused in one class sees "2 of 3 attended" for the class but 100% overall.

**Suggested fix:** use sessions.length - excused as the per-class denominator, like `attendance_rate()` in the backend.

## 21. Attendance shows 0% when nothing counts yet or every class was excused

Priority: low. Status: open. Tags: backend, statistics.

**Where:** `app/main.py` `attendance_rate()`

**What happens:** with no counted sessions (a new student, or all sessions excused) the rate is 0, which reads as "missed everything".

**Suggested fix:** return null when nothing is expected (schema: float | None) and show a dash in the UI.

## 22. Roster import merges two students with the same name into one account

Priority: low. Status: open. Tags: backend, roster.

**Where:** `app/roster.py` `placeholder_email()`, `import_roster()`

**What happens:** the placeholder email is built from the name only, on purpose so one person in two groups gets one account. Two different students with the same name (common, e.g. Popescu Andrei) in different groups become one account with both enrollments, and their attendance is mixed.

**Suggested fix:** use a stable id from the export if it has one (student number), or include the group in the key and let a professor merge accounts by hand when it really is one person.

## 23. Scanning the same sheet twice creates duplicate sessions

Priority: low. Status: open. Tags: backend, scan.

**Where:** `app/attendance_scan.py` `confirm_scan()`

**What happens:** nothing checks for scanned sessions that already exist for the section and date. Uploading and approving the same photo again (or the next day by mistake) creates a second session, so every student not on the sheet gets two absences.

**Suggested fix:** when a scanned session with the same section, date and label exists, return 409 with an option to replace it, and show a warning in the scan view.

## 24. Registering the same email twice at the same moment returns 500

Priority: low. Status: open. Tags: backend, auth.

**Where:** `app/main.py` `register()`

**What happens:** the email check and the insert are separate. Two requests at the same time both pass the check and the second insert hits the unique constraint, which is not caught: 500 instead of 409.

**Suggested fix:** catch IntegrityError around the commit and return 409, as `enroll_student()` already does.

## 25. PROBLEMS.md describes a different version of the project

Priority: low. Status: open. Tags: docs.

**Where:** `PROBLEMS.md` (review date 2026-10-26), `plan.md`, `plan(1).md`, `plan(2).md`

**What happens:** `PROBLEMS.md` reviews a checkout with a React/Vite frontend/ folder, two tests and no rate limiting. The current branch has a vanilla JS UI in `app/static`, about 15 test files, rate limits, attachment uploads, a lifespan handler and professor sign-up codes, so most of its items are fixed or no longer apply. The three plan files are near copies of each other.

**Suggested fix:** update `PROBLEMS.md` to the current code (or move it to the HelpDesk and delete it), and keep one plan file.
