# Plan: Minimal Bed-Delivery Conveyor (current hardware plan)

**Supersedes the hardware half of [`phase-2-plan.md`](phase-2-plan.md).**
That plan's software (patient app, consultations, verification, admin site) is
built and tested. Only the physical delivery mechanism is planned here, and it
is deliberately kept as small as it can be while still working.

**The conveyor belt is the project.** Everything else exists to feed it a job
and show its status. Nothing is bought or built that the belt does not need.

---

## The concept in one paragraph

A doctor prescribes medicine for a patient who is in a numbered **bed**. The
prescription becomes a dispensing job. A staff member puts the medicine box on
the conveyor and presses one button. The ESP32 pulls the job from the backend
over WiFi, runs the belt, and when the box reaches the target bed's position a
servo **pushes it off the belt** into that bed's chute. The station reports
back, and the dashboard and the patient's phone update live.

Two beds, one belt, two pushers. That's the whole machine.

---

## What is in scope, and what is not

| In | Out |
|---|---|
| Conveyor belt driven by one motor | Mobile robot / line following |
| **Bed numbers** (Bed 1, Bed 2) | Room numbers, ward routing |
| 2 servo pushers, one per bed | RFID station markers |
| IR sensor to detect the box | Ultrasonic obstacle detection |
| One button to start a run | Autonomous navigation |
| Buzzer for arrival / error | Battery pack, charging, return-home |

---

## The flow

1. **Register patient** → assign **Bed 1** or **Bed 2**.
2. **Doctor prescribes** medicine for that patient. Backend creates a
   dispensing job (`preparing`) carrying the patient's bed number.
3. **The ESP32 picks the job up by itself** — it polls
   `GET /conveyor/next-job` every 4 s and receives `{delivery_id, bed_no}`.
   It buzzes once to say "job waiting" and waits.
4. **Someone puts the box on the belt and presses the button.** No app, no
   login, no scan. The ESP32 reports `in_transit` and starts the belt.
5. **The IR sensor detects the box** reaching the pusher line → belt stops →
   **that bed's servo pushes the box off** into its chute → buzzer → ESP32
   reports `arrived`. Dashboard tile and patient's phone update live.

Exactly two human actions in the whole run: the doctor writes the
prescription, and someone presses one button.

---

## Roles

There is **no QR scanning and no verification step** in this build. That makes
the role list very short:

| Role | Does |
|---|---|
| **Doctor** | Registers patient with a bed number, writes the prescription |
| **Admin** | Live dashboard, patient and prescription records |
| **Patient** | Sees "your medicine is on the way / delivered" *(optional)* |

The person who loads the box and presses the button **does not need an app at
all** — the button on the machine is the trigger. The ESP32 tells the backend
it has started, and tells it again when the box has been pushed into the bed.

Pharmacist, Nurse and Technician are **not used** in this build. Their screens
already exist and work, so nothing needs deleting — they simply aren't part of
the demo or the report's main flow.

> **Why we are not deleting them:** the software is already built and tested.
> Removing working features costs effort and gains nothing — it only reduces
> what you can show. All the remaining work is hardware, so that is where the
> simplification is worth doing.

> **Trade-off, stated honestly:** with no scan anywhere in the flow, the system
> automates *delivery* but does not *verify* that the right medicine reached
> the right patient. The report should claim automation and traceability, not
> safety. See the differentiation section at the end.

---

## Bill of materials — **~₹1,844**

**Mechanical frame + belt — bought, not hand-built:**

| Part | Source | ₹ | Why |
|---|---|---|---|
| "Move It! DIY Automatic Conveyor Belt Model Kit" | kitsguru.com | 499 | Laser-cut wood frame, real rubber belt, pulleys, and its own small DC motor. 20cm × 6.5cm × 5.2cm — replaces the foam-board/dowel/paper-belt DIY build entirely |

**Electronics — ordered from robu.in (cart confirmed 2026-09-02):**

