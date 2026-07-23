The CPU register file should be switched to a 9-bit output for each register to support the $SP. That 
final bit could be paired to zero unless all 4-bits of the reg-select are high

It could be cool to add an interrupt handler. Each interrupt would be paired to a pin, rather than
specific addresses. The ideas are:
- Controller Start Press
- Audio Pin (for whatever idk)
- Timer (IDK BRO)
- Another interrupt

The idea would be that all the PC control pins go through the interrupt handler, and if a pin gets 
flipped, it saves the current state to the Return Stack, and then loads an address from the interrupt
handlers into the PC.
