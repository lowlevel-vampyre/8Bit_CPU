# PPU Specifications
This document holds specifications of the PPU, as well as the description of how it behaves and what constraints it has on it. This document acts as the primary descripter for the PPU when designing it in RTL Design

## Technical Specs:
- Display Output: VGA, 640x480@60Hz
- Virtual Resolution: 320x240@60Hz
- Game Window: 256x240
- Color: 8-bit color, 16-color palette
- Virtual Tile Area: 40 tiles wide, 30 tiles tall (each tile 8x8 pixels)
- Unique Tiles: 64
- Unique Sprites: 16
- Sprite Features: Rotate, Flip (x and y)
- Concurrent Sprites: 4
- MMIO: 1200B Tile Map, 16B Object Attribute Memory, 16B Color Map


## Overall Description:
The process of the PPU is simply described like so: The PPU Master is the Horizontal Counter (HC). This counts each pixel on the screen, and tells the timing control unit to increment the Vertical Counter (VC) when neccesary. The Timing Control Unit delegates on behalf of the HC, controlling the flow of data in the PPU. The Timing Control Unit spends most of its time selecting tiles and sprites. It uses bits 9-4 of the HC, and bits 9-4 of the VC to index the Tile Map. Each virtual tile on the screen corresponds to a memory location in the Tile Map
