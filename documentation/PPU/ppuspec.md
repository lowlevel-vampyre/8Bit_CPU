# PPU Specifications
This document holds specifications of the PPU, as well as the description of how it behaves and what constraints it has on it. This document acts as the primary descripter for the PPU when designing it in RTL Design


### Technical Specs:
- Display Output: VGA, 640x480@60Hz
- Virtual Resolution: 320x240@60Hz
- Color: 8-bit color, 16-color palette
- Game Window: 256x240
- Virtual Tile Area: 32 tiles wide, 30 tiles tall (each tile 8x8 pixels)
- Unique Tiles: 64
- Unique Sprites: 16
- Sprite Features: Rotate, Flip (x and y)
- Concurrent Sprites: 4
- MMIO: 1200B Tile Map, 16B Object Attribute Memory, 16B Color Map


### Overall Description:
The process of the PPU is simply described like so: The PPU Master is the Horizontal Counter (HC). This counts each pixel on the screen, and tells the timing control unit to increment the Vertical Counter (VC) when neccesary. The Timing Control Unit delegates on behalf of the HC, controlling the flow of data in the PPU. The Timing Control Unit spends most of its time selecting tiles and sprites. It uses bits 9-4 of the HC, and bits 9-4 of the VC to index the Tile Map. Each virtual tile on the screen corresponds to a memory location in the Tile Map, which holds the Key for which tile is to be displayed at that location. 

Onc the key is selected, it is passed to the Tile ROM, which has 3-inputs: Tile_Sel, Y_Pixel_Sel, and X_Pixel_Sel. Each tile is stored inside the Tile ROM as a series of bits. Each individual pixel is represented by 4 bits, meaning each byte in the Tile ROM holds the data for 2 pixels. Each 8x8 tile then is 256-bits long. To select a pixel, the 2-bit X_Pixel_Sel (bits 3-2 of HC) are used to index which of the 4 pixel bytes are to be selected. The 3-bit Y_Pixel_Sel (bits 3-1 of VC) are used to determine which row of pixels to select. Finally, the 6-bit Tile Sel is used to index each sprite. All these values are concatonated togethor to form a full 11-bit address, which selects the full 2048-bytes address each pixel byte. The Pixel byte is then output to the Pixel Calculator.

The Pixel Calculator has many inputs that it used to determine which pixel to output. Its primary job is to determine if one of the 4 visible sprites overlaps with the current tile. It does this by first comparing is X and Y inputs (from the HC and VC, respectively), to the 4 Sprit_X and 4 Sprite_Y registers that is connected to. If there is an overlap, it needs to display the sprite pixel over the Tile Pixel. While each tile is constrained to a specific Virtual Tile on the screen, sprites are not limited in this way, and are able to occupy any set of pixels in the Game Window. This means that we have to calculate the sprite pixel differently. In the event of an overlap, the Pixel Calculator computes the difference between the current screen pixel and the sprite location (Pixel Location - Sprite Position). The location of a sprite is considered to be the top left pixel of that sprite (pixel 0x00). The computed X and Y difference is which Sprite Pixel that is to be selected. 
