# Pick and Place Cell: Siemens S7-1500, Ladder Logic, Factory I/O

A pick-and-place machine with a robot arm, controlled by a program I wrote in **ladder logic** for a **Siemens S7-1500 PLC**, and tested in a 3D factory simulation. Both the PLC and the machine are simulated.

**Author:** Samuel Suvarnaraj

---

## The short version

Imagine a small factory machine where boxes arrive on one conveyor belt. A mechanical arm picks each box up, swings it over, puts it down on a second belt, and goes back for the next one. Then the box rides away to be stacked on a pallet.

A **PLC** is the small industrial computer that runs machines like this. It reads sensors ("is there a box here?"), thinks through a list of steps, and switches things on and off (belts, the arm, the gripper).

This project is the "brain" of that machine. I wrote the program and tested it on a virtual copy of the machine, so nothing real could break.

The brain also looks after safety:

- **Stop button:** returns the machine safely to rest.
- **Emergency stop:** shuts everything down at once. The machine will not restart until someone clears the problem with the Reset button.
- **Watchdog (a stopwatch):** if a move takes too long, something is probably stuck, so the machine stops itself and raises a fault.

There is also a simple display screen, built with Siemens WinCC, that shows the belts, the arm and the machine's status lamps. It only shows information; it has no buttons.

## Demo videos

| Video | What it shows |
|---|---|
| [`videos/clip1_normal_cycle.mp4`](videos/clip1_normal_cycle.mp4) | The machine running its normal pick-and-place cycle |
| [`videos/clip2_panel_closeup.mp4`](videos/clip2_panel_closeup.mp4) | The control panel (Start, Stop, Reset, Auto selector) next to the running line |
| [`videos/clip3_estop_recovery.mp4`](videos/clip3_estop_recovery.mp4) | Emergency stop, fault, Reset, then a restart |
| [`videos/clip4_watchdog_fault.mp4`](videos/clip4_watchdog_fault.mp4) | A stuck move: the watchdog trips after 20 seconds, then the machine recovers |
| [`videos/clip5_hmi_demo.mp4`](videos/clip5_hmi_demo.mp4) | The WinCC operator screen next to Factory I/O: normal run, emergency stop, recovery |

---

## The engineering detail

### Tools

| Tool | Version | Used for |
|---|---|---|
| Siemens TIA Portal | V21 (trial) | Writing and downloading the program |
| Siemens S7-PLCSIM | V21 | Simulated S7-1500 CPU (CPU 1513-1 PN) |
| Factory I/O | 2.5.10, Ultimate Edition | 3D machine simulation |
| Siemens WinCC (in TIA Portal) | V21, TP1200 Comfort panel, runtime simulation | Operator screen |
| Language | Ladder (LAD) | All sequence logic (no SCL) |

### How the program works

The program is `FC_PickPlace_LAD` (FC2), called from OB1. It is a **state machine** with one memory bit per step, using Set and Reset coils.

States:
`Idle` → `Wait` → `LowPick` → `Grab` → `RaisePick` → `ExtendX` → `LowPlace` → `Release` → `RaisePlace` → `RetractX` → back to `Wait`, plus `Fault`.

Design points:

- **No end-of-travel sensors on the arm axes.** "Axis finished" is detected when the axis has moved (`SawMov`) and then stopped (`Moving_X` and `Moving_Z` both off). This gives a `MovDone` signal.
- **Timers:** a 500 ms dwell for grab and release, and a **20 s watchdog** across every active step.
- **Safety inputs wired normally closed:** Stop and E-stop read TRUE when healthy, so a broken wire fails safe.
- **Fault handling:** an E-stop or a watchdog timeout clears all step bits and sets `Fault`. Only Reset, with the E-stop healthy, clears the fault.
- **Size:** the program has 26 networks.

### I/O

| Address | Tag | Description |
|---|---|---|
| %I0.0 | Item_at_Entry | Box at end of entry conveyor |
| %I0.1 | Item_at_Exit | Box at exit conveyor |
| %I0.2 | Moving_X | Arm moving horizontally |
| %I0.3 | Moving_Z | Arm moving vertically |
| %I0.4 | Detected | Box detected under the arm |
| %I0.5 | Start_PB | Start button |
| %I0.6 | Stop_PB | Stop button (normally closed) |
| %I0.7 | Reset_PB | Reset button |
| %I1.0 | EStop_OK | Emergency stop (normally closed) |
| %I1.1 | Mode_Auto | Auto/Manual selector |
| %Q0.0 | Entry_Conveyor | Entry belt |
| %Q0.1 | Exit_Conveyor | Exit belt |
| %Q0.2 | X_Out | Arm horizontal move |
| %Q0.3 | Z_Out | Arm vertical move |
| %Q0.4 | Grab_Out | Gripper |
| %Q0.5 | Start_Light | Running lamp |
| %Q0.6 | Reset_Light | Fault lamp |
| %Q0.7 | Stop_Light | Idle lamp |

