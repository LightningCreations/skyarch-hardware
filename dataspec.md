
# Skyarch CPU Datasheet

## Pinout

Legend:

* Pull Down: Pin Connected Internally via 10kOhm resistor to GND provided both 5VCC and GND are connected
* Pull Up: Pin Connected Internally via 10kOhm Reistor to 5VCC provided both 5VCC and GND are connected
* Float: Pin floats when not connected
* Active: Pin is Strong Unknown (output)/Don't Care (input) when not in use
* Input: Data/Control Signals used by the CPU and supplied externally
* Output: Data/Status Signals set by the CPU
* Inout: Both Input and Output behaviour
* NC: Not Connected

| Pin(s) | Name             | Direction | Behaviour | Description                                |
|:------:|------------------|:---------:|:---------:|--------------------------------------------|
| 0      | 5VCC             | Input     | N/A       | Main System Supply/Logic Reference Voltage |
| 1      | BE               | Inout     | Pull Down | Bus Transfer Enable/Bus Occupied           |
| 2-21   | A0-19            | Output    | Float     | Address Bus (Bits 0-19)                    |
| 22     | A20/CX           | Output    | Float     | Address Bus (Bit 20)/Coprocessor Execute   |
| 23     | A21              | Output    | Float     | Address Bus (Bit 21)                       |
| 24-28  | A22-26/N0-4/W0-4 | Output    | Float     | Address Bus (Bits 22-28)/Coprocessor Number (Bits 0-4)/Transfer Width |
| 29-31  | A28-29           | Output    | Float     | Address Bus (Bits 29-31)                   |
| 32-63  | D0-31            | Inout     | Pull Down | Data Bus                                   |
| 64     | W                | Output    | Float     | Bus Read (0)/Write (1)                     |
| 65     | IO               | Output    | Float     | Memory Access (0)/IO Bus Access (1)        |
| 66     | SC               | Output    | Float     | Instruction Fetch(IO=0)/Coprocessor (IO=1) |
| 67     | BA               | Input     | Pull Down | Bus Acknowledge                            |
| 68     | CLK              | Input     | Puil Down | System Reference Clock                     |
| 69     | CCLK             | Output    | Active    | System Execution Clock                     |
| 70-71  | S0-S1            | Output    | Active    | System Status                              |
| 72-75  | CA0-3            | Input     | Pull Down | Coprocessor n available                    |
| 76-79  | CE0-3            | Output    | Active    | Coprocessor n enabled                      |
| 80-83  | CB0-3            | Input     | Float     | Coprocessor n busy                         |
| 84-87  | EX0-3            | Input     | Pull Down | Exception/NMI Trigger                      |
| 88-91  | IRQ0-3           | Input     | Pull Down | IRQ Trigger                                |
| 92     | RESET            | Input     | Pull Down | External Reset                             |
| 93     | IM               | Output    | Active    | Interrupts Masked                          |
| 94     | NC               | NC        | Float     | Not Connected                              |
| 95     | GND              | Input     | N/A       | Main System Ground/Logic Reference Ground  |
| 96-99  | CA4-7            | Input     | Pull Down | Coprocessor n available                    |
| 100-103| CE4-7            | Output    | Active    | Coprocessor n enabled                      |
| 104-107| CB4-7            | Input     | Float     | Coprocessor n busy                         |
| 108-111| WM0-3            | Output    | Pull Down | Write Mask (0-3)                           |
| 112-126| NC               | NC        | Float     | Not Connected                              |
| 127    | GND              | Input     | N/A       | Main System Ground/Logic Reference Ground  |

Pin 127 and 95 are internally connected together to form a common ground.

## Power Characteristics

The main logic current is supplied on pin 0. The pin requires 5V (TODO, specify effective range) relative to ground (pins 95 and 127).

The Logic reference ground is provided on pin 95 and ping 127. Only one pin is required to be connected to 0V. If both are connected, they must exhibit 0V relative to each other, as they are tied together internally. If only one pin is connected, the other can be used as a 0V Reference out.

## Power On Status

When the CPU becomes powered (5V on 5VCC, 0V on GND), the CPU is intiailized to an internally consistent state that exhibits at least the following properties:

* CE0-7 are 0
* BE is not powered
* S0-1 is `0b11`

After powering on the CPU in this state, the RESET pin must be brought high for a minimum of 2 clock cycles, before the CPU begins executing the Skyarch instruction set.

## Reset

A RESET is performed by bringing the RESET pin high. If the CPU already has S0-1=11, this must be held for a minimum of two cycles. If either status bit has any other value, it must be held until S0-1=11, and then for a minimum of two additional cycles. Once S0-1 have both become high, they will remain high as long as RESET is asserted. The CPU will not finish the RESET until the RESET pin is brought low.

The After a RESET (including a RESET from power on) the CPU is placed in the following architectural state:

* `IP=0`
* `ictl.m=0, ictl.a=0`
* `cpe=0`
* `cpa[0:7]=CA0-7, cpa[8:]=0`
* All other registers have undefined values. These values are electrically stable.

Additionally, the CPU is placed in a valid, neutral state, with the following properties:

* S0-1 is `0b00`
* CE0-7 are 0
* BE is not powered

A RESET also occurs if the processor executes an instruction that raises a priority 1 or priority 0 exception while `ictl.a=1`.

## Clock

The CPU Clock is controlled by the CLK pin. While the CPU is Active (S0-1=00), the CPU Mirrors the clock signal to the CCLK pin. Otherwise the CCLK pin is low.

## Bus Transfer

(Note: In Table below, BE=1 refers to when the CPU sets it. Other devices setting BE prevents the CPU from performming bus transfers until BE falls low)

| BE | W | IO | SC | CX | Address    | Width | Notes                |
|----|---|----|----|----|------------|-------|----------------------|
| 0  | x | x  | x  | x  | N/A        | N/A   | No transfer occuring |
| 1  | 0 | 0  | 0  | x  | A0-29*4    | 32    | Main Memory Read     |
| 1  | 1 | 0  | 0  | x  | A0-29*4    | 32    | Main Memory Write    |
| 1  | 0 | 0  | 1  | x  | A0-29*4    | 32    | Memory Read (Instruction Fetch) |
| 1  | 0 | 1  | 0  | x  | A0-19      | W0-4  | I/O Port Read |
| 1  | 1 | 1  | 0  | x  | A0-19      | W0-4  | I/O Port Write |
| 1  | 0 | 1  | 1  | 0  | N0-7:A0-4* | 32    | Coprocessor Register Read |
| 1  | 1 | 1  | 1  | 0  | N0-7:A0-5* | 32    | Coprocessor Register Write |
| 1  | 1 | 1  | 1  | 1  | N0-7       | 32    | Coprocessor Register Execute |

All unused bits of A0-29 are set to 0.