| Part | Qty | ₹ | Why |
|---|---|---|---|
| ESP-WROOM-32 Dev Board (SKU 835800) | 1 | 399 | Brain + WiFi |
| L298N 2A Motor Driver Module (SKU 43821) | 1 | 131 | Drives the belt motor |
| 55–110 RPM BO Motor, dual shaft (SKU R258397) | 1 | 41 | **Spare/fallback** — see note below |
| TowerPro SG90 Mini Servo (SKU 5764) | 2 | 186 | One pusher per bed |
| HS-S02P IR Obstacle Sensor (SKU R223459) | **2** | 276 | One per bed's pusher line — a single sensor can't tell Bed 1 arrival from Bed 2 arrival without relying on drift-prone belt timing. **Uses a PH2.0 JST connector, not a 0.1" header** — see note below |
| SmartElex Active Buzzer Module (SKU R196225) | 1 | 42 | Arrival / error beep |
| 12x12x12mm Tactile Push Button, 10pk (SKU 618207) | 1 | 47 | "Loaded, go" |
| LM2596S Buck Converter w/ LED (SKU 699581) | 1 | 45 | 12 V → clean 5 V for the ESP32 |
| Standard 12V 2A Wall Adapter, 5.5mm plug (SKU R199671) | 1 | 389 | Steadier than batteries for a machine that never moves — fully enclosed, plugs straight into the wall |
| 10cm Dupont jumper combo, M-M/M-F/F-F (SKU 76277) | 1 | 129.69 | Wiring |
| USB-A to Micro-USB cable, 1m (SKU 353307) | 1 | 25 | Flashing firmware from Arduino IDE — the ESP32 board has no USB-C port |

**Electronics subtotal: ₹1,710.69** (robu.in cart total, GST included).
Several of the original picks went out of stock between planning and
ordering and were swapped for in-stock equivalents — the IR sensor and the
wall adapter came in meaningfully higher than first estimated.

**Grand total ≈ ₹2,210** (₹499 KitsGuru kit + ₹1,710.69 robu.in electronics).
No local-shop frame materials needed anymore — the kit supplies the frame
and belt. Keep a little glue/a few small screws on hand for mounting the
servos and IR sensors onto the wood panels.

> **IR sensor connector note:** the HS-S02P uses a small PH2.0 JST
> connector, not the 0.1"-pitch pin header the Dupont combo expects. Either
> snip the connector and solder the 3 wires directly (the board also has
> through-hole pads next to the connector), or use a short PH2.0-to-Dupont
> adapter cable if you'd rather not solder. Flagged now so it isn't a
> surprise at assembly time.

**Prop to put on the belt:** roughly matchbox-sized, **~4cm × 3cm × 1.5cm**
(a small paper/card "tablets" packet), so it clears the belt's 6.5cm width
with margin and leaves enough of the 20cm length for a load zone plus two
pusher/IR stations spaced ~6–7cm apart.

> **Why the BO motor stays in the order as a spare, not dropped:** the kit's
> motor is built to run off 2×AA (≈3V), but this design drives the belt
> motor from the L298N off the 12V rail via PWM. That's normally fine —
> PWM at a low duty cycle keeps the *average* voltage the small motor sees
> well under 12V — but it's untested until the parts are in hand. Keep the
> ₹41 BO motor as cheap insurance: try the kit's own motor first, and swap
> to the BO motor only if it struggles or overheats.
>
> **Why 2 IR sensors, not 1:** with one shared sensor at a single point, the
> firmware would have to guess which bed a box is passing by counting elapsed
> belt time — that drifts as the belt loosens or the motor slows under load.
> One sensor per pusher (Bed 1 line, Bed 2 line) makes "is a box here right
> now" a direct read instead of a guess.

### Deliberately not buying

| Skipped | Why |
|---|---|
| 2WD chassis kit (₹479) | Nothing moves — the kit's own motor (or the ₹41 BO motor as fallback) drives the belt |
| 5-channel IR line array (₹194) | No line to follow |
| RC522 RFID + tags (₹95) | Beds are fixed positions, not scanned markers |
| HC-SR04 ultrasonic (₹69) | Nothing to avoid |
| 18650 pack + holder (₹627) | Wall adapter is better for a stationary machine |
| Resistor kit (₹174) | Only needed for the HC-SR04 divider, which is gone |
| Transistor/diode kit (₹529) | Never used by this design |

That is **₹2,167 not spent**, and a lot less to assemble and debug.

---

## Physical build

```
        load here
            ↓
   ┌────────────────────────────────────┐
   │  ▣ ─── belt ──────────────────▶    │
   └────────┬─────────────┬─────────────┘
       [servo 1]     [servo 2]
            ↓             ↓
        ┌───────┐     ┌───────┐
        │ BED 1 │     │ BED 2 │
        └───────┘     └───────┘
```

- **Rollers, belt, frame:** all supplied by the KitsGuru "Move It!" kit —
  laser-cut wood panels, real rubber belt, pulleys. Assemble per its
  instruction manual first, as a plain motorised belt, before adding
  anything below.
- **Motor wiring:** disconnect the kit's AA battery holder; wire the same
  motor's two leads into the L298N's output terminals instead, so the
  ESP32 controls start/stop/speed instead of a battery switch.
- **Pushers:** a small card/foam paddle glued to each servo horn, mounted
  through a small drilled hole in the wood frame at the pusher line,
  sweeping across the belt to shove the box sideways into that bed's chute.
