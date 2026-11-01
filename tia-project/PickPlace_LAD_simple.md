# FC_PickPlace_LAD: network-by-network notes

Plain-English notes for the 26 networks of `FC_PickPlace_LAD [FC2]`, called from OB1.
Normally closed inputs (Stop_PB, EStop_OK) read TRUE when healthy.

## Fault, stop and housekeeping
1. **Fault reset.** Fault + Reset pressed + E-stop healthy: clear Fault.
2. **Fault entry.** E-stop pressed or watchdog expired, and not already faulted: clear all 10 step bits (RESET_BF) and set Fault.
3. **Saw movement.** Arm moving in X or Z: remember it (SawMov).
4. **Movement done.** SawMov and arm no longer moving in X or Z: MovDone.
5. **Watchdog.** Timer runs 20 s whenever the machine is not Idle, Wait or Fault. If a move takes too long, it fires.
6. **Dwell.** 500 ms timer during Grab and Release.
7. **Stop button.** Stop pressed while not Idle: clear all 9 step bits, so the cell returns to Idle (no fault).
8. **Start.** Idle + Start + Auto + Stop healthy + E-stop healthy: go to Wait, clear SawMov.

## State sequence
9. Wait + box at entry: go to LowPick.
10. LowPick + MovDone: go to Grab.
11. Grab + dwell done + box detected: go to RaisePick.
12. RaisePick + MovDone: go to ExtendX.
13. ExtendX + MovDone + box at exit sensor free: go to LowPlace.
14. LowPlace + MovDone: go to Release.
15. Release + dwell done: go to RaisePlace.
16. RaisePlace + MovDone: go to RetractX.
17. RetractX + MovDone: back to Wait.

## Outputs
18. **Idle** = no step bit and no Fault set.
19. **Entry conveyor** runs in Wait while no box is at the entry.
20. **Exit conveyor** runs in Wait, LowPick, Grab, RaisePick, RaisePlace, RetractX.
21. **X_Out** on in ExtendX, LowPlace, Release, RaisePlace.
22. **Z_Out** on in LowPick, Grab, LowPlace, Release.
23. **Grab_Out** on in Grab, RaisePick, ExtendX, LowPlace.
24. **Start_Light** on when not Idle and not Fault.
25. **Reset_Light** on in Fault.
26. **Stop_Light** on in Idle.
