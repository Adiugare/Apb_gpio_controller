# APB GPIO Controller

## 1. Project Overview

The APB GPIO Controller is a parameterized RTL design that provides software-controlled General Purpose Input/Output (GPIO) functionality through an AMBA APB interface.

The controller supports GPIO data read/write, configurable GPIO direction, SET/CLR/TOGGLE operations, input synchronization, and interrupt generation based on GPIO input activity.

The design is implemented in Verilog and verified using a self-checking testbench with directed functional test cases.

## 2. Features

* AMBA APB-compatible slave interface
* Parameterized GPIO width
* GPIO input and output support
* Configurable GPIO direction
* GPIO DATA register
* GPIO SET operation
* GPIO CLR operation
* GPIO TOGGLE operation
* Two-flip-flop GPIO input synchronizer
* Level-sensitive interrupt detection
* Edge-sensitive interrupt detection
* Interrupt enable control
* Interrupt status register
* Software interrupt status clearing
* APB address validation
* APB slave error generation
* Zero-wait-state APB response
* Active-low asynchronous reset
* Self-checking Verilog testbench

## 3. Block Diagram

```text
                    +-----------------------+
                    |       APB Master       |
                    +-----------+-----------+
                                |
                                | APB
                                v
                    +-----------------------+
                    |    APB Interface &    |
                    |   Address Decoder     |
                    +-----------+-----------+
                                |
              +-----------------+------------------+
              |                 |                  |
              v                 v                  v
       +-------------+   +-------------+   +-------------+
       | GPIO Data   |   | GPIO DIR    |   | SET/CLR/    |
       | Register    |   | Register    |   | TOGGLE      |
       +------+------+   +------+------+   +-------------+
              |                 |
              |                 |
              v                 v
         gpio_out           gpio_oe


 gpio_in
    |
    v
+-------------------+
| 2-FF Synchronizer |
+---------+---------+
          |
          v
+-------------------+
| Edge / Level      |
| Detection        |
+---------+---------+
          |
          v
+-------------------+
| Interrupt Status  |
+---------+---------+
          |
          v
+-------------------+
| Interrupt Enable  |
+---------+---------+
          |
          v
       gpio_irq
```

## 4. Directory Structure

```text
APB_GPIO_Controller/
│
├── rtl/
│   └── apb_gpio.v
│
├── tb/
│   └── apb_gpio_tb.v
│
├── sim/
│   └── simulation_files
│
└── README.md
```

## 5. Module Information

### Top Module

```text
apb_gpio
```

### Parameters

| Parameter  | Default | Description         |
| ---------- | ------: | ------------------- |
| GPIO_WIDTH |      32 | Number of GPIO pins |
| ADDR_WIDTH |       8 | APB address width   |
| DATA_WIDTH |      32 | APB data width      |

## 6. Interface

### APB Signals

| Signal   | Direction |      Width | Description                   |
| -------- | --------- | ---------: | ----------------------------- |
| PCLK     | Input     |          1 | APB clock                     |
| PRESET_n | Input     |          1 | Active-low asynchronous reset |
| PSEL     | Input     |          1 | Peripheral select             |
| PENABLE  | Input     |          1 | APB enable                    |
| PWRITE   | Input     |          1 | Read/write control            |
| PADDR    | Input     |          8 | APB address                   |
| PWDATA   | Input     |         32 | APB write data                |
| PRDATA   | Output    |         32 | APB read data                 |
| PREADY   | Output    |          1 | APB ready response            |
| PSLVERR  | Output    |          1 | APB slave error               |
| gpio_in  | Input     | GPIO_WIDTH | GPIO input pins               |
| gpio_out | Output    | GPIO_WIDTH | GPIO output data              |
| gpio_oe  | Output    | GPIO_WIDTH | GPIO output enable            |
| gpio_irq | Output    |          1 | GPIO interrupt                |

## 7. Register Map

