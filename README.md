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

\`\`\`scl
// -------------- INITIALISATION --------------
// Run only once during first PLC scan 

IF NOT #Initialised THEN
    #Current_Floor := 1;       // Lift initially parked at Floor 1
    #Target_Floor := 0;
    #Motor_START := FALSE;
    #Motor_DIRECTION_UP := FALSE;
    #Motor_DIRECTION_DOWN := FALSE;
    #Door_OPEN := FALSE;
    #Door_CLOSE := FALSE;
    #Motor_SPEED := 0;
    
    // Reset all stored floor requests
    #Request_Floor1 := FALSE;
    #Request_Floor2 := FALSE;
    #Request_Floor3 := FALSE;
    #Request_Floor4 := FALSE;
    
    // Reset all cabin and hall call buttons
    #Cabin_Button_Floor1 := FALSE;
    #Cabin_Button_Floor2 := FALSE;
    #Cabin_Button_Floor3 := FALSE;
    #Cabin_Button_Floor4 := FALSE;
    
    #Floor1_UP := FALSE;
    #Floor2_UP := FALSE;
    #Floor2_DOWN := FALSE;
    #Floor3_UP := FALSE;
    #Floor3_DOWN := FALSE;
    #Floor4_DOWN := FALSE;
    
    #Initialised := TRUE;
END_IF;

// -------------- E-STOP & FAULT HANDLING --------------
// Latch fault if any safety condition becomes active
// 
IF #EStop_Button OR #Motor_Overload OR #Drive_Fault THEN
    #Fault_Latch := TRUE;
END_IF;

// Fault reset allowed only when all fault conditions are cleared
// 
IF #Fault_Reset AND NOT #EStop_Button AND NOT #Motor_Overload AND NOT #Drive_Fault THEN
    #Fault_Latch := FALSE;
END_IF;

// If fault is active, stop all movement immediately
IF #Fault_Latch THEN
    #Motor_START := FALSE;
    #Motor_DIRECTION_UP := FALSE;
    #Motor_DIRECTION_DOWN := FALSE;
    #Motor_SPEED := 0;
    RETURN;   // Skip remaining logic
END_IF;


// -------------- STORE FLOOR REQUESTS --------------
// Store Floor request unless lift is already idle at that Floor
//
IF (#Floor1_UP OR #Cabin_Button_Floor1) AND NOT (#Current_Floor = 1 AND #Target_Floor = 0) THEN
    #Request_Floor1 := TRUE;
END_IF;

IF (#Floor2_UP OR #Floor2_DOWN OR #Cabin_Button_Floor2) AND NOT (#Current_Floor = 2 AND #Target_Floor = 0) THEN
    #Request_Floor2 := TRUE;
END_IF;

IF (#Floor3_UP OR #Floor3_DOWN OR #Cabin_Button_Floor3) AND NOT (#Current_Floor = 3 AND #Target_Floor = 0) THEN
    #Request_Floor3 := TRUE;
END_IF;

IF (#Floor4_DOWN OR #Cabin_Button_Floor4) AND NOT (#Current_Floor = 4 AND #Target_Floor = 0) THEN
    #Request_Floor4 := TRUE;
END_IF;

// -------------- TARGET FLOOR SELECTION --------------
// Priority given to requests in current travelling direction

IF #Target_Floor = 0 THEN
    
    // --- UP direction ---
    IF #Motor_DIRECTION_UP OR (NOT #Motor_DIRECTION_UP AND NOT #Motor_DIRECTION_DOWN) THEN
        
        // Check requests above current floor first
        IF #Request_Floor2 AND #Current_Floor <= 2 THEN
            #Target_Floor := 2;
        ELSIF #Request_Floor3 AND #Current_Floor <= 3 THEN
            #Target_Floor := 3;
        ELSIF #Request_Floor4 AND #Current_Floor <= 4 THEN
            #Target_Floor := 4;
            
        // If no request above, scan downward
        ELSIF #Request_Floor3 THEN
            #Target_Floor := 3;
        ELSIF #Request_Floor2 THEN
            #Target_Floor := 2;
        ELSIF #Request_Floor1 THEN
            #Target_Floor := 1;
        END_IF;
        
        // DOWN direction scan
    ELSIF #Motor_DIRECTION_DOWN THEN
        
        // Check requests below current floor first
        IF #Request_Floor3 AND #Current_Floor >= 3 THEN
            #Target_Floor := 3;
        ELSIF #Request_Floor2 AND #Current_Floor >= 2 THEN
            #Target_Floor := 2;
        ELSIF #Request_Floor1 AND #Current_Floor >= 1 THEN
            #Target_Floor := 1;
            
        // If no request below, scan upward
        ELSIF #Request_Floor2 THEN
            #Target_Floor := 2;
        ELSIF #Request_Floor3 THEN
            #Target_Floor := 3;
        ELSIF #Request_Floor4 THEN
            #Target_Floor := 4;
        END_IF;
        
    END_IF;
    
END_IF;


// -------------- DIRECTION SELECTION --------------

#Motor_DIRECTION_UP := FALSE;
#Motor_DIRECTION_DOWN := FALSE;

// Decide lift movement direction based on target floor
IF #Target_Floor > #Current_Floor THEN
    #Motor_DIRECTION_UP := TRUE;
ELSIF #Target_Floor < #Current_Floor AND #Target_Floor > 0 THEN
    #Motor_DIRECTION_DOWN := TRUE;
END_IF;


// -------------- SPEED SELECTION --------------
// Select motor speed based on travel distance
// 
IF #Target_Floor > 0 THEN
    IF ABS(#Target_Floor - #Current_Floor) = 1 THEN
        #Motor_SPEED := 500;    // Slow speed: Single-floor run 
    ELSE
        #Motor_SPEED := 1500;   // High speed: Multi-floor run
    END_IF;
ELSE
    #Motor_SPEED := 0;
END_IF;


// -------------- DOOR INTERLOCK LOGIC --------------
// No movement while door is open

// Reopen door immediately if obstruction detected
IF #Door_Safety_Sensor THEN
    #Door_OPEN := TRUE;
    #Door_CLOSE := FALSE;
    #Timer_Door.TON (IN := FALSE,  // Reset door timer
    PT := T#5s);  
END_IF;

// Manual door open command
IF #Cabin_Button_Door_OPEN THEN
    #Door_OPEN := TRUE;
    #Door_CLOSE := FALSE;
    #Timer_Door.TON (IN := FALSE, // Reset auto-close timer
    PT := T#5s);  
END_IF;

// Manual door close button — only allow if safety sensor is clear
IF #Cabin_Button_Door_CLOSE AND NOT #Door_Safety_Sensor THEN
    #Door_OPEN := FALSE;
    #Door_CLOSE := TRUE;
END_IF;

// Prevent simultaneous Door_OPEN and Door_CLOSE command
IF #Door_OPEN AND #Door_CLOSE THEN
    #Door_CLOSE := FALSE;
END_IF;


// -------------- MOTOR START CONDITION --------------
// Door interlock applied here

// Default motor command OFF
#Motor_START := FALSE;

// Allow movement only when all interlocks are healthy
IF (#Target_Floor <> 0)
    AND (#Target_Floor <> #Current_Floor)
    AND NOT #Door_OPEN
    AND NOT #Door_CLOSE       
    AND (#Current_Floor >= 1) AND (#Current_Floor <= 4) 
THEN
    #Motor_START := TRUE;
END_IF;


// -------------- MOVEMENT SIMULATION --------------
// Simulated travel time = 5 seconds per floor

// Timer runs only while motor is active 
#Timer_Move.TON(IN  := #Motor_START AND NOT #Move_Pulse,
                PT  := T#5s);
// Capture timer completion pulse
#Move_Pulse := #Timer_Move.Q;  

// Update current floor after timer completion
IF #Move_Pulse THEN
    
    // Simulate upward movement
    IF #Motor_DIRECTION_UP AND #Current_Floor < 4 THEN
        #Current_Floor := #Current_Floor + 1;
        
    // Simulate downward movement
    ELSIF #Motor_DIRECTION_DOWN AND #Current_Floor > 1 THEN
        #Current_Floor := #Current_Floor - 1;
    END_IF;
    
    // Reset pulse for next movement cycle
    #Move_Pulse := FALSE; 
END_IF;


// -------------- FLOOR SENSOR STATUS --------------
// Floor sensors derived from simulated lift position

#Floor1_Sensor := (#Current_Floor = 1);
#Floor2_Sensor := (#Current_Floor = 2);
#Floor3_Sensor := (#Current_Floor = 3);
#Floor4_Sensor := (#Current_Floor = 4);


// -------------- ARRIVAL & REQUEST CLEARING --------------
// 
IF (#Target_Floor > 0) AND (#Current_Floor = #Target_Floor) THEN
    
    // Clear active request for arrived floor
    IF #Floor1_Sensor THEN
        #Request_Floor1 := FALSE;
    END_IF;
    IF #Floor2_Sensor THEN
        #Request_Floor2 := FALSE;
    END_IF;
    IF #Floor3_Sensor THEN
        #Request_Floor3 := FALSE;
    END_IF;
    IF #Floor4_Sensor THEN
        #Request_Floor4 := FALSE;
    END_IF;
    
    // Reset cabin and hall buttons for current floor
    CASE #Current_Floor OF
        1:
            #Cabin_Button_Floor1 := FALSE;
            #Floor1_UP := FALSE;
        2:
            #Cabin_Button_Floor2 := FALSE;
            #Floor2_UP := FALSE;
            #Floor2_DOWN := FALSE;
        3:
            #Cabin_Button_Floor3 := FALSE;
            #Floor3_UP := FALSE;
            #Floor3_DOWN := FALSE;
        4:
            #Cabin_Button_Floor4 := FALSE;
            #Floor4_DOWN := FALSE;
    END_CASE;
    
    // Stop lift and open door on arrival
    #Target_Floor := 0;
    #Motor_START := FALSE;
    #Door_OPEN := TRUE;
    #Door_CLOSE := FALSE;
    
    // Restart door timer from zero
    #Timer_Door.TON (IN := FALSE,
    PT := T#5s);
    
END_IF;


// -------------- DOOR AUTO-CLOSE TIMER --------------

// Run timer while door remains open
#Timer_Door.TON(IN := #Door_OPEN,
                PT := T#5s);

// Auto-close door after timeout if doorway is clear
IF #Timer_Door.Q AND NOT #Door_Safety_Sensor THEN
    #Door_OPEN := FALSE;
    #Door_CLOSE := TRUE;
END_IF;

// Reset close command after one PLC scan
// Real system should use door-closed feedback sensor
IF #Door_CLOSE AND NOT #Door_OPEN THEN
    #Door_CLOSE := FALSE;  
END_IF;
\`\`\`

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
