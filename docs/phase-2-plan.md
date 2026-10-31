# Plan: Patient-Facing App + Delivery Robot (Phase 2) — IoT Medicine Dispensing System

> ## ⚠️ Hardware half superseded
>
> The **software** half of this plan (patient app, consultations, two-gate
> verification, pharmacist scan-to-dispatch, admin site) was built and tested
> and is still current — read it as the record of what exists.
>
> The **hardware** half (line-following robot, RFID room markers, ultrasonic
> obstacle detection, track layout) was **deliberately replaced** with a fixed
> conveyor dispensing station, for demo-day reliability. See
> **[`phase-3-conveyor-plan.md`](phase-3-conveyor-plan.md)** for the current
> hardware design, revised BOM, and wiring.
>
> This document is kept as design history — the robot → conveyor decision, and
> the reasoning behind it, is worth showing in the report.

## Context

The software half of this VTU project is **built and tested end-to-end**: FastAPI backend, Next.js
admin dashboard, and a Kotlin/Compose Android app for Doctor / Nurse / Pharmacist / Technician,
with Firebase push working on a real phone. What's missing is (a) the **IoT half** — the conveyor
is still a software simulator — and (b) the **patient**, who until now had no place in the system.

This plan covers both, for a **2-room examination demo**.

### What changes conceptually

**The patient becomes an app user.** Previously the patient's QR code had nowhere real to live —
testing meant showing a QR on the laptop and scanning it with a phone, which implied an admin
walking laptop-in-hand to every room. Now the patient logs into the Android app and their QR is
simply on their own phone. A **consultation layer** sits in front of the prescription: the patient
browses doctors by speciality and requests a consultation.

**Verification splits into two gates**, each performed by whoever is *already standing there* —
nobody walks between pharmacy and ward, which is the workload saving the report's objectives claim:

| Gate | Who | Where | Checks |
|---|---|---|---|
| **Medicine verification** | Pharmacist | Pharmacy, before dispatch | Is this the prescribed medicine? |
| **Patient verification** | Nurse | Bedside, after arrival | Is this the correct patient? |

This is exactly what the Phase-1 report's block diagram describes; the current code conflates both
into one step at the pharmacy, so this change makes the implementation match the report.

### Three flow problems found and fixed

1. **Outpatient/inpatient conflict** — browsing for a specialist implies an outpatient with no bed,
   but room delivery implies an admitted patient. **Resolved**: the patient *self-checks-in* and
   picks their room (101 or 102) at sign-up, then requests consultations from that bed.
2. **Pharmacist typing the room number** was a new error source — a typo sends medicine to the wrong
   patient, the exact failure this project exists to prevent. **Resolved**: the room is auto-filled
   from the patient record and the pharmacist *confirms* it (override still possible if the patient
   was moved).
3. **Two rooms means two patients can be prescribed at once, but there is one robot.** **Resolved**:
   a FIFO job queue; the pharmacist app shows queue position.

### Decisions confirmed

- Patient **self-signs-up in the app** and picks their room at check-in; admin can reassign.
- Pharmacist **confirms an auto-filled room**, doesn't type it.
- Doctor availability is a simple **Available/Busy toggle** plus a speciality.
- Bedside scanning is done by the **nurse** with the existing app (no scanner on the robot — but the
  robot still enforces safety: its lid only unlocks once the backend reports `confirmed`).
- **2 rooms** (101, 102) for the demo.

---

## The complete flow: patient sign-up → tablets in hand

**Stage 1 — Patient signs up and checks in.** *(Android app, Patient)* Enters name, age, gender,
phone, email, password, and picks **Room 101** from a dropdown. Backend creates a `User` with role
`patient` **and** a linked `Patient` record with a unique `qr_code`. The app home screen shows their
**QR code** (this is what the nurse will scan) and a "My deliveries" tracker.

**Stage 2 — Patient finds a doctor.** Taps *Find a doctor* → a speciality grid
(**Cardiology, Gastroenterology, Nephrology, Pulmonology**) → picks e.g. Gastroenterology for
stomach pain → sees the doctors in that speciality, each with a **green (Available) or grey (Busy)
dot** → taps an available doctor, types the complaint ("stomach pain since 2 days"), and sends a
**consultation request**. Push notification → that doctor's phone.