| Address | Register   | Access | Description               |
| ------- | ---------- | ------ | ------------------------- |
| 0x00    | DATA       | R/W    | GPIO data register        |
| 0x04    | DIR        | R/W    | GPIO direction register   |
| 0x08    | SET        | W      | Set selected GPIO bits    |
| 0x0C    | CLR        | W      | Clear selected GPIO bits  |
| 0x10    | TOGGLE     | W      | Toggle selected GPIO bits |
| 0x14    | INT_EN     | R/W    | Interrupt enable          |
| 0x18    | INT_STATUS | R/W    | Interrupt status          |
| 0x1C    | INT_TYPE   | R/W    | Interrupt type            |

All supported registers are 32-bit wide.

## 8. Register Description

### DATA Register — 0x00

Stores the GPIO output data.

For every bit:

```text
gpio_out = gpio_data
```

Writing a value directly updates the GPIO output data.

Example:

```text
Write 0x000000FF

gpio_out = 0x000000FF
```

### DIR Register — 0x04

Controls the direction of each GPIO pin.

```text
0 = Input
1 = Output
```

Example:

```text
DIR = 0x0000000F
```

GPIO[3:0] are configured as outputs and GPIO[31:4] as inputs.

The register is also reflected on:

```text
gpio_oe = gpio_dir
```

### SET Register — 0x08

Sets selected GPIO data bits to 1.

Operation:

```text
gpio_data = gpio_data | PWDATA
```

Example:

```text
Current DATA = 0x00000000
SET          = 0x0000000F

New DATA     = 0x0000000F
```

### CLR Register — 0x0C

Clears selected GPIO data bits.

Operation:

```text
gpio_data = gpio_data & ~PWDATA
```

Example:

```text
Current DATA = 0x000000FF
CLR          = 0x0000000F

New DATA     = 0x000000F0
```

### TOGGLE Register — 0x10

Toggles selected GPIO data bits.

Operation:

```text
gpio_data = gpio_data ^ PWDATA
```

Example:

```text
Current DATA = 0x000000FF
TOGGLE       = 0x0000000F

New DATA     = 0x000000F0
```

### INT_EN Register — 0x14

Enables or disables interrupts for individual GPIO inputs.

```text
0 = Interrupt disabled
1 = Interrupt enabled
```

The final interrupt output is generated using:

```text
gpio_irq = |(gpio_int_status & gpio_int_en)
```

### INT_STATUS Register — 0x18

Stores the interrupt status for each GPIO input.

A status bit is set when the corresponding GPIO input satisfies the configured interrupt condition.

Writing a `1` to a status bit clears that interrupt status bit.

### INT_TYPE Register — 0x1C

Selects the interrupt detection mode.

```text
0 = Level-sensitive interrupt
1 = Edge-sensitive interrupt
```

The current RTL detects any input transition for edge mode:

```text
gpio_edge = gpio_in_sync1 ^ gpio_in_sync2
```

Therefore, both rising and falling transitions can generate an edge event.

## 9. GPIO Input Synchronization

GPIO inputs may not be synchronized with the APB clock domain.

To reduce metastability risk, the design uses a two-flip-flop synchronizer.

```text
gpio_in
   |
   v
+--------+
| sync1  |
+--------+
   |
   v
+--------+
| sync2  |
+--------+
   |
   v
Synchronized GPIO
```

The synchronized signal is used for interrupt detection.

## 10. Interrupt Operation

The controller supports two interrupt modes.

### Level Mode

When:

```text
INT_TYPE = 0
```

the synchronized GPIO input level is used as the interrupt condition.

```text
int_set_bits = ~gpio_int_type & gpio_level
```

If the GPIO input is high, the corresponding interrupt status can be set.

### Edge Mode

When:

```text
INT_TYPE = 1
```

the design detects changes between synchronized GPIO samples.

```text
gpio_edge = gpio_in_sync1 ^ gpio_in_sync2
```

A transition generates an interrupt event.

### Interrupt Masking

The interrupt enable register controls whether an active interrupt status contributes to `gpio_irq`.

```text
gpio_irq = |(gpio_int_status & gpio_int_en)
```

Therefore:

```text
INT_STATUS = 1
INT_EN     = 1
```

results in:

```text
gpio_irq = 1
```

If the interrupt is disabled:

```text
INT_EN = 0
```

the corresponding status may remain set, but it does not assert `gpio_irq`.

## 11. APB Operation

The design uses the standard APB setup and access phases.

