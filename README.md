This is a VHDL-based RTL design of a smart traffic signal controller using a Finite State Machine (FSM).
*  What the system actually does (clear explanation)
 Vehicle-based control
veh_ns → vehicle detected on North-South road
veh_ew → vehicle detected on East-West road

Signals change based on traffic presence, not just fixed timing.

Pedestrian crossing
ped_pulse → button press
Internally stored as ped_req_latched

 Once pressed:
System guarantees a walk phase
Even if signal is busy → request is not lost
 Emergency handling
emergency = 1

 Immediate action:

All signals → RED
Traffic stops instantly
 Night mode
night_mode = 1

 Changes behavior:

No normal cycle
Lights blink (yellow/red) instead.
The system moves through states like this:

NS Green → NS Yellow → All Red →
EW Green → EW Yellow → All Red →
(Pedestrian if requested) →
Repeat
* Timing mechanism (important interview point)

Instead of delays, it uses:

tick → slow timing pulse
Internal counter → timer_count
timer_limit → defines duration

 Example:

Green = 5 sec
Yellow = 2 sec
All Red = 1 sec

This makes it:
✔ Synthesizable
✔ Hardware-accurate
* Main blocks in your code
1. Pedestrian Latch

Stores button press:

if ped_pulse = '1' → ped_req_latched = '1'

 Ensures:

No missed pedestrian request
2. Timer Block

Counts ticks:

Starts when timer_start = 1
Ends when timer_done = 1

 Controls duration of each signal

3. State Register
state <= next_state;

 Handles:

Normal transitions
Emergency override
Night mode override
4. Next-State Logic (Brain of system)

This decides:

Which state comes next
When to change signals
 6. What outputs it generates

These are digital signals (1-bit each) — meant to drive LEDs or GPIO pins.

Traffic Lights (Two Roads)
North-South Road
ns_g → Green light
ns_y → Yellow light
ns_r → Red light
East-West Road
ew_g → Green
ew_y → Yellow
ew_r → Red
 Pedestrian Signals
ped_walk → WALK signal
ped_dontwalk → STOP signal
7. Example Output Behavior
Normal case:
NS: Green
EW: Red
Ped: Don’t Walk

Then:

NS: Yellow
EW: Red

Then:

NS: Red
EW: Green
Pedestrian request:
All Red
Ped: WALK
Emergency:
NS: Red
EW: Red
Ped: Don’t Walk
Night mode:
Blinking Yellow (main road)
Blinking Red (side road)
8. What makes this design strong (say this!)

 In interview, highlight this:

Uses FSM for deterministic control
Implements real-time timing using tick-based counter
Handles asynchronous events (pedestrian, emergency)
Includes mode-based operation (normal + night)
Designed at RTL level → synthesizable on FPGA