- **Chutes:** the kit has no built-in sorting — glue two small foam-board or
  card funnels to the frame's sides at the Bed 1 and Bed 2 pusher lines to
  catch the box as it's pushed off.
- **IR sensor:** mounted so the box breaks its beam right at each pusher line.
- **Prop:** ~4cm × 3cm × 1.5cm, matchbox-sized, to clear the belt's 6.5cm
  width with margin.

Keep the belt **slow** — a fast belt slides and tips the box. Tune
`BELT_SPEED` in `config.h` down until it rides steadily.

---

## Wiring (ESP32 DevKit V1) — 9 signal pins total

| Component | Pin |
|---|---|
| L298N `ENA` / `IN1` / `IN2` | 25, 26, 27 |
| Bed 1 servo | 18 |
| Bed 2 servo | 19 |
| IR sensor A (Bed 1 line, `IR1` in firmware) | 4 |
| IR sensor B (Bed 2 line, `IR2` in firmware) | 32 |
| Buzzer | 23 |
| Push button (`INPUT_PULLUP` → GND) | 15 |

### Common power and ground rails (the ESP32 has two GND pins, one VIN, one 3V3)

1. **Ground one** — the green-wire rail: both sensors' G (green) wires, the
   button's other leg and the buzzer's GND (yellow) wire, all joined and taken
   to one ESP32 GND pin. *(Decided by the builder, 2026-10-24.)*
2. **3V3 rail** — the `3V3` pin goes to one brown wire, and that wire joins the
   V (brown) wires of sensor A and sensor B. *(Decided by the builder, 2026-10-24.)*
3. **Ground two** — the other ESP32 GND pin goes to one brown wire, which joins
   servo 1's ground (brown), servo 2's ground (brown) and the L298N GND wire.
   *(Decided by the builder, 2026-10-24.)* Note brown is ground on the servos but
   power (3V3) on the sensors — don't mix them up.
4. **VIN rail (5 V)** — the `VIN` pin goes to one red wire, which joins servo 1's
   power (red), servo 2's power (red) and the buzzer VCC (white). *(Decided by the
   builder, 2026-10-24.)* VIN is power; `VN` is only a signal pin.

**Three rules that prevent most failures:**

1. **Split the power, share the ground.** 12 V → L298N for the motor;
   12 V → LM2596 → 5 V → ESP32 `VIN` and the servos. Running the ESP32 off
   the L298N's onboard regulator makes it reboot when the motor starts.
2. **Servos take 5 V power, not an ESP32 pin.** Only the signal wire goes to
   the GPIO.
3. **Don't hold the button while powering on** — GPIO 15 is read at boot.

---

## Software changes needed

Small and mechanical — everything else already works:

1. **`room_no` → `bed_no`** across backend, Android app, and admin site, with
   values `1` and `2` instead of `101`/`102`. A rename, covered by the
   existing test suite.
2. **Bed picker** on patient registration / signup shows *Bed 1 / Bed 2*.
3. **`GET /conveyor/next-job`** returns the oldest **`preparing`** delivery as
   `{delivery_id, bed_no, patient_name}` (today it returns the oldest
   `in_transit` one, which assumed a pharmacist had already dispatched it).
4. **`POST /deliveries/{id}/dispatch`** — new, no payload, no scan. Called by
   the ESP32 when the button is pressed: `preparing` → `in_transit`.
5. **`POST /deliveries/{id}/mark-arrived`** — already exists. Called by the
   ESP32 after the servo push: `in_transit` → `arrived`.
6. **`POST /deliveries/{id}/receive`** — new (web-only build). The *patient* taps
   "I received it" on their page once the delivery has arrived: `arrived` →
   `confirmed`, and the prescription becomes `delivered`. Only the delivery's own
   patient can do it (anyone else gets a 404). Replaces the nurse's QR scan.
7. **`POST /deliveries/{id}/fail`** — new. A technician or admin (or the firmware
   on a timeout/emergency stop) closes a delivery that is still open
   (`preparing`/`verifying`/`in_transit`) as `failed`; the prescription is marked
   failed too. Without it a stopped run stayed "in transit" forever. The
   technician page lists open deliveries with a "Mark failed" button.

Both new endpoints are covered by tests (7 backend tests pass). The existing
scan-based endpoints (`verify-medicine`, `confirm`) stay in the codebase and keep
working — they just aren't part of this flow.

No deletions.

---

## Firmware milestones

