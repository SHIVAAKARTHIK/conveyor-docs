# MediDispense: automated medicine delivery for hospital wards

A VTU final-year engineering project. A doctor's prescription becomes a delivery
job, a small ESP32-driven conveyor carries the medicine box to the right bed,
and a nurse confirms it at the bedside. Every step is stored and shown live in a
web app.

This repository holds the **documentation, demos and overview**. The code is in
three other repositories:

| Part | Repository | Stack |
|---|---|---|
| Backend API | [conveyor-backend](https://github.com/SHIVAAKARTHIK/conveyor-backend) | FastAPI, SQLAlchemy, SQLite (MySQL-ready), JWT, WebSocket |
| Web app (every role) | [conveyor-admin-web](https://github.com/SHIVAAKARTHIK/conveyor-admin-web) | Next.js, React, Tailwind, TypeScript |
| Conveyor firmware | [conveyor-firmware-](https://github.com/SHIVAAKARTHIK/conveyor-firmware-) | ESP32, Arduino C++ |

## What is here

| Folder | Contents |
|---|---|
| [`docs/`](./docs) | Project report (Word and PDF), presentation (PowerPoint and PDF), and the plans: [hardware plan with the wiring](./docs/phase-3-conveyor-plan.md), [phase 2](./docs/phase-2-plan.md), [phase 4](./docs/phase-4-web-and-connectivity-plan.md) |
| [`demo/`](./demo) | Animated demos. Open the HTML files in a browser, no install needed. |

## How it fits together

```
 Web app (browser)  <--REST + WebSocket-->  Backend  <--REST-->  ESP32 conveyor
   patient, doctor,                         FastAPI              belt, 2 sensors,
   nurse, technician,                       + database           2 servos, button,
   admin                                                         buzzer
        \_______ direct local control (start, stop, dispatch, push, beep) ______/
```

The backend is the single source of truth. The ESP32 logs in as a technician
account, asks for the next job, and reports back. The browser can also control
the conveyor directly over the local network for testing.

## The flow

1. A **patient** signs up (no bed yet).
2. A **nurse** assigns Bed 1 or Bed 2.
3. The patient asks a **doctor**, who accepts and writes a prescription. That
   creates the delivery job.
4. At the conveyor the **technician** loads the box and presses the button once
   (or clicks **Dispatch** in the web app). The belt runs, the sensor of that
   bed stops it, that bed's servo pushes the box into the chute, and the
   buzzer sounds.
5. The **nurse** checks the patient's wristband and confirms the delivery.
   The patient then sees it as delivered.

Bed 1 uses sensor A and servo 1; Bed 2 uses sensor B and servo 2.

## Running it

1. **Backend:** follow the backend repo's README, then
   `python -m uvicorn app.main:app --host 0.0.0.0 --port 8000`. On first run it
   creates a default admin; change that password.
2. **Web app:** `npm install`, then `npm run dev`, and open http://localhost:3000.
   Staff accounts (doctor, nurse, technician) are created by the admin on the
   Staff page. Create one technician account for the conveyor station.
3. **Conveyor:** wire it as in the [hardware plan](./docs/phase-3-conveyor-plan.md),
   copy `config.h.example` to `config.h` in the firmware repo, fill in the WiFi
   and station login, then build and flash with `arduino-cli`.

## Status

- Backend: complete, with 8 automated end-to-end tests passing.
- Web app: patient, doctor, technician, nurse and admin interfaces built.
- Conveyor: belt, both sensors, both servos, button and buzzer verified on the
  hardware, and repeated full deliveries to both beds have completed.
- Still to do: calibrate the servo push angles, add a password to the
  conveyor's local control routes, change the default admin password, and
  move to HTTPS for anything beyond a private network.

## Notes

- Hardware parts cost about Rs. 2,210; the bill of materials is in the report
  and the hardware plan.
- This is a model-scale prototype. It is not a clinical device and has not been
  validated for use with real medicines.
