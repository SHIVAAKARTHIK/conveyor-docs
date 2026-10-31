# Plan: Staff web pages, live hardware control, and WiFi resilience

Builds on [`phase-3-conveyor-plan.md`](phase-3-conveyor-plan.md), which covers the
belt itself. This plan covers four separate asks:

1. **The system goes web-only.** Doctor, Technician, Nurse, *and Patient* all
   move to `admin-web`. Confirmed 2026-09-22: the Android app is **kept in the
   repo, unused** — same treatment already given the Nurse/Pharmacist Android
   screens — not deleted, not actively developed further. It remains a valid
   thing to mention in the viva ("we built both, standardized the live demo
   on the web so everything is on one screen").
2. Web pages for **Doctor**, **Technician**, **Nurse**, and **Patient** in
   `admin-web`.
3. A real way to **send start/stop (and other commands) to the physical ESP32**
   over WiFi — today that only happens over a USB-tethered Serial link.
4. **Reconnecting the ESP32 to a new WiFi network without reflashing** — the
   exact pain from the SSID/password change a few sessions back.

**Why web-only fits now specifically:** the strongest usual reason for a
native app — camera-based QR scanning — doesn't apply anymore, since scanning
was already dropped from this project's flow ("no scanning at all"). What's
left for a patient to do (browse doctors, request a consultation, watch a
delivery status) is a plain web page, nothing a browser can't do.

**Report note, not yet actioned:** `docs/report/Chapter-3-System-Requirements.docx`
currently describes Android as part of the stack and justifies it in §3.5.
That will need a small update once the web pages are built — flagged here so
it isn't forgotten, not fixed in this pass.

---

## What already exists (checked before writing this)

- **Backend**: every endpoint these pages need is already built and tested —
  `POST /auth/signup/patient`, `POST /users/me/availability`,
  `GET /doctors`, `POST /consultations`,
  `GET/POST /consultations/...`, `GET /conveyor/status`,
  `POST /conveyor/simulate`, `GET/POST /alerts/...`, `GET/POST /deliveries/...`
  (already patient-scoped, per the privacy fix from phase-2). **No backend
  changes needed** for any of the four web pages — this is a frontend
  addition only.
- **Android app**: Doctor, Nurse, Pharmacist, Technician *and Patient* screens
  are all already built and working
  (`android-app/.../ui/{doctor,nurse,pharmacist,technician,patient}/`). The
  web pages are a **second front end for the same backend**, not new
  functionality — a browser now does what the phone app already does. Per
  the decision above, Android stays in the repo but stops being the thing we
  build new features on.
- **admin-web**: has `dashboard`, `patients`, `prescriptions`, `consultations`,
  `alerts`, `reports`, `users` (staff management) — all admin-facing. Nothing
  role-specific, and nothing public/patient-facing yet.

**One scope note, stated plainly:** Nurse and Pharmacist are **not part of the
live conveyor flow** (the physical button replaces the nurse's bedside scan,
per the phase-3 decision). Giving Nurse a web page doesn't change that — it's
parity with the Android screen that already exists and already isn't used in
the demo. If you want the nurse's scan step back **in the live flow**, that's
a different, bigger conversation — say so and we'll plan that separately.

---

## Part 1 — Patient, Doctor, Technician, and Nurse pages in admin-web

Each is a **role-gated page**: the logged-in user's role decides what they see,
the same way `ProtectedPage` already gates the existing pages. Add a role
check so a doctor visiting `/technician` gets redirected, not an error.

The Patient pages are different from the rest in one way: they need to be
reachable **without a staff login** (signup is public, and a patient's own
account is not a staff account). Route them separately from the admin-gated
pages — e.g. `/signup` and `/patient/*` check for a `patient`-role JWT instead
of the admin/staff check, rather than reusing `ProtectedPage` as-is.

### ✅ `/signup` (public) and `/patient` — mirrors the Android patient screens — built, verified live

| Page | Backend call | Behaviour |
|---|---|---|
| `/signup` | `POST /auth/signup/patient` | Name, age, gender, phone, email, password. No bed: a nurse assigns Bed 1 or Bed 2 afterwards |
| `/patient` (home) | `GET /patients/me`, `GET /deliveries` (already patient-scoped) | "Your medicine is being prepared / on the way / delivered" — live status, no QR needed since scanning is dropped |
| `/patient/find-doctor` | `GET /doctors?speciality=` | Speciality grid → doctor list with Available/Busy dot → tap to request |
| `/patient/consultations` | `POST /consultations`, `GET /consultations` (own) | Complaint text box on request; status list afterward |

Verified 2026-09-22 end to end in the browser: signup → login/logout → role
redirects (`/dashboard` bounces a patient back to `/patient`) → find a doctor
→ request sent → shows up for the doctor.

### ✅ `/doctor` — mirrors `DoctorScreen.kt` — built, verified live

| Section | Backend call | Behaviour |
|---|---|---|
| Availability toggle | `POST /users/me/availability` | Available/Busy switch in the header |
| Incoming consultation requests | `GET /consultations` (role-scoped), split into Requested / Accepted / Completed | Accept button, then Prescribe button once accepted |
| Prescribe | `POST /prescriptions` | Modal: one or more medicines (name, dosage, form, qty) + notes, against the accepted consultation |
| Patients | `GET /patients` | Read-only list — every patient, not doctor-scoped (there's no doctor-patient link beyond consultations, so "My patients" would have been misleading) |

Verified 2026-09-22: accept → prescribe → the patient's own delivery tracker
picked it up immediately, showing "Being prepared".

### ✅ `/technician` — mirrors `TechnicianScreen.kt` — status/alerts/simulator built and verified live; hardware panel still pending Part 2

| Section | Backend call | Behaviour |
|---|---|---|
| Live conveyor status | `GET /conveyor/status`, live-updated over the existing `useLiveChannel("conveyor", ...)` WebSocket hook (same one the dashboard uses) | Idle / Running / Jammed tile |
| Alerts | `GET /alerts`, `POST /alerts/{id}/acknowledge` \| `resolve`, live over `useLiveChannel("alerts", ...)` | Open → Acknowledged → Resolved, three sections |
| Software simulator | `POST /conveyor/simulate` | Existing fallback — drives status without touching real hardware |
| **Live hardware control** *(placeholder — see Part 2)* | Direct call to the ESP32, not the backend | Section is on the page, clearly marked "not built yet" — wires up once the ESP32's local `WebServer` + mDNS exist |

Verified 2026-09-22: simulated a jam → alert appeared **without a manual
refresh** (confirms the WebSocket wiring, not just polling) → acknowledged →
resolved → conveyor status reset to idle. All three alert-list sections
(Open/Acknowledged/Resolved) render correctly as an alert moves through them.

### `/nurse` — mirrors `NurseScreen.kt`

Kept for parity and as a fallback if the scan-based flow is ever turned back
on. Shows deliveries in `arrived` state with a QR-scan-style confirm — on web,
a scan can either use the device camera (`getUserMedia` + a JS QR-reading
library) or, more realistically for a desktop browser, a manual code-entry
fallback. Given it's not in the live flow, **build the simple manual-entry
version first**, add camera scanning only if you actually want to demo it.

### Suggested build order
Patient signup + home first — without it there's no one to generate a job for
the rest of the demo. Then Doctor (used in the real flow). Then Technician's
status + alerts (all existing endpoints, fastest to build). Then the live
hardware control panel (needs Part 2 done first). Nurse last (lowest
priority, not in the live flow).

---

## Part 2 — Actually sending start/stop to the hardware

### The gap, stated honestly
Right now "sending a command" means: I open a PowerShell session, talk to
`COM3` over the USB cable, and type `f`/`s`. That only works because the ESP32
is physically tethered to this laptop. **It is not how the finished project
works**, and it's not something a web page can do — browsers can't open a
serial port to a USB device.

The **production** design (from phase-3) doesn't need remote start/stop at
all: the ESP32 *pulls* jobs from `GET /conveyor/next-job`, and a physical
button starts the run. That's correct and stays as-is. What's missing is a way
to **test and demo the hardware from a web page**, the way we've been doing
by hand over Serial.

### The fix: a tiny web server *on the ESP32 itself*

The ESP32's WiFi is already connected for polling — add a few HTTP routes it
serves locally, using the core's built-in `WebServer` library (no new
hardware, no backend changes):

| Route | Effect |
|---|---|
| `GET /status` | Returns JSON: belt running?, speed, last IR reading, WiFi RSSI |
| `POST /start` | `beltRun(beltSpeed)` |
| `POST /stop` | `beltStop()` |
| `POST /push?bed=1` \| `2` | Runs that bed's servo push cycle |
| `POST /beep` | Test the buzzer |

The **Technician page's "Live hardware control" panel** calls these directly
— your laptop's browser and the ESP32 are already on the same WiFi network,
so this is a plain `fetch()` to the device, no backend round-trip, no added
latency. This replaces every PowerShell command we've been running by hand.

**One thing this needs to work smoothly: the ESP32's IP keeps changing**
(it's been `.100`, `.103`, and will change again). Fixing that is Part 3's
mDNS piece — solve them together.

**Security note, for the report:** these routes have no authentication and
work for anyone on the LAN. Fine for a demo on a private WiFi network; call it
out as a stated limitation, same as the plan already does for plain-HTTP
between the other components.

### ✅ Status: built, flashed, and verified (2026-09-25)

The ESP32 now serves these routes on port 80, reachable at `http://conveyor.local`
(mDNS) as well as its IP:

| Route | Result |
|---|---|
| `GET /status` | JSON: `belt_running`, `belt_speed`, `ir1`, `ir2`, `servo1_deg`, `servo2_deg`, `wifi_rssi`, `ip` |
| `POST /start` / `POST /stop` | Runs / stops the belt |
| `POST /push?bed=1\|2` | Runs that bed's push cycle (400 with a clear message for a missing or invalid bed) |
| `POST /beep` | Buzzer test |
| anything else | JSON 404; a `GET` on a `POST` route is refused, so the belt can't be started from an address bar |

Verified with `curl` **and** with a real `fetch()` from the admin site's own
origin (`http://localhost:3000`) — the CORS preflight headers are correct, and
`/start` flipped `belt_running` false → true → false as reported by `/status`.

**Not yet verified against real hardware:** the test was run with only the USB
cable connected (no L298N, buzzer or servos wired at the time), so the routes'
*logic* is proven but a physical belt/servo/buzzer response over HTTP still
needs a check with everything wired. Nothing about the code is expected to
differ — the routes call the same functions the Serial commands already
exercise on the real machine.

✅ **Done and verified on hardware (2026-10-10):** the "Live hardware control"
panel (`admin-web/src/components/HardwareControl.tsx`) is on the Technician page
and, so the admin can use it too, on the admin Dashboard. It polls `/status`
every 1.5 s and sends Start, Stop, Push Bed 1/2 and Beep. Tested from the web
app against the real conveyor: Start belt, Stop belt, Push Bed 1 and Beep all
worked, and a web-started Bed 1 run completed end to end (belt ran, sensor A
stopped it, servo 1 pushed the box cleanly, delivered beep). Bed 2 not run yet.

Known limits: the board's DHCP address changes (seen .107, .105, .103, .104),
so the panel defaults to `http://conveyor.local` and has an editable address box;
a router DHCP reservation would stop the drift. Plain http only, so the page must
not be served over https.

### Firmware changes this needs
- Add `#include <WebServer.h>`, start a `WebServer server(80)` in `setup()`,
  register the routes above, call `server.handleClient()` in `loop()`.
- Small — the belt, servo and buzzer functions already exist and just get
  called from the new route handlers instead of from `handleBeltCommand()`.
  Keep the Serial commands too; they're still useful for bench testing.

---

## Part 3 — Reconnecting WiFi without reflashing

### The problem, stated honestly
Two sessions ago, the router's SSID gained a trailing space and then the
password changed entirely. Both times, fixing it meant: edit `config.h`,
recompile, and re-flash over USB. That's a real weakness for a device that's
meant to just sit on a shelf and work — and it'll happen again the next time
this WiFi changes, or when the project is demoed on a different network (like
the exam hall's).

### The fix: `WiFiManager` (captive portal), replacing hardcoded credentials

1. Add the `WiFiManager` library (`tzapu/WiFiManager` — the standard one for
   this, works with the ESP32 core already installed).
2. On boot, it tries the last-saved WiFi (stored in the ESP32's own flash, not
   in `config.h`). If that fails within ~10 seconds, it **starts its own WiFi
   access point** named something like `MediDispense-Setup`.
3. Connect a phone to that AP. A setup page opens automatically (or open
   `192.168.4.1`), listing nearby networks — pick one, type the password, hit
   save. The ESP32 reboots and joins it.
4. `config.h`'s `WIFI_SSID`/`WIFI_PASSWORD` become just the **first-time
   default** — WiFiManager takes over after that, and the credentials it
   saves survive reboots and reflashes of the *sketch* (they live in a
   separate flash partition).

### Add mDNS at the same time (small, same session's work)
`#include <ESPmDNS.h>`, `MDNS.begin("conveyor")` in `setup()`. The device is
then always reachable at **`http://conveyor.local`**, regardless of what IP
it's actually been given. This is what the Technician page's live-control
panel (Part 2) should call — not a hardcoded IP that breaks every time the
router hands out a new one.

### A nice-to-have, not required for the exam
Wire the existing start button (`D15`) as a "hold 5 seconds at boot to force
setup mode" trigger, so you can re-open the WiFi setup portal on demand
without needing to power-cycle onto a network that no longer exists.

---

## Summary: what changes where

| Where | What |
|---|---|
| `admin-web` (frontend only) | 5 new pages: `/signup`, `/patient`, `/doctor`, `/technician`, `/nurse` |
| `backend` | **No changes** — every endpoint already exists |
| `conveyor-firmware` | Add `WebServer` (local HTTP control), `WiFiManager` (no-reflash WiFi setup), `ESPmDNS` (stable hostname) |
| `android-app` | **Kept, unused.** No further feature work; stays in the repo as a built-and-working second client, not deleted |
| `docs/report/Chapter-3-System-Requirements.docx` | Needs a small update once the web pages exist — §3.3.3 (Android) and §3.5's justification — flagged, not done yet |

## Suggested order
1. **mDNS + WiFiManager first** — small, self-contained, and directly fixes
   the pain we already hit twice. Do this before anything else touches the
   firmware again.
2. **Patient signup + home** — pure frontend, no hardware needed, and nothing
   else in the demo has a job to work with until a patient exists.
3. **The ESP32's local web server** (`/start`, `/stop`, `/push`, `/status`) —
   replaces the PowerShell-over-Serial workflow with real HTTP.
4. **Doctor page** — pure frontend, can happen any time, doesn't block on
   hardware.
5. **Technician page** — status + alerts (existing endpoints) first, then the
   live-control panel once step 3 is flashed and reachable at
   `conveyor.local`.
6. **Nurse page** — lowest priority, manual-entry version only unless you
   want camera QR scanning for the demo.