(x means Don't Care)

The lower Width bits of the Data bus contain the salient data, other bits are Connected to a Pull Down Resistor.

BE is an IO pin that controls whether or not the Bus is in use. When not set to 1 by the CPU, it is connected to a pull down resistor.

When BE is not set by the CPU, all other bits are unconnected, except for Data and WM, which are connected to a pull down resistor.

The BA signal is used to indicate when read operations are completed. A bus read is aborted (BA is not set) if EX=1 is set.
Write operations are expected to be received no later than the 4th cycle since BE became High.

Combination not specified above is treated as invalid and are not produced in current versions of the bus.

WM0-3 controls the bytes in the output that are written back to memory. They are not used for read transfers. If WMn is set during a write cycle, the corresponding byte in the written word should not be written back to memory. WM may be ignored for memory mapped devices, and are not used when W=0 or IO=1. The contents of the data pins for a given byte that is masked is unspecified, but connected.

### Bus Transfer Protocol

The following timing protocol is used for data transfer:
* Cycle 0(RE): CPU sets BE=1,
* Cycle 0(FE): Address lines, Data lines (for writes), W, WM, IO, and SC become available
* Cycle 1(FE): Data and Address, W, WM, IO, and SC lines latched by downstream,
* Cycle 2(RE): Data, IO, WM, SC lines become unavailable (may not contain salient values),
* Cycle N(RE) (W=0): Downstream sets BA=1
* Cycle N(FE) (W=0): CPU latches Data from downstream.
* Cycle N(FE): Earliest Cycle for CPU to set BE=0.
* Cycle N+1(RE) (W=0, IO=0): CPU May set W=1 to perform a succesive write. If so, downstream releases the Data lines.
* Cycle N+1(FE) (initial W=0, IO=0, W=1): Data lines for successive write are made available.
* Cycle N+2(FE) (init W=0, IO=0, W=1): New Data lines latched by downstream
* Cycle N+k(FE): CPU unsets BE=1,
* Cycle N+k+2(RE): Earliest cycle that CPU will set BE=1 for subsequent bus operation

(N is the number of cycles before downstream sets BA in the case of a read, and is 3 for a write, k is the number of cycles after the earliest point the CPU unsets BE=1)

If, during an initial Memory Read, the CPU performs an immediate write with the same Bus Enable, k is a minimum of 2. Otherwise k is a minimum of 0.

### Coprocessor Transfer Address

Coprocessor registers are identified by A0-4. The Coprocessor to access is identified by N0-4.

As a special case for Writes, An address of 100000 (A5=1, A0-4=0) accesses the Coprocessor Control Register (visible for Coprocessor N as Map 6, Register N)
A5 is not used for reads, and any other values of A0-4 with A5=1 are invalid.

### Co-processor Execution

A Coprocessor Write with CX=1 is an instruction execution. A0-4 are set to 0. The lower 8 bits of the Data contains the function number, and the upper 24 bits contains the payload. 

### LOCK and External BE

The BE pin is IO to allow multiple devices (including coprocessors and other Micron CPUs) to share the same bus for basic operations without complicated coherency logic. While BE is brought high externally, the Micron CPU will not attempt to perform any bus accesses or coprocessor instruction dispatches. Instead, the CPU will wait on them until BE becomes low.

The LOCK pin is a specialized pin for supporting atomic memory accesses. When LOCK is high, the Micron CPU will not attempt to perform any memory accesses, other than instruction fetches. The LOCK pin does not control I/O port accesses or coprocessor register accesses. It also does not block instruction fetches. LOCK is an output pin because it may be used as one in a future chip. Current Micron CPUs do not generate LOCK signals. Bus devices should nonetheless respect this signal

### Primitive Read-Modify-Writes

When reading main memory (IO=0, W=0), after BA becomes High, the CPU may set W=1 at the immediately following rising edge (Cycle N+1). If so, it places data to write on the bus at the falling edge of that cycle.

### Coprocessor Available/Enabled/Busy

For each coprocessor there is a distinct Coprocessor Available/Coprocessor Enabled

## Exceptions and IRQs

EX and IRQ allow the processor to respond to external conditions. EX triggers an urgent external condition (NMI or asynchronous exception). IRQ triggers a non-urgent external condition (Interrupt).

An Exception or Interrupt Request occurs by placing a non-zero value on the bus before the rising edge of a clock pulse. It the value must be held until the falling edge. 

Interrupt Requests are only serviced if the IM signal is set to 0. This is controlled by the CPU's i and t flags (i=1 and t=0), and by the CPU status (`S < 3`). In this state, the IRQ signals are not read (but must still be connected).

Additionally, only certain exceptions are serviced when the CPUs t flag is set (indicating that the CPU is handling an exception) or when the CPU status is Stopped.
There is no hardware indication of this flag's status on its own. Non-serviced exceptions are ignored.

The following values are permissible for the EX pin:

* EX=1: Bus Error (Triggers `Ex[2]`). Ignores Trap flag if recieved during Bus Cycle
* EX=7: Non-maskable Interrupt (Triggers `Ex[7]`)
* EX=8-15: Coprocessor Error. Ignores Trap flag if recieved CPU is waiting on Coprocessor EX-8

## Status

| S | Status  | Notes |
|---|---------|-------|
| 0 | Normal  | Running |
| 1 | Waiting | Waiting on Co-processor |
| 2 | Halted  | Stopped by Halt Instruction (Waiting for Interrupt) |
| 3 | Stopped | Stopped by STOP instruction (Waiting for Reset) |