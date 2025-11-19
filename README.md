# DIGITAL-TACHOMETER

## Introduction
A digital tachometer is an instrument that measures the
rotational speed (RPM) of an object, most commonly shafts or
motors in vehicles and industrial equipment. Accurate RPM
measurement is crucial for diagnosing machine health, optimizing performance, and ensuring operational safety in
automotive and factory environments. Digital tachometers use
sensors to detect rotational motion and then process these
signals electronically for display.

## Real-World Applications
# Automotive:
Used to monitor engine RPM, enabling drivers and technicians
to maintain engines within optimal operating ranges, thus
preserving fuel efficiency and engine health. 
# Industrial Sector:
Essential for monitoring the operation of motors, pumps, conveyors, and fans, allowing preventative maintenance and
reducing machine downtime.

# Project Objective:
The goal is to design a digital tachometer capable of accurately
calculating the RPM of a motor and displaying it in real time. The design is modular, leveraging core concepts of Verilog
HDL for digital logic design. 
# Functional Modules
# 1. Sensor Simulation
-- Periodic pulses are generated to simulate the real-
world pulses produced by a magnetic, optical, or Hall-effect
RPM sensor. 
-- Each pulse represents one shaft rotation or a specified
angular increment. 
# 2. Counter Module
-- Counts the number of input pulses over a precisely
timed interval (typically 1 second). 
-- Implemented using Verilog counters and timing
blocks to ensure accurate pulse accumulation. -- The counter resets at the end of each measurement
period. 
# 3. Computation Unit
-- At the end of the counting window, the pulse count is
converted to the RPM value. -- For a 1-second window, RPM = pulse count × 60. -- Arithmetic is performed using Verilog operations
within always blocks sensitive to the timing signal. -- Handles cases where the sensor produces more than
one pulse per rotation (scaling factor applied if required). 4. Display Module
-- Shows the calculated RPM on a digital readout (7- segment display or LCD, depending on hardware). -- Reads the computed RPM and drives the
corresponding output lines using Verilog logic.

# BLOCK DIAGRAM:

<img width="692" height="330" alt="image" src="https://github.com/user-attachments/assets/9ba33cb8-e555-43c8-a034-955ff91f6234" />

