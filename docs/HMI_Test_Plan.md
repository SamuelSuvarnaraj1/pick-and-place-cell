# WinCC HMI test plan: Pick and Place cell (phase 1)

**Scope (phase 1):** a display-only operator screen (one screen, "Overview") for the Pick and Place cell. It shows what the machine is doing and whether it is faulted. It has no control buttons, so the PLC logic is unchanged.

**Setup:** TIA Portal V21, WinCC Runtime Advanced simulation on a TP1200 Comfort panel, S7-PLCSIM V21, Factory I/O 2.5.10 (S7-PLCSIM driver). Everything is simulated; no physical hardware.

## What the screen shows

The screen is a top-down view of the cell: two lanes (entry belt at the bottom, exit belt at the top, as in the Factory I/O scene) with a gripper block at the left end that swings between them.

| Screen element | PLC address | Behaviour |
|---|---|---|
| Entry belt | %Q0.0 (Entry_Conveyor) | Green when the belt runs, grey when off |
| Exit belt | %Q0.1 (Exit_Conveyor) | Green when the belt runs, grey when off |
| Gripper block, lane | %Q0.2 (X_Out) | Shown over the entry lane when 0, over the exit lane when 1 |
| Gripper block, colour | %Q0.4 (Grab_Out) | Grey when open, orange when closed (holding a box) |
| "Running" lamp | %Q0.5 (Start_Light) | Green when running |
| "Idle" lamp | %Q0.7 (Stop_Light) | Amber when idle |
| "Fault" lamp | %M11.1 (St_Fault) | Red when the machine is in fault |
| "Arm Lowered" lamp | %Q0.3 (Z_Out) | Orange when the arm is down |
| "E-Stop OK" lamp | %I1.0 (EStop_OK) | Green when healthy, red when the E-stop is pressed |

## Tests

| # | Test | Expected result | Result | Evidence |
|---|---|---|---|---|
| 1 | HMI connects to the PLC | Screen shows live PLC values, no tag errors | Passed | `docs/hmi_running.png` |
| 2 | Normal cycle | Belts, gripper lane, gripper colour and Arm Lowered lamp follow the cycle and match the Factory I/O arm | Passed | `docs/hmi_running.png`, `videos/clip5_hmi_demo.mp4` |
| 3 | Stop button mid-cycle | Idle lamp on, Running off, no fault, gripper back over the entry lane | Passed | `docs/hmi_idle.png` |
| 4 | E-stop mid-cycle | Fault lamp on, E-Stop OK lamp red, Running off | Passed | `docs/hmi_fault.png`, `videos/clip5_hmi_demo.mp4` |
| 5 | Reset and Start after E-stop | E-Stop OK green, Fault off, Running on, cycle resumes | Passed | `videos/clip5_hmi_demo.mp4` |
| 6 | Watchdog stall (simulation paused about 25 s) | Fault lamp on, E-Stop OK stays green (distinguishes it from an E-stop), recovers after Reset and Start | Passed | `docs/hmi_watchdog.png` |
| 7 | Screen values match the TIA watch table at the same moment | HMI lamps equal the PLC bits | Not run (compared visually with Factory I/O only) | |
| 8 | Alarm screen | A fault raises an alarm | Not built in phase 1 (the wizard's Alarm screen is empty; no alarms configured) | |

## Issues found and fixed while building the screen

- **Belts always showed one grey colour.** The belt rectangles' border colour was painting over the whole bar, so the fill colour never showed. Fixed by setting the border colour in each animation row to match the fill.
- **Movement animations would not accept the gripper tag.** WinCC movement animations need a numeric tag, not an on/off bit. The gripper is drawn as two blocks (one per lane) and shown or hidden with a visibility animation on X_Out.
- **Boxes can tip over in the Factory I/O scene after interrupted cycles** (E-stop or pause while the arm is carrying a box). This is a scene effect, not a PLC fault. The scene Reset button clears it.
- **WinCC Runtime shows a licence notice** (too many PowerTags for the unlicensed runtime). It is only a simulation notice and does not stop the screen.

## Not done in phase 1

- Alarm list (test 8) and a formal watch-table comparison (test 7).
- Control buttons (Start, Stop, Reset) on the screen.