### Write Transaction

```text
Cycle 1:
PSEL    = 1
PENABLE = 0
PWRITE  = 1
PADDR   = Register Address
PWDATA  = Write Data

Cycle 2:
PSEL    = 1
PENABLE = 1
PWRITE  = 1
```

The write operation occurs during the APB access phase.

### Read Transaction

```text
Cycle 1:
PSEL    = 1
PENABLE = 0
PWRITE  = 0
PADDR   = Register Address

Cycle 2:
PSEL    = 1
PENABLE = 1
PWRITE  = 0
```

The corresponding register value is returned through `PRDATA`.

### PREADY

The controller provides zero-wait-state APB operation:

```text
PREADY = 1'b1
```

### PSLVERR

`PSLVERR` is asserted when an APB access uses an invalid or unaligned address.

```text
PSLVERR = access_phase && !addr_valid
```

## 12. Reset Behavior

The controller uses an active-low asynchronous reset.

When:

```text
PRESET_n = 0
```

the following registers are cleared:

```text
gpio_data
gpio_dir
gpio_int_en
gpio_int_status
gpio_int_type
gpio_in_sync1
gpio_in_sync2
```

After reset:

```text
GPIO output data = 0
GPIO direction   = Input
Interrupt enable = Disabled
Interrupt status = 0
```

## 13. Verification

The design is verified using a self-checking Verilog testbench.

The testbench performs APB read and write transactions and automatically compares the actual DUT outputs against expected values.

The testbench maintains:

```text
pass_count
fail_count
```

At the end of simulation, the testbench reports whether the verification completed successfully.

## 14. Test Cases

| Test | Description                         |
| ---- | ----------------------------------- |
| T1   | Reset verification                  |
| T2   | DATA register read/write            |
| T3   | DIR register configuration          |
| T4   | SET, CLR and TOGGLE operations      |
| T5   | Interrupt enable and interrupt type |
| T6   | Level interrupt generation          |
| T7   | Edge interrupt generation           |
| T8   | Interrupt status clearing           |
| T9   | Interrupt masking                   |
| T10  | Invalid APB address                 |
| T11  | Unaligned APB address               |
| T12  | Valid APB address                   |

## 15. Expected Verification

The testbench checks:

* Register read/write functionality
* GPIO output operation
* GPIO direction configuration
* SET operation
* CLR operation
* TOGGLE operation
* Level interrupt generation
* Edge interrupt generation
* Interrupt status clearing
* Interrupt masking
* APB error response
* Valid APB access response

## 16. Simulation

The RTL can be simulated using standard Verilog/SystemVerilog simulators such as:

* QuestaSim
* Synopsys VCS
* Xilinx Vivado Simulator

Example QuestaSim flow:

```text
vlog rtl/apb_gpio.v
vlog tb/apb_gpio_tb.v
vsim apb_gpio_tb
run -all
```

The exact command may vary depending on the simulator setup and project directory.

## 17. Expected Result

A successful simulation should show PASS messages for the functional test cases and:

```text
FAIL count = 0
```

at the end of the simulation.

## 18. Design Highlights

The main design concepts demonstrated by this project are:

```text
APB Protocol
Memory-Mapped Registers
Parameterized RTL
GPIO Control
Register-Based Hardware Control
2-FF Synchronization
Edge Detection
Level Detection
Interrupt Generation
Interrupt Masking
APB Error Handling
Self-Checking Verification
```

## 19. Limitations and Possible Improvements

The current implementation can be extended with:

* APB4 PSTRB support
* Separate rising-edge and falling-edge interrupt selection
* Interrupt polarity configuration
* Programmable interrupt debounce
* SystemVerilog Assertions
* Functional coverage
* Constrained-random verification
* UVM-based verification environment
* Formal verification
* Additional lint and CDC checks

## 20. Conclusion

The APB GPIO Controller provides a simple and configurable interface for controlling and monitoring GPIO pins through an APB bus.

The design demonstrates important RTL concepts including memory-mapped register design, APB protocol handling, GPIO direction and data control, input synchronization, interrupt generation, and self-checking verification.

The project provides a good practical example of integrating a peripheral IP block with an AMBA APB-based system.