# VERILOG HDL CODE
```
module gate_generator (
input wire clk, input wire reset, output reg gate_enable, output reg measurement_done
);
// 50,000,000 cycles for 1 second
// 5,000,000 cycles for 0.1 second (100 ms)
parameter GATE_COUNT_MAX = 24'd4999999;
// A 23-bit counter is needed since 2^23 = 8,388,608
reg [22:0] gate_counter;
always @(posedge clk) begin
if (reset) begin
gate_counter <= 0;
gate_enable <= 1'b0;
measurement_done <= 1'b0;
end else begin
measurement_done <= 1'b0; // Default to low
if (gate_counter == GATE_COUNT_MAX) begin
// End of the 100ms window
gate_counter <= 0;
gate_enable <= 1'b0; // Disable the counter now
measurement_done <= 1'b1; // Pulse for one clock
cycle
end else begin
// Counting cycle
gate_counter <= gate_counter + 1;
// Keep the gate open during the counting phase
if (gate_counter == 0) begin
// Start the gate only when the counter resets to 0
gate_enable <= 1'b1;
end
end
end
end
endmodule
module pulse_counter (
input wire clk, input wire reset, input wire sensor_pulse, input wire gate_enable, // Add the missing input signal here:
input wire measurement_done, output reg [15:0] pulse_count
);
// ... (Existing Synchronization Code) ... reg sensor_sync_0, sensor_sync_1;
wire pulse_rising_edge;
always @(posedge clk) begin
sensor_sync_0 <= sensor_pulse;
sensor_sync_1 <= sensor_sync_0;
end
assign pulse_rising_edge = (sensor_sync_1 == 1'b0) &&
(sensor_sync_0 == 1'b1);
// --- Corrected Counter Logic --- always @(posedge clk) begin
if (reset) begin
pulse_count <= 0;
end else if (gate_enable) begin
// 1. Counting Phase: Gate is open (100ms)
if (pulse_rising_edge) begi
pulse_count <= pulse_count + 1;
end
end else begin
// 2. Latch/Reset Phase: Gate is closed
if (measurement_done) begin
// Reset counter ONLY when the computation unit has
read the value
pulse_count <= 0;
end
// Note: If measurement_done is low and gate_enable is
low, the count is held. end
end
endmodule
module rpm_calculator (
input wire clk, input wire reset, input wire measurement_done, input wire [15:0] pulse_count_in, output reg [23:0] rpm_value_out
);
// Multiplier derived from formula: RPM = pulse_count * 150
parameter RPM_MULTIPLIER = 8'd150;
// The output RPM value needs 24 bits: Max RPM * 150 =
400 * 150 = 60,000. // 2^16 = 65,536, so 16 bits is enough. We use 24 bits just to
be safe
// for a larger pulse_count_in (16 bits) * 8 bits. // Use a non-pipelined, combinatorial multiplication for
simplicity. // In a high-speed design, this would be registered. wire [2:0] calculated_rpm;
assign calculated_rpm = pulse_count_in *
RPM_MULTIPLIER;
always @(posedge clk) begin
if (reset) begin
rpm_value_out <= 0;
end else if (measurement_done) begin
// Latch the calculated value when the measurement is
complete
rpm_value_out <= calculated_rpm;
end
end
endmodule
`timescale 1ns / 1ps
module tachometer_tb;
// --- TB Signals ---
reg clk;
reg reset;
reg sensor_pulse_gen;
// --- Module Wires --- wire gate_enable;
wire measurement_done;
wire [15:0] pulse_count;
wire [23:0] rpm_value;
// --- System Parameters --- parameter CLK_PERIOD = 20; // 50 MHz clock (20 ns
period)
// 5 ms period for 200 Hz -> 2.5 ms high, 2.5 ms low
parameter PULSE_HALF_PERIOD = 5_000_000; // 2.5 ms
(2,500,000 ns)
// --- Instantiate Modules ---
gate_generator UUT_GATE (
.clk(clk),
.reset(reset),
.gate_enable(gate_enable),
.measurement_done(measurement_done)
);
pulse_counter UUT_COUNTER (
.clk(clk),
.reset(reset),
.sensor_pulse(sensor_pulse_gen),
.gate_enable(gate_enable),
.pulse_count(pulse_count)
);
rpm_calculator UUT_CALC (
.clk(clk),
.reset(reset),
.measurement_done(measurement_done),
.pulse_count_in(pulse_count),
.rpm_value_out(rpm_value)
);
// --- Clock Generation ---
initial begin
clk = 1'b0;
forever #(CLK_PERIOD / 2) clk = ~clk;
end
// --- Sensor Pulse Generation (Simulates 3000 RPM) ---
initial begin
sensor_pulse_gen = 1'b0;
// Generate a 200 Hz square wave (5ms period)
forever #(PULSE_HALF_PERIOD) sensor_pulse_gen =
~sensor_pulse_gen;
End
// --- Test Sequence ---
initial begin
$dumpfile("tachometer.vcd");
$dumpvars(0, tachometer_tb);
// 1. Initial Reset
reset = 1'b1;
#100;
reset = 1'b0; // Release Reset
$display("--- Starting Tachometer Simulation (3000 RPM
Target) ---");
$display("System Clock: 50 MHz, Gate Time: 100 ms, PPR: 4");
$display("Expected Pulse Count in 100ms: 200 Hz * 0.1 s = 20 pulses.");
$display("Expected Final RPM: 3000.");
$display("---------------------------------------------------------");
// 2. Run for multiple measurement cycles (100ms per cycle)
// Wait for 3 complete measurement cycles (approx 3 *
100ms)
#300_000_000;
// 3. Final Check and Finish
$display("Simulation Time: %0t ns. Final RPM
Value: %0d", $time, rpm_value);
if (rpm_value == 3000) begin
$display("--- PASS: RPM calculation is correct. ---");
end else begin
$display("--- FAIL: Calculated RPM (%0d) does not
match expected (3000). ---", rpm_value);
end
#100;
$finish;
end
endmodule
```
OUTPUT:

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/5c09ff21-eb54-4d4c-8d19-a5bc267029c4" />

<img width="1907" height="1074" alt="image" src="https://github.com/user-attachments/assets/8069f58f-7f86-45a5-900d-4821a7a2e9bc" />


<img width="688" height="387" alt="image" src="https://github.com/user-attachments/assets/e870e9a0-5550-4e58-94a2-77d8b9d268f1" />

<img width="692" height="389" alt="image" src="https://github.com/user-attachments/assets/8eeefb83-eb92-4a79-9568-a3be2d57407d" />

# Verilog Design Concepts

# Counters: 
Implemented with always blocks driven by either system clock or derived timing signals.

# Arithmetic Operations: 
Simple multipliers and accumulators in Verilog to convert counts to RPM.

# Timing Blocks: 
Use timers or clock dividers for precise measurement intervals.

# Modular Structure: 
Each block (sensor, counter, computation, display) is written as a separate Verilog module for clear hierarchy and testability.
`
# Synchronization: 
Proper clock domain crossing techniques if different clock domains are involved.

# Operation Workflow 
-- The sensor module produces periodic pulses representing motor rotation.
-- The counter accumulates pulses over a fixed interval set by a timing block.
-- When the interval concludes, the computation unit multiplies the pulse count as per the calculation formula to derive the RPM.
-- The computed RPM is latched and presented on the output display.
-- The process repeats at regular intervals for real-time monitoring.


# Performance & Benefits
-- Produces real-time, accurate RPM readings critical for performance tuning and fault detection in motors.
-- Modular Verilog implementation enables scalable design and ease of integration with other digital systems.
-- Fast update rate ensures timely alerts and precision in control applications.

# Conclusion

The digital tachometer project implements core Verilog concepts such as counters, arithmetic units, and timing logic to deliver reliable, real-time RPM measurements. Its applications span automotive diagnostics and industrial machinery maintenance, offering a robust and educational design suitable for engineering project work.


# Product References (Digital Tachometers)


### [SKF TKRT 31 Tachometer]()
#### industrial / precise
₹1,10,369.01

### [DT-2234C+ Laser Digital Tachometer]()
#### budget laser
₹1,537

### [Non‑contact Laser Tachometer]()
#### very cheap laser
₹975.28

### [RS PRO Digital Tachometer]()
#### professional
₹37,759

### [Trail Tech Vapor Computer Kit (with tach)]()
#### motorcycle computer
*₹14,362.37*Some highlights:

[UT373 Mini Non‑Contact Tachometer](): Compact and portable, suitable for basic RPM measurements.

[SKF TKRT 31 Tachometer](): More industrial-grade; accurate and durable.

[DT‑2234C+ Laser Digital Tachometer](): Uses laser for non-contact measurement; good for higher RPMs.

[Non‑contact Laser Tachometer](): Very cost-effective option for basic speed checking.

[RS PRO Digital Tachometer](): Reliable, with good build quality.


