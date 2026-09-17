# Z80 SCC and Z8536 CIO fixes for an interrupt driven daisy chain

Two device fixes found while bringing up Zilog's ZEUS on an emulated S8000. Both
only show up when a driver feeds the serial port from its own interrupt routine
and the devices sit on a shared Z80 style daisy chain, which is why they had
survived.

## The SCC transmit interrupt

`z80scc.cpp`, `data_write()`. Loading a character into the transmit buffer did
not rearm the interrupt. The `m_tx_int_disarm` flag is set by the "Reset Tx Int
Pending" command, WR0 = 0x28, and the data sheet says that suppresses interrupts
only until the next character is loaded. MAME cleared it much later, at the end
of the following `tra_complete()`, so it swallowed the buffer empty interrupt for
that very character and a driver that feeds from the interrupt stopped after one
byte.

The symptom is a console that dies after a few characters, intermittently,
depending on where the write falls against the shifter.

## The CIO under-service level

`z8536.cpp`, `check_interrupt()`. The device only announced a change when its own
interrupt line moved, not when it released its under-service level. With that
level set it returns `Z80_DAISY_IEO` and blocks everything further down the
chain, so when it cleared nobody re-evaluated and the SCC below it never got
served again.

The fix publishes the change when the under-service level moves too, which is why
the patch also touches `z8536.h`: the level has to be saved with the rest of the
device state.

## What was tried and dropped

Implementing the SCC's TRxC clock and its RETI handling. Both were written and
measured, and the system behaves the same without them. Worth saying because they
look necessary and are not.

Measuring fixes one at a time means turning the others off: the CIO one looked
unnecessary until it was the only one left.

## Layout

    patches/z80scc-z8536-daisy-chain.patch   against MAME 0.289

## Building

    cd <mame>
    patch -p1 < <this>/patches/z80scc-z8536-daisy-chain.patch

## License

BSD-3-Clause, the same as the MAME source it is built against. See LICENSE.
