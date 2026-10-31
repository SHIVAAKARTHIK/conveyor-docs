# Ward Delivery Run — flow simulation

An interactive simulation of the Phase-2 flow: patient check-in through to
tablets in the patient's hand, with the delivery robot, both phones and the
admin dashboard all updating in step.

**This shows the system as designed, not as built.** The consultation layer,
patient login and the robot do not exist in code yet — see
`../docs/phase-2-plan.md`. Use this to explain and present the idea; it is not
a working prototype.

## Opening it

Double-click `ward-delivery-run.html` — it opens in any browser and needs no
server, no internet (fonts fall back gracefully) and no backend running.

## Using it

| Control | What it does |
|---|---|
| **Play / Pause** | Runs the whole sequence automatically |
| **Step** | Advances one stage at a time — use this when narrating to an examiner |
| **Restart** | Back to the beginning |
| **0.5× / 1× / 2×** | Playback speed |

**The four scenario buttons (top right) are the ones worth demonstrating:**

- **Normal run** — all 11 stages, ending with the robot back at the pharmacy.
- **Wrong medicine** — stops dead at stage 4. The robot never leaves.
- **Wrong patient** — the robot completes its journey but the lid **never
  opens**, and it returns home with the medicine still sealed inside.
- **Obstacle** — stops mid-corridor, alerts the technician, resumes on its own.

Showing the system *refuse* is what separates a working demo from a good one.

## Making a video from it

Screen-record the browser window (Windows: `Win + G`, or OBS) while it plays.
2× speed keeps a full run to roughly 15 seconds.
