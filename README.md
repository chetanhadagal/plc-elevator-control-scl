# Elevator (Lift) Control System Simulation Using Siemens TIA Portal and SCL

The Elevator Control System is a comprehensive simulation project developed using **Siemens TIA Portal** with **Structured Control Language (SCL)** to model real-world elevator behavior. The primary goal was to implement a fully functional lift system that can respond to user requests, move between floors, and control door operations — while incorporating safety interlocks, fault management, a **WinCC Unified HMI** interface, and a complete alarm and event logging system.

## What it does

- **Floor request handling** — manages requests from both cabin buttons and external hall call buttons, storing them in dedicated request flags for each floor
- **LOOK algorithm dispatch** — dynamically selects the target floor based on pending requests and current position, serving floors in the current direction of travel before reversing
- **Motor control** — ensures the elevator moves in the correct direction, stops precisely at the target floor, and adjusts speed depending on distance to target
- **Movement simulation** — assumes 5 seconds per floor, updating the elevator's virtual position and activating floor-specific sensors
- **Automated door control** — timer-driven doors that open for a fixed duration at each stop
- **Safety interlocks**:
  - Prevents elevator movement when doors are open
  - Mutual exclusion logic to avoid conflicting door signals
  - Door obstruction detection
  - Manual door open/close override via cabin buttons
- **Emergency stop** — fault latch with manual reset pattern to safely halt and resume the system under fault conditions

## How it works

The logic runs every scan cycle:

1. **Initialise on first scan.** Resets floor, requests, buttons, and outputs exactly once using an `Initialised` latch.
2. **Latch faults from E-Stop, overload, or drive fault.** While the fault is active, motor outputs are forced off and the rest of the scan is skipped with `RETURN`.
3. **Store floor requests** from hall call and cabin buttons into per-floor request flags, unless the lift is already idle at that floor.
4. **Select the target floor** using directional scanning — requests in the current direction of travel are served first (up-first if idle), falling back to the opposite direction if none are pending.
5. **Set direction and speed.** Direction is derived from target vs. current floor; speed switches between a slow single-floor run and a faster multi-floor run based on distance.
6. **Enforce door interlocks** — obstruction sensor and manual open always reopen the door and reset the auto-close timer; manual close is only accepted when the doorway is clear; open and close can never be commanded at the same time.
7. **Gate motor start** on target validity, door state, and floor range — movement is only allowed when every interlock is satisfied.
8. **Simulate movement** with a 5-second-per-floor timer, incrementing or decrementing the current floor once the timer completes.
9. **Update floor sensors** directly from the simulated position.
10. **On arrival**, clear the served request and corresponding buttons, stop the motor, open the door, and restart the door timer.
11. **Auto-close the door** after a timeout, but only if the doorway is clear.

[*SCL Code*] (https://github.com/chetanhadagal/plc-elevator-control-scl/blob/main/Elevator_Control_SCL.scl)

## The SCL program

Single scan-cycle program (`Elevator_Control_SCL.scl`) organised into clearly commented sections:

- **Initialisation** — first-scan reset of floor, requests, and outputs
- **E-Stop & fault handling** — fault latch with manual reset, early `RETURN` on active fault
- **Store floor requests** — hall call + cabin button flags per floor
- **Target floor selection** — directional scan logic (services requests ahead of the current direction first)
- **Direction & speed selection** — sets `Motor_DIRECTION_UP` / `_DOWN` and slow/fast speed
- **Door interlock logic** — obstruction sensor, manual open/close, mutual exclusion
- **Motor start condition** — combines all interlocks into a single start permissive
- **Movement simulation** — 5 s/floor timer updates simulated position
- **Floor sensor status** — derived from simulated position
- **Arrival & request clearing** — clears requests/buttons, stops lift, opens door
- **Door auto-close timer** — closes automatically after timeout if clear

## Tech

- **Siemens TIA Portal** (v17/v18 — set to your version)
- **Structured Control Language (SCL)** for the control logic
- **WinCC Unified** for the HMI (SIMATIC styling, animated shaft view)
- Runs in **Simulation (PLCSIM / PLCSIM Advanced)**, no PLC hardware needed

## Run it

1. Open the project in TIA Portal.
2. Compile the PLC program (right-click → Compile).
3. Start **PLCSIM** and download the program to the simulated PLC.
4. Set the PLC to `RUN` mode.
5. Open the WinCC Unified runtime and use the hall call buttons / cabin buttons to call the elevator.

## What I learned

- **Directional scanning beats naive FCFS.** Checking requests ahead of the current direction first, then falling back to the opposite direction, avoids needless direction reversals.
- **Fault handling needs an early exit.** Using `RETURN` after latching a fault keeps the rest of the scan from fighting the safety shutdown.
- **Door logic has more edge cases than expected.** Obstruction, manual open, manual close, and mutual exclusion between open/close all had to be resolved in a strict order to avoid contradictory commands in the same scan.
- **Simulating time in SCL.** Using `TON` timers for both floor-to-floor travel and door dwell time made the system behave predictably without any real hardware.
- **Request clearing must be tied to arrival, not to the button press.** Clearing on arrival (via floor sensors) rather than on request avoids losing a request if the lift was already busy.

## Next steps

- Add priority/VIP or fire-service override requests
- Support more than 4 floors without rewriting the scan logic (array-based requests/sensors)
- Add door-closed feedback sensor instead of resetting `Door_CLOSE` after one scan
- Multi-car dispatch logic
- Integrate real hardware I/O
- Port to CODESYS for comparison

## Author

**Chetan Hadagal** — Robotics & Automation Engineer
[LinkedIn](www.linkedin.com/in/chetanhadagal)