**Stage 3 — Doctor consults and prescribes.** *(Android app, Doctor)* Has an **Available/Busy
toggle** in the top bar. Sees the incoming request, taps **Accept**, sees the patient's name, room,
age and complaint, then writes the prescription (medicine, dosage, form, qty) against that
consultation. Backend creates the prescription + a delivery in `preparing`, and pushes to the
pharmacist. **Patient's app → "Being prepared".**

**Stage 4 — Pharmacist prepares, verifies and dispatches — one person, one place.**
*(Android app, Pharmacist)* Sees the job showing **"Deliver to: Ramesh Kumar — Room 101"**.
Physically picks the tablets into the box and sticks the **medicine QR label** on it (printed from
the admin site's Prescriptions page). Then **scans that label — the scan *is* the verification**, so
there's no separate "mark prepared" tap:
- ✅ Code matches the prescription → taps **Confirm & Dispatch** on the pre-filled room → status
  `in_transit`. **Patient's app → "On the way".**
- ❌ Mismatch → status `failed`, alert pushed to the doctor, **robot is never dispatched**.

**Stage 5 — Robot loads at the pharmacy.** The robot is already parked there. Polling
`GET /conveyor/next-job` every 4 s it receives `{delivery_id, room_no: "101"}`, buzzes and opens its
lid. The **pharmacist** puts the box in — an IR sensor inside the compartment confirms a box is
actually present — and **presses the button** → lid closes and locks.

**Stage 6 — Robot drives to the ward.** Line-follows the tape, reading RFID tags and skipping
stations that aren't 101. Reports `running` / `"en route to 101"`; the **admin dashboard tile updates
live** over the existing WebSocket. Blocked for >5 s → stops, buzzes, reports `jammed`, which fires
the **technician's push notification** through code that already exists.

**Stage 7 — Robot arrives.** Reads the ROOM 101 tag → stops, buzzes,
`POST /deliveries/{id}/mark-arrived` → status `arrived`. **The compartment stays locked.**
**Patient's app → "Arrived — show your QR code to the nurse".**

**Stage 8 — Bedside patient verification.** *(Android app, Nurse — already on the ward)* The
delivery is under *Arrived – confirm delivery*. **The patient holds up their phone** showing their
QR; the nurse scans it.
- ✅ Match → status `confirmed`.
- ❌ Wrong patient → rejected, alert raised, **lid stays locked**.

**Stage 9 — Robot releases the medicine.** Polling the delivery, the robot sees `confirmed` and
**only then drives the servo to unlock the lid**. The nurse hands the tablets to the patient.
*Lead with this in the viva: the box cannot physically open until the backend has verified the
correct patient — even though the robot itself has no scanner.*

**Stage 10 — Robot returns home.** Closes the lid, line-follows to the `PHARMACY` tag, reports
`idle` / `"home"`. **Patient's app → "Delivered ✅".** The admin dashboard shows the full timeline
and the Reports page counts the delivery.

---

## Edge cases — what breaks, and how each is handled

You asked for the gaps. These are the ones worth building for; the rest are noted as limitations.

| # | Edge case | Handling |
|---|---|---|
| 1 | **Two deliveries ready, one robot** | FIFO queue: `next-job` returns the oldest `in_transit` only. Pharmacist app shows *"Robot busy — 1 ahead in queue"*. |
| 2 | **Patient not in the room / doesn't collect** | Robot waits 3 min, then returns home **with the medicine still locked inside**, sets the delivery `failed`, and raises a `delivery_not_collected` alert to nurse + pharmacist. |
| 3 | **Wrong medicine scanned** | Dispatch blocked, status `failed`, alert to doctor. Robot never leaves. |
| 4 | **Wrong patient's QR scanned at bedside** | Confirm rejected, `wrong_patient_scan` alert raised, **lid never unlocks**. |
| 5 | **Two patients sharing a room** | Room only routes the robot; the **QR decides who gets it**. Worth demoing deliberately. |
| 6 | **Patient has no room assigned** | `verify-medicine` refuses to dispatch with a clear message rather than sending the robot nowhere. |
| 7 | **Patient moved rooms after dispatch** | `Delivery.room_no` is a **snapshot taken at dispatch**, so the robot completes its original route instead of changing target mid-corridor. Pharmacist can override the room before dispatching. |
| 8 | **Box not actually loaded, button pressed** | IR sensor inside the compartment refuses to dispatch and buzzes. |
| 9 | **Obstacle in the corridor** | Stop, buzz, report `jammed` → technician push + dashboard alert. Auto-resumes when clear. |
| 10 | **Robot loses WiFi mid-route** | Retry with 5 s timeouts; after 5 consecutive failures, stop safely, buzz, and hold position rather than driving blind. |
| 11 | **Line lost** (all-white or all-black >2 s) | Stop immediately and alert — never wander off the track hunting for it. |
| 12 | **Double dispatch of the same delivery** | Status guard: only a delivery in `preparing` can be verified; only `in_transit` can be marked arrived; only `arrived` can be confirmed. Already enforced in the existing code. |
| 13 | **Doctor is Busy when a request arrives** | Request queues; patient sees *"Waiting for doctor to accept"*. |

**Known limitations to state honestly in the report** (don't hide them — examiners ask):
- A patient could screenshot and share their QR. Mitigation would be a **rotating QR** (code +
  timestamp + hash, refreshed every 30 s). Note as future work; implement only if time allows.
- No encryption on the LAN (plain HTTP) — fine for a demo, would need HTTPS in a real deployment.
- The robot has no lift, so it serves one floor.

---

## What has to change, by component

### Backend — [backend/app/](D:\Engineering Projects\backend\app)

**Model changes** (`models.py`):

| Change | Why |
|---|---|
| Add `patient` to the `UserRole` enum | Patients now log in, reusing all existing JWT/auth machinery |
| `User.is_available: bool = True` | Doctor's Available/Busy toggle. **Reuse the existing `department` column as speciality** — don't add a duplicate field |
| `Patient.user_id` → FK to `users.id` | Links the login account to the patient record |
| **New `Consultation` table** | `patient_id`, `doctor_id`, `complaint`, `status` (`requested`/`accepted`/`completed`), timestamps |
| `Prescription.consultation_id` (nullable FK) | Ties a prescription back to the consultation it came from |
| `Delivery.room_no` | **Snapshot at dispatch** — see edge case 7 |
| Add `delivery_not_collected`, `wrong_patient_scan` to `AlertType` | Edge cases 2 and 4 |

**Endpoint changes** (`routers/`):

| Endpoint | Notes |
|---|---|
| `POST /auth/signup/patient` | **Public.** Creates `User(role=patient)` + linked `Patient` with `qr_code` and chosen `room_no` |
| `GET /doctors?speciality=` | Lists doctors with speciality + `is_available`; readable by patients |
| `POST /users/me/availability` | Doctor toggles Available/Busy |
| `POST /consultations`, `GET /consultations`, `POST /consultations/{id}/accept` | New router. `GET` is role-scoped: a doctor sees requests addressed to them, a patient sees their own |
| `GET /patients/me`, `GET /patients/me/deliveries` | Patient's own record + QR image + live tracking |
| `verify-medicine` → **the pharmacist's single dispatch action** | Drop the `scanned_patient_qr` check (medicine code only); change guard from `require_roles(nurse, pharmacist, admin)` to `require_roles(pharmacist, admin)`; accept status `preparing` (not `verifying`) → sets `in_transit`; accept an optional `room_no` override and store the snapshot |
| **Delete `mark-prepared`** | The scan *is* the verification. Set both `prepared_at` and `verified_at` in `verify-medicine` so the existing admin timeline UI still renders; the `verifying` enum value simply goes unused |
| `mark-arrived` | Keep `technician` allowed — the **robot** calls this now, not the nurse |
| `POST /deliveries/{id}/mark-uncollected` | New; robot calls it on the 3-min timeout (edge case 2) |
| `GET /conveyor/next-job` | Oldest `in_transit` delivery, returning `Delivery.room_no`; `204` when idle; gated `require_roles(technician, admin)` |
| `POST /conveyor/report` | Same behaviour as `/conveyor/simulate` but honestly named for real hardware. **Extract the shared body into a helper and have both call it**, so the admin simulator panel keeps working |

`confirm` needs no change beyond raising the new alert on mismatch. Jam alerts, push notifications
and the live dashboard tile all work already.

**Database note:** several columns are added and SQLite's `create_all()` won't retrofit them.
Simplest path: **delete `backend/dev.db` and restart** — the admin account re-seeds automatically
and you recreate staff from the admin site in ~2 minutes. Current data is only test data.

### Android app — [android-app/app/src/main/java/com/medidispense/app/ui/](D:\Engineering Projects\android-app\app\src\main\java\com\medidispense\app\ui)

`HomeScreen.kt` already switches on `UserRole` — add a `patient` branch there.

| Role | Screens |
|---|---|
| **Patient** *(new)* | Sign-up (incl. room dropdown 101/102) · Home = **big QR code** + delivery tracker · *Find a doctor* speciality grid → doctor list with availability dots → request consultation · consultation status list |
| **Doctor** | **Available/Busy toggle** in the top bar · consultation requests → Accept → prescription form (existing). **Remove the "Register patient" card** — patients self-register now |
| **Pharmacist** | *Biggest change*: replace **"Mark prepared"** with **"Scan medicine QR → Confirm & Dispatch"**, showing the auto-filled patient name + room. Needs the QR scanner this screen doesn't currently use — wire in the existing `rememberQrScanner()` from `ui/common/QrScanner.kt` (already used by the nurse screen, so no new dependency). Show queue position |
| **Nurse** | *Gets simpler*: **delete the "Ready to verify & dispatch" section** (pharmacist's job now) and the manual "Mark arrived" button (robot's job now). Keep "In transit" as a read-only heads-up, and "Arrived – confirm delivery" with its single patient-QR scan. Keep a hidden manual "Mark arrived" as a no-hardware fallback |
| **Technician** | Unchanged — conveyor status + alerts |

ML Kit reads QR codes off a phone screen without trouble.

### Admin website — [admin-web/src/app/](D:\Engineering Projects\admin-web\src\app)

Mostly unchanged. Add a **Consultations** page (read-only list), show room + linked account on the
Patients page, and **keep the conveyor simulator panel** — it's your demo-day fallback.

### Robot firmware — new folder `robot-firmware/`

Arduino IDE sketch, four tabs: `robot-firmware.ino` (state machine), `config.h` (WiFi, backend URL,
robot login, pins, RFID-UID→room map), `net.cpp/.h` (API calls), `hardware.cpp/.h` (line following,
motors, RFID, ultrasonic, servo, buzzer).

| State | Behaviour |
|---|---|
| `IDLE` | Parked at PHARMACY. Polls `/conveyor/next-job` every 4 s, reports `idle` |
| `LOADING` | Job received → buzz, open lid, wait for **box sensor + button press**, lock lid |
| `NAVIGATING` | Line-follow; read every RFID tag; report `running` + `"en route to <room>"` |
| `ARRIVED` | Target tag matched → stop, buzz, `mark-arrived`. **Lid stays locked** |
| `WAIT_CONFIRM` | Poll `GET /deliveries/{id}` until `confirmed` → **unlock lid**. After 3 min → `mark-uncollected` and return home locked |
| `RETURNING` | Lock lid, line-follow to the `PHARMACY` tag → `IDLE` |
| `BLOCKED` | Obstacle <20 cm for >5 s, or line lost → stop, buzz, report `jammed`. Resume when clear |

**Line following:** weighted error across the 5 IR channels (`-2…+2`), proportional correction on the
two motor PWMs (start `Kp≈25`, base speed ~150/255; add a D term only if it oscillates).

**Networking:** plain `HTTPClient` over LAN. JWT lasts 12 h; on `401`, re-login and retry once.
**Set `http.setTimeout(5000)` on every call** or the robot freezes mid-corridor when WiFi hiccups.

---

## The device: what to build and what it costs

A **line-following delivery robot** with RFID room markers and a servo-locked medicine compartment.

### Bill of materials — **₹2,535** core

| Part | Qty | ~₹ | | Part | Qty | ~₹ |
|---|---|---|---|---|---|---|
| ESP32 DevKit V1 (30-pin) | 1 | 450 | | 18650 cells | 2 | 300 |
| 2WD chassis kit (2 BO motors, wheels, caster) | 1 | 350 | | 18650 holder | 1 | 60 |
| L298N motor driver | 1 | 140 | | LM2596 buck converter | 1 | 70 |
| 5-channel IR line array | 1 | 220 | | Active buzzer | 1 | 25 |
| RC522 RFID reader (+card, +keyfob) | 1 | 170 | | Push button | 1 | 20 |
| Extra 13.56 MHz tags (3 stations total) | 2 | 40 | | Switch, jumpers, breadboard | — | 200 |
| HC-SR04 ultrasonic | 1 | 90 | | Resistors 1 kΩ + 2 kΩ | — | 30 |
| SG90 servo (lid lock) | 1 | 110 | | Black tape + white flex sheet | — | 200 |
| IR obstacle sensor (box-present, edge case 8) | 1 | 60 | | | | |

**Optional:** 0.96" I²C OLED **+₹180** — shows *"ROOM 101 · EN ROUTE"* on the robot itself, which
reads very well to an examiner. Requires a pin swap (below).

**Buy spares (~₹500)** of the IR array, one BO motor, and the L298N. Those are what fail the night
before a demo. **Realistic total: ₹3,000–3,300**, comfortably under your ₹5,000 budget.

### What each part actually does

| Part | In plain terms | How it works |
|---|---|---|
| **ESP32** | The brain | A small computer with built-in WiFi. Runs your program ~50×/second: read sensors → decide → drive motors → talk to the backend. Everything else plugs into it |
| **2 BO motors + wheels** | The legs | One motor per wheel. It steers by **running the two wheels at different speeds** — there is no steering mechanism. Equal speed = straight; left wheel slower = turns left |
| **Caster wheel** | The third leg | A free-rolling ball that only balances the robot. Not driven, not steered |
| **L298N driver** | The muscle | An ESP32 pin supplies ~40 mA — nowhere near enough to turn a motor. The L298N takes the 7.4 V battery and switches it to the motors on command. `IN1–IN4` set direction; `ENA/ENB` take a PWM signal that sets speed |
| **5-channel IR array** | Eyes that see the line | Five downward-facing IR emitter/detector pairs. Black tape absorbs IR → reads LOW; white sheet reflects it → reads HIGH. *Which* of the five see black tells the robot whether it is centred or drifting |
| **RC522 RFID reader** | Eyes that see the stations | Reads a tag's unique ID from 2–4 cm. Mounted face-down under the robot |
| **RFID tags** | Station name-plates | Passive stickers — no battery, no wiring. Each carries a permanent factory ID. One at PHARMACY, one at 101, one at 102 |
| **HC-SR04 ultrasonic** | Eyes that see ahead | Emits an ultrasonic chirp and times the echo to compute distance. Under 20 cm means something is in the way |
| **SG90 servo** | The lock | Rotates to an exact commanded angle. 0° = latch closed, 90° = latch open. This is the only thing physically holding the medicine in |
| **IR sensor in the box bay** | Checks a box is really there | Same reflectance trick, aimed at the compartment floor. Stops the pharmacist dispatching an empty robot |
| **Push button** | "Loaded — go" | The pharmacist's single physical input |
| **Buzzer** | The voice | Beeps on arrival, on error, and on a jam |
| **2× 18650 + holder** | Fuel tank | 7.4 V for motors and electronics |
| **LM2596 buck converter** | Voltage regulator | Steps 7.4 V down to a clean 5 V for the ESP32. Motors cause large voltage dips; this stops the brain resetting every time the robot moves |

### Does it drive itself, or does someone steer it from the app?

**It is fully automatic. There is no remote control, no joystick, and no drive buttons in the app.**

The app never sends steering commands. It only creates the *job*. The robot **pulls** work down by
itself and executes it independently:

> While idle, every 4 seconds the robot asks the backend *"is there a delivery waiting?"*. If the
> answer contains a job, it starts. Nothing is ever pushed to it.

There are exactly **two human physical actions** in a whole run:

1. The **pharmacist** puts the box in and presses the button — one press meaning "loaded".
2. The **nurse** scans the patient's QR in the app — a *data* action, not a driving action.

Setting off, steering, stopping at the right room, unlocking and returning home are all decided by
the robot on its own.

### The control loop — what happens ~50 times a second

While `NAVIGATING`, each cycle (~20 ms) does four things in order:

1. **Stay on the line.** Read the 5 IR sensors → work out how far off-centre it is → speed one wheel
   up and slow the other to steer back.
2. **Look ahead.** Read the ultrasonic. Something under 20 cm for more than 5 s → stop, buzz, report
   a jam.
3. **Look down for a station.** Try to read an RFID tag. Most cycles there is nothing there.
4. **If a tag was read** → look its UID up in `ROOM_TAGS` → get the room → **is this my target?**
   No → drive straight past. Yes → **stop**.

### How "stop" actually happens

Not a brake pad — it's electrical. The firmware sets both `ENA`/`ENB` PWM values to 0 (no power) and
puts the four `IN` pins into the brake state so the motors resist coasting. The robot then buzzes,
tells the backend it has arrived, and waits — **with the lid still locked**.

### How the lid opens

The servo is the only thing holding the medicine in, and it opens on **exactly one condition**: the
backend reporting the delivery `confirmed`, which only happens after the nurse scans the correct
patient's QR.

While waiting, every 2 s the robot asks *"is delivery X confirmed yet?"*:
- Not yet → keep waiting, up to 3 minutes.
- **Confirmed** → rotate the servo, buzz, lid opens.
- 3 minutes with no confirmation → report it uncollected, keep the lid **shut**, and drive home with
  the medicine still sealed inside.

### Power flow

The 7.4 V battery splits two ways, with **all grounds tied together**:

- → **L298N → motors** (high current, electrically noisy)
- → **LM2596 → 5 V → ESP32 → sensors** (clean and stable)

That split is not optional. Running the ESP32 from the same rail as the motors makes it reset every
time the robot starts moving — the most common "my robot keeps restarting" failure.

### Does it need a black path, or can it run on plain tiles?

**It needs a guide line.** The IR sensors work by detecting a reflectance contrast — on bare tiles
there is nothing to follow. Free navigation across an open floor means SLAM: a LIDAR (₹8,000+ for the
sensor alone), a Raspberry Pi instead of an ESP32, and ROS. That's ₹20,000+ and months of work, so
it's out of scope; note it in the report as future work.

This is **defensible in the viva**: real indoor AGVs in warehouses and hospitals routinely use line
or magnetic-tape guidance precisely because it is cheap and reliable. Call it *guided navigation*.

**Two practical points:**

- The tape *can* go straight onto normal floor tiles. The real hazard is **glossy tiles** — polished
  surfaces scatter IR unpredictably and are the single most common cause of a line follower losing
  the line mid-demo.
- **Use the white flex sheet** (decided). It gives the sensors clean matte contrast on any floor,
  and — more importantly for an exam — it makes the whole track **portable**: roll it up, carry it
  to the exam hall, unroll it, running in two minutes regardless of that room's flooring.

### Track layout for 2 rooms

White flex sheet ~1.5 m × 1 m, 25 mm black tape in a **closed loop** so the robot only ever drives
forward — no 180° turns to code. Three stations, each with an RFID tag stuck flat on the track:
`PHARMACY` → `ROOM 101` → `ROOM 102` → back to `PHARMACY`. Mount the RC522 face-down ~2 cm above
the sheet (read range is only 2–4 cm). Keep curves gentle — tight corners are where line followers
overshoot.

### How the room number is actually set up

Each RFID tag carries a **factory-burned unique ID you cannot choose** (e.g. `A1B2C3D4`). The robot
learns which UID means which room from a table you write once. **One-time procedure:**

1. Stick a tag at each of the three stations on the sheet.
2. Flash the **milestone H4** sketch (RFID read + print) and hold the robot over each tag in turn.
   Each one prints its UID to the Serial Monitor at 115200.
3. Paste those UIDs into `config.h`, mapping each to its room:
   ```cpp
   struct RoomTag { const char* uid; const char* room; };
   const RoomTag ROOM_TAGS[] = {
     { "A1B2C3D4", "HOME" },   // pharmacy / charging bay
     { "11223344", "101"  },
     { "55667788", "102"  },
   };
   ```
4. Re-upload the full firmware. Done — the mapping is fixed until you move a tag.

**At run time**, the robot receives `room_no: "101"` from `GET /conveyor/next-job`, then line-follows
while reading every tag it passes. Each read UID is looked up in `ROOM_TAGS`; if the result equals
its target room it stops, otherwise it drives straight past. `"HOME"` is what it looks for when
returning.

*Trade-off, and the right viva answer:* hardcoding means adding or moving a room needs a re-flash.
That's fine for 2 rooms and it's 5 minutes of setup. If an examiner asks how this scales to 50 rooms,
the answer is to move the table into a `rooms` table in the backend, manage it from the admin
website, and have the robot fetch the map at boot — the firmware lookup stays identical, only the
source of the table changes.

### Wiring (ESP32 DevKit V1)

Chosen to avoid GPIO 6–11 (internal flash) and the awkward strapping pins — **don't reshuffle at
random**, several ESP32 pins misbehave at boot.

| Component | Pins |
|---|---|
| IR line array IR1–IR5 | 36, 39, 34, 35, 32 |
| L298N ENA/IN1/IN2 · IN3/IN4/ENB | 25, 26, 27 · 14, 13, 33 |
| RC522 SS/SCK/MOSI/MISO | 5, 18, 23, 19 · **RST → 3.3 V** |
| HC-SR04 TRIG/ECHO | 17, 16 |
| Servo · Buzzer · Button (`INPUT_PULLUP`→GND) · Box sensor | 21 · 22 · 15 · 4 |

*Optional OLED:* SDA → 4, SCL → 22, move the buzzer to **12 with a 10 kΩ pull-down** (GPIO 12 must
be low at boot), and drop the box sensor — or use an I²C expander to keep both.

**Four rules that prevent most beginner failures:**
1. **RC522 is 3.3 V only** — 5 V destroys it.
2. **Power the IR array from 3.3 V**, so its outputs don't push 5 V into ESP32 inputs.
3. **HC-SR04 ECHO needs a divider**: `ECHO →1 kΩ→ GPIO16 →2 kΩ→ GND`.
4. **Split the power, share the ground**: battery → L298N for motors; battery → LM2596 → 5 V → ESP32
   `VIN`. Running the ESP32 off the L298N's onboard regulator causes reboots the instant the motors
   draw current. **Don't hold the button while powering on** (GPIO 15 is read at boot).

### Getting code onto the robot

Arduino IDE 2.x → Preferences → add board URL
`https://espressif.github.io/arduino-esp32/package_esp32_index.json` → Boards Manager → install
**esp32 by Espressif** → Library Manager → **ArduinoJson**, **MFRC522**, **ESP32Servo** → install the
**CP2102 or CH340 USB driver** (the #1 first-time blocker on Windows) → Board: *ESP32 Dev Module*,
select the COM port → Upload. If it hangs at "Connecting…", hold **BOOT** while it connects. Serial
Monitor at **115200**. Once it works, add `ArduinoOTA` — re-plugging a moving robot gets old fast.

---

## Build order

Software and hardware can proceed in parallel. **Test each milestone before starting the next** —
don't wire everything and flash the full program, you won't be able to find the fault.

**Software track**

| # | Milestone |
|---|---|
| S1 | Model + DB changes; delete `dev.db` and re-seed |
| S2 | Patient sign-up + login + QR on the app home screen |
| S3 | Doctor speciality/availability + consultation request → accept → prescribe |
| S4 | Pharmacist scan-to-dispatch with auto-filled room; nurse screen simplified |
| S5 | Patient delivery tracker; admin Consultations page |

**Hardware track**

| # | Milestone |
|---|---|
| H1 | ESP32: blink LED, join WiFi, print IP |
| H2 | Motors drive forward/back/stop via L298N (no sensors) |
| H3 | Print IR readings, then line-follow the loop **offline** (no network) |
| H4 | Print RFID UIDs — **this is how you collect the UIDs to paste into `config.h`** |
| H5 | Ultrasonic + servo lid + buzzer + box sensor |
| H6 | Network only, wheels off the floor: login, fetch job, report status, poll delivery |
| H7 | Full state machine on the track |

**Integration:** S1–S5 + H1–H7 → full run-through, then rehearse the demo script below.

**If you run short on time**, the delivery loop is what you're being examined on. The consultation
layer can degrade to a plain doctor list without availability (Stage 2 shortens, nothing else
changes). Do **not** cut the two verification gates or the alert paths — those are the project's
entire safety argument.

---

## Examination demo script (~6 minutes)

Have all four on the same WiFi: laptop (backend + admin dashboard on the projector), staff phone,
patient phone, robot on the track.

1. **Patient phone**: sign up, pick Room 101 → QR appears. *(30 s)*
2. **Patient phone**: Gastroenterology → Dr. X shows **green** → send "stomach pain". *(30 s)*
3. **Staff phone → Doctor**: accept request, prescribe Paracetamol 650 mg ×2. *(45 s)*
4. **Staff phone → Pharmacist**: shows *"Ramesh Kumar — Room 101"* → scan the box label → **Confirm
   & Dispatch**. Point at the **projected dashboard** as the tile flips to `running`. *(45 s)*
5. **Load the box**, press the button → robot departs. Everyone watches it drive to Room 101. *(60 s)*
6. **Patient phone** now reads *"Arrived — show your QR"*. *(10 s)*
7. **Staff phone → Nurse**: scan the patient's phone → **the lid pops open**. Hand over the tablets.
   *(30 s)*
8. Robot drives home on its own. *(30 s)*

Then **deliberately demo two failures** — this is what separates a working demo from a good one:

- **Wrong medicine**: scan the wrong label at step 4 → dispatch blocked, doctor alerted, robot never
  moves.
- **Wrong patient**: at step 7 scan the *other* patient's QR → rejected, **lid stays shut**.
- *(If there's time)* put your foot in front of the robot → it stops and the technician's phone
  buzzes with a jam alert.

**Fallbacks to have ready:** a charged spare battery, a spare IR array, and — most importantly — the
admin dashboard's **conveyor simulator panel**, which drives the entire software flow from the
laptop if the hardware misbehaves on the day.

---

## Report updates

- **Ch. 3 (SRS)**: replace the hardware table with the BOM above.
- **Ch. 4 (Methodology)**: the block diagram's "Medication Delivery" and "Conveyor Failure
  Monitoring" blocks are now the robot's `NAVIGATING` and `BLOCKED` states — add the state table and
  the 2-room track layout. Note that the two-gate verification (medicine at pharmacy, patient at
  bedside) now matches the block diagram precisely. Add the consultation flow as a new sub-section.
- **One deliberate deviation to write up, not hide**: the report's "Nurse Notification" block has
  *nursing staff* compiling medication at the pharmacy. In the implemented system the **pharmacist**
  prepares and verifies at the pharmacy and the **nurse** only verifies the patient at the bedside.
  Justify in one line: each actor stays where they already work, so no staff member walks between
  pharmacy and ward — precisely the workload reduction the objectives claim.
- State honestly that the system is demonstrated on a **scale-model track with 2 rooms**, RFID
  station markers standing in for ward locations, and that the REST-over-WiFi architecture is what
  would scale to a real corridor. Include the limitations listed in the edge-case section.
