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

## HMI

Built in **WinCC Unified** using standard SIMATIC colour conventions:

- Graphical shaft view with animated cabin movement
- Door animation
- Hall call buttons
- Full drive status panel
- Live alarm log

Gives the operator a complete picture of system state at all times.

## Design notes

This project emphasises **modularity and scalability**, making it straightforward to extend for additional floors or integrate new features. The SCL code is organised into clearly commented sections, making use of timers, conditional statements, boolean latching logic, edge detection, and LOOK-algorithm queuing to simulate realistic industrial control behaviour.

## Next steps

Feel free to use this project as a learning resource or a starting point for your own work:

- Add priority request handling
- Integrate real hardware I/O
- Simulate more complex elevator systems (multi-car dispatch, etc.)
- Explore other industrial automation applications