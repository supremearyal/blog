TI Stellaris ARM Cortex-M4F Assembly Tutorial
#############################################

:date: 2014-07-19
:tags: arm, cortex, assembly, stellaris, tutorial

I have a TI Stellaris development board which has a ARM Cortex-M4F processor. I
wanted to dive into ARM development and figured starting with assembly would be
more useful to understand what’s going on and also be helpful for working with
other ARM boards/chips. So, the TI Stellaris uses the LM4F120H5QR chip
(datasheet `here <http://www.mouser.com/ds/2/405/lm4f120h5qr-124014.pdf>`__)
which is based on the ARM Cortex-M4F processor core. The M4 is
broadly grouped as a Cortex M core which includes the M0 (less powerful) and M3
(M4 minus DSP features). And the F means that a hardware floating point unit is
included.

So, let’s start with a hello world project for microcontrollers: lighting a led
(we’ll get to blinking later). Lighting a led is simply a matter of setting
some pin value but you have to do a bunch of setup before you can set the state
of a pin. First, let’s find a led to light up on the board. The Stellaris has
RGB leds in port F. The red led is connected to port F, pin 1. Before we can
set the pin, we have to enable port F. After that, it’s like other
microcontrollers where you set the pin direction, pin function, pin state. In
assembly::

    @ constant 1 in r1
    mov r1, #1
    
    @ enable port f
    @ bitband alias calculation
    @ base address 0x400FE000 + 0x608
    @ => 0x42000000 + 0xFE608 * 32 + 5 * 4
    ldr r0, =BB_RCGCGPIO_PORTF
    str r1, [r0]
    
    @ set pin 1 direction to output
    @ base address 0x40025000 + 0x400
    @ => 0x42000000 + 0x25400 * 32 + 1 * 4
    ldr r0, =BB_PORT_F_PIN_1_DIR
    str r1, [r0]
    
    @ turn on digital feature of pin 1
    @ base address 0x40025000 + 0x51C
    @ => 0x42000000 + 0x2551C * 32 + 1 * 4
    ldr r0, =BB_PORT_F_PIN_1_DEN
    str r1, [r0]
    
    @ set pin 1 output value to 1
    @ base address 0x400025000 + 0x3FC
    @ => 0x42000000 + 0x253FC * 32 + 1 * 4
    ldr r0, =BB_PORT_F_PIN_1_DATA
    str r1, [r0]

The addresses for the registers come from the datasheet. The only weird address
is BB_PORT_F_PIN_1_DATA where 0x3FC is the mask which means all bits get set;
this can be used to avoid extra computation when only setting/clearing a single
bit. These aren’t actual registers on the ARM used for computation but more
like registers for configuring GPIO among other things. I’m using the
bit-banding feature of the Cortex M4 here. Bit banding basically allows you to
set a bit of a memory location atomically. This isn’t necessary here, the bit
band region addresses (as opposed to the aliases) can be used as well to set a
bit non-atomically by getting the existing number and ORing a mask. Basically
bit-banding translates every bit to a single memory location with the formula::

    bit_word_offset = (byte_offset x 32) + (bit_number x 4)

The program from above isn’t ready to assemble. The addresses have to be
defined and extra things need to be added so it can be flashed to the
Stellaris. I’ll be using the GCC ARM toolchain from
`here <https://launchpad.net/gcc-arm-embedded>`__ to assemble
the code. To flash the final binary, I’m using
`lm4tools <https://github.com/utzig/lm4tools>`__. To define the memory
addresses, we can use the .equ assembly directive. We also have to let the
processor know where to start running the program. This is done with the vector
table. The vector table is a table of interrupts. We’re only interested here in
the reset interrupt.  Memory location 0x0 on the Cortex M4 defines the stack
pointer, we can just set that to 0x20008000. Memory location 0x4 defines the
location from where the program starts running. We have to tell the assembler
that we are assembling to Thumb mode which is used by the Cortex M4 as opposed
to ARM mode and mark our entry point as a function so the vector table works.
The final code looks like::

        @ addresses
        .equ STACK_TOP, 0x20008000
        .equ BB_RCGCGPIO_PORTF, 0x43FCC114
        .equ BB_PORT_F_PIN_1_DIR, 0x424A8004
        .equ BB_PORT_F_PIN_1_DEN, 0x424AA384
        .equ BB_PORT_F_PIN_1_DATA, 0x424A7F84
    
        @ code
        .text
        .syntax unified
        .thumb
        .global _start
        .type start, %function
    
        @ vector table
    _start:
        .word STACK_TOP, start
    start:
        mov r1, #1
        ldr r0, =BB_RCGCGPIO_PORTF
        str r1, [r0]
        ldr r0, =BB_PORT_F_PIN_1_DIR
        str r1, [r0]
        ldr r0, =BB_PORT_F_PIN_1_DEN
        str r1, [r0]
        ldr r0, =BB_PORT_F_PIN_1_DATA
        str r1, [r0]
    
        @ loop forever
    done:
        b done
        .end

To compile, we’ll use as to first compile to an object file::

    arm-none-eabi-as -mcpu=cortex-m4 hello.s -o hello.o

Then, we link the code with ld::

    arm-none-eabi-ld -Ttext 0x0 -o hello.axf hello.o

Finally, we convert to binary::

    arm-none-eabi-objcopy -O binary hello.axf hello.bin

To check that everything assembled properly, we can use objdump::

    arm-none-eabi-objdump -m arm -b binary -M force-thumb -D hello.bin

You should see::

    00000000 <.data>:
       0:   8000            strh    r0, [r0, #0]
       2:   2000            movs    r0, #0
       4:   0009            movs    r1, r1
       6:   0000            movs    r0, r0
       8:   f04f 0101       mov.w   r1, #1
       c:   4804            ldr     r0, [pc, #16]   ; (0x20)
       e:   6001            str     r1, [r0, #0]
      10:   4804            ldr     r0, [pc, #16]   ; (0x24)
      12:   6001            str     r1, [r0, #0]
      14:   4804            ldr     r0, [pc, #16]   ; (0x28)
      16:   6001            str     r1, [r0, #0]
      18:   4804            ldr     r0, [pc, #16]   ; (0x2c)
      1a:   6001            str     r1, [r0, #0]
      1c:   e7fe            b.n     0x1c
      1e:   0000            movs    r0, r0
      20:   c114            stmia   r1!, {r2, r4}
      22:   43fc            mvns    r4, r7
      24:   8004            strh    r4, [r0, #0]
      26:   424a            negs    r2, r1
      28:   a384            add     r3, pc, #528    ; (adr r3, 0x23c)
      2a:   424a            negs    r2, r1
      2c:   7f84            ldrb    r4, [r0, #30]
      2e:   424a            negs    r2, r1

The instructions go from 0x8 to 0x1c, the rest is the vector table and the
data. Finally to flash::

    lm4flash hello.bin

The red led should light up now.

More info
---------
* http://fun-tech.se/stm32/index.php
* http://e2e.ti.com/support/microcontrollers/tiva_arm/f/908/t/243460.aspx
* http://en.wikipedia.org/wiki/ARM_Cortex-M
* http://www.bravegnu.org/gnu-eprog/index.html
* The Definitive Guide to the ARM Cortex-M3