| # | Milestone | Status |
|---|---|---|
| H1 | WiFi joins, prints IP, LED heartbeat | ✅ written, compiles |
| H2 | Belt runs forward/back/stop, Serial-controlled | ✅ working on hardware (jumper off, 1000 Hz PWM, speed 90) |
| H3 | IR sensor prints when a box breaks the beam | ✅ both sensors soldered (wires onto the connector legs) and verified: A = D4 (Bed 1), B = D32 (Bed 2); each reacts cleanly to a hand and to the box on the belt |
| H4 | Both servos sweep — calibrate rest and push angles | 🟡 servo 1 (D18) mounted for Bed 1 and verified: 5 confirmed-clean box pushes in belt trials (4 repeat runs plus the button-started run); one slow run was never confirmed either way. No misses reported. Servo 2 (D19) verified moving but not yet mounted/run with a box. Angles still at defaults (0/90) |
| H4b | Bench test: button starts the belt, sensor stops it, servo pushes, buzzer beeps | ✅ Bed 1 verified end to end on hardware (toggle with `t` over Serial; resets OFF on reboot). Bed 2 not yet run |
| H5 | Button starts a run, buzzer beeps on completion | ✅ button (D15↔GND, debounced) and buzzer (SIG=D23, VCC=VIN, GND; red wire is NC) both verified on hardware |
| H6 | Full loop: poll job → wait for button → run belt → detect box → push at the right bed → report delivered | 🟡 **Written and compiles; NOT yet flashed or run on hardware** (the ESP32 was unreachable when flashing was tried, 2026-10-24). Backend side verified: the five backend tests pass (including the exact station sequence), and the machine's technician login in `config.h` was tested against the live backend. Bed-aware: a Bed 2 box ignores sensor A. Finds the laptop by name (`<name>.local`) with the `config.h` address as a fallback. Serial `h` toggles it; bench test `t` switches it off. Reports status to `/conveyor/simulate` (a jam/emergency stop raises an alert). 40 s timeout if no box arrives; on a timeout or emergency stop it also closes the delivery with `POST /deliveries/{id}/fail`. **This `fail` call was added after the last compile check of the earlier build and has not been run on hardware.** |

### H6 state machine

| State | Behaviour |
|---|---|
| `IDLE` | Poll `/conveyor/next-job` every 4 s |
| `LOADING` | Job received → buzz once, wait for **button press** |
| `RUNNING` | Belt on, report `running` |
| `PUSHING` | IR triggers → stop belt, sweep that bed's servo, return it to rest |
| `REPORTING` | Report delivered → back to `IDLE` |

Set `http.setTimeout(5000)` on every HTTP call, or the station freezes when
WiFi hiccups.

---

## Demo script (~3 minutes)

Laptop (backend + dashboard on the projector), one phone, the conveyor.

1. **Doctor** registers a patient in **Bed 1**, writes a prescription. *(45 s)*
2. Dashboard tile shows a new job. **Staff** puts the box on the belt and
   presses the button. *(20 s)*
3. Belt runs, box travels, **servo 1 pushes it into the Bed 1 chute**, buzzer
   sounds. Dashboard flips to delivered. *(30 s)*
4. Repeat for **Bed 2** to show it routes to the correct bed, not just the
   first one. *(45 s)*

**Fallback:** the admin dashboard's conveyor simulator panel drives the whole
software flow from the laptop if the hardware misbehaves on the day.

---

## What makes this different from the 2024 department project

A similar conveyor project exists in this department (Flask + Arduino Uno over
USB). The differences here are real and cost nothing extra, because the
software is already built:

| | Prior project | This project |
|---|---|---|
| Link to the machine | Arduino **tethered to a laptop by USB cable** | **ESP32 over WiFi** — pulls its own jobs from a REST API, no cable |
| Backend | Flask + pySerial | FastAPI, REST + WebSocket, automated test suite |
| Clients | One web page | Next.js admin dashboard **+ a Kotlin/Compose Android app** |
| Patient | Not a user | Has a login and sees their own delivery status |
| Live updates | Manual refresh | WebSocket dashboard + Firebase push |
| Target | Two fixed trays | **Bed-addressed** delivery selected per job |

One-line viva answer: *"theirs automated the machine over a USB cable; this is
a networked device that pulls its own work and reports its own status."*

Limitations to state honestly:

- Two beds at model scale; delivery reaches a bedside chute, not the patient's
  hand.
- **No verification step.** The system does not check that the loaded box
  actually matches the prescription, or that the right person collects it. It
  automates and logs delivery; it does not make delivery safer. If an examiner
  asks how you would add that, the answer is a QR scan at load time checked
  against the prescription — the backend already supports it
  (`POST /deliveries/{id}/verify-medicine`), it is simply not used in this
  build.
- Plain HTTP over a LAN, no encryption.
