The CPU register file should be switched to a 9-bit output for each register to support the $SP. That 
final bit could be paired to zero unless all 4-bits of the reg-select are high

I will likely need an address decoder to select between RAM adn MMIO

It could be cool to add an interrupt handler. Each interrupt would be paired to a pin, rather than specific addresses. The ideas are:
- Controller Start Press
- Audio Pin 1
- Audio Pin 2
- Timer (IDK BRO)

The idea would be that all the PC control pins go through the interrupt handler, and if a pin gets flipped, it saves the current state to the Return Stack, and then loads an address from the interrupt handler registers into the PC.

Need to update the CPU to correctly show its MMIO inputs. 