The full tag list is in [`tia-project/PickPlace_LAD_tags_full.csv`](tia-project/PickPlace_LAD_tags_full.csv), and the network-by-network notes are in [`tia-project/PickPlace_LAD_simple.md`](tia-project/PickPlace_LAD_simple.md). The ladder printout is in [`docs/FC_PickPlace_LAD.pdf`](docs/FC_PickPlace_LAD.pdf).

### What was tested

All tests were run on **S7-PLCSIM + Factory I/O** (not real hardware):

| Test | Result |
|---|---|
| Normal cycle (pick, place, retract, repeat) | Passed |
| Stop button mid-cycle: aborts to Idle with no fault, Start resumes | Passed |
| Emergency stop mid-cycle: fault, Reset light on, Reset then Start recovers | Passed |
| Stalled axis (simulation paused): watchdog faults after 20 s, Reset then Start recovers | Passed |

Screenshots from the tests:

- [`docs/watch_fault.png`](docs/watch_fault.png): the PLC watch table in the fault state (all step bits cleared, `St_Fault` set, Reset light on).
- [`docs/factory_io_driver.png`](docs/factory_io_driver.png): the Factory I/O driver, connected to S7-PLCSIM, with the input/output mapping.

### Bugs found and fixed during testing

- `SawMov` stayed set after a fault, which could skip steps on restart. Fixed by clearing it when the cycle starts.
- `MovDone` stayed true within a scan, so several steps could be skipped in one pass. Fixed by clearing it at each transition that uses it.

### Operator screen (WinCC)

A display-only screen for a simulated TP1200 Comfort panel, built in TIA Portal and run with the WinCC Runtime simulation next to Factory I/O. It shows what the PLC is doing and has no buttons, so the PLC program is unchanged.

It is a top-down view of the cell: the entry belt (lower lane) and the exit belt (upper lane), with a gripper block at the left end that moves between them.

| Screen element | PLC address | What it shows |
|---|---|---|
| Entry belt | %Q0.0 | Green when running |
| Exit belt | %Q0.1 | Green when running |
| Gripper position | %Q0.2 | Over the entry lane (0) or the exit lane (1) |
| Gripper colour | %Q0.4 | Orange when holding a box |
| Running lamp | %Q0.5 | Green when running |
| Idle lamp | %Q0.7 | Amber when idle |
| Fault lamp | %M11.1 | Red when in fault |
| Arm Lowered lamp | %Q0.3 | Orange when the arm is down |
| E-Stop OK lamp | %I1.0 | Green when healthy, red when pressed |

![Operator screen while running](docs/hmi_running.png)

![Operator screen after an emergency stop](docs/hmi_fault.png)

More screenshots: [`docs/hmi_idle.png`](docs/hmi_idle.png) (after Stop) and [`docs/hmi_watchdog.png`](docs/hmi_watchdog.png) (watchdog fault, with the E-stop still healthy).

| Test | Result |
|---|---|
| Screen connects and shows live PLC values | Passed |
| Normal cycle: belts, gripper, lamps follow the real arm | Passed |
| Stop button mid-cycle: Idle lamp on, no fault | Passed |
| Emergency stop mid-cycle: Fault and E-Stop OK lamps red | Passed |
| Reset and Start after the E-stop: running again | Passed |
| Watchdog stall: Fault lamp red, E-Stop OK still green | Passed |

Not done: an alarm list, and a formal comparison against the PLC watch table. The full list is in [`docs/HMI_Test_Plan.md`](docs/HMI_Test_Plan.md).

### How to run it

You need TIA Portal V21, S7-PLCSIM V21 and Factory I/O (Ultimate Edition, S7-PLCSIM V20-21 driver). The order matters:

1. Retrieve the TIA project from the archive in `tia-project/`, start PLCSIM, download the program, and set the CPU to RUN.
2. In PLCSIM, import and run `factory-io/FactoryIOConnectSequenceForS7PlcSim1500.xlsx`, bound to the PLC instance.
3. Open `factory-io/PickPlaceCell_Final.factoryio` in Factory I/O. In the driver (Siemens S7-PLCSIM, model S7-1500 V20+), check that the mapping is **Stop = %I0.6** and **Reset = %I0.7**, then click **Connect**.
4. Press **Play**. On the panel, press **Reset**, set the selector to **Auto**, then press **Start**.
5. To see the operator screen, select `HMI_PickPlace` in the TIA Portal project tree and click **Start simulation**. If WinCC Runtime shows a licence notice, acknowledge it; it is only a simulation notice.

## Repository layout

```
README.md
docs/           ladder PDF, screenshots, HMI test plan
tia-project/    TIA archive (.zap21, includes the HMI), tag lists, ladder notes
factory-io/     scene file, connect sequence
videos/         demo clips
```
