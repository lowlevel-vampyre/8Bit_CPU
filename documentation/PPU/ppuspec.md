# PPU Specifications
This document holds specifications of the PPU, as well as the description of how it behaves and what constraints it has on it. This document acts as the primary descripter for the PPU when designing it in RTL Design


### Technical Specs:
- Display Output: VGA, 640x480@60Hz
- Virtual Resolution: 320x240@60Hz
- Game Window: 256x240
- Virtual Tile Area: 32 tiles wide, 30 tiles tall (each tile 8x8 pixels)
- Unique Tiles: 64
- Unique Sprites: 16
- Sprite Features: Rotate, Flip (x and y)
- Concurrent Sprites: 4
- Color: 8-bit color (RRR-GGG-BB), 16-color palette
- MMIO: 1200B Tile Map, 16B Object Attribute Memory, 16B Palette Map


### Path of a Pixel:
The process of the PPU is simply described like so: The PPU Master is the Horizontal Counter (HC). This counts each pixel on the screen, and tells the Timing Control Unit (TCU) to increment the Vertical Counter (VC) when neccesary. The TCU delegates on behalf of the HC, controlling the flow of data in the PPU. The TCU spends most of its time selecting tiles and sprites. It uses bits 9-4 of the HC, and bits 9-4 of the VC to index the Tile Map. Each virtual tile on the screen corresponds to a memory location in the Tile Map, which holds the Key for which tile is to be displayed at that location. 

Onc the key is selected, it is passed to the Tile ROM, which has 3-inputs: Tile_Sel, Y_Pixel_Sel, and X_Pixel_Sel. Each tile is stored inside the Tile ROM as a series of bits. Each individual pixel is represented by 4 bits, meaning each byte in the Tile ROM holds the data for 2 pixels. Each 8x8 tile then is 256-bits long. To select a pixel, the 2-bit X_Pixel_Sel (bits 3-2 of HC) are used to index which of the 4 pixel bytes are to be selected. The 3-bit Y_Pixel_Sel (bits 3-1 of VC) are used to determine which row of pixels to select. Finally, the 6-bit Tile Sel is used to index each sprite. All these values are concatonated togethor to form a full 11-bit address, which selects the full 2048-bytes address each pixel byte. The Pixel byte is then output to the Pixel Calculator (PXC).

The Pixel Calculator (PXC) has many inputs that it used to determine which pixel to output. Its primary job is to determine if one of the 4 visible sprites overlaps with the current tile. It does this by first comparing is X and Y inputs (from the HC and VC, respectively), to the 4 Sprit_X and 4 Sprite_Y registers inside the Object Attribute Memory (OAM) that the PXC has a direct connection to. If there is an overlap, it needs to display the sprite pixel over the Tile Pixel. 

While each tile is constrained to a specific Virtual Tile on the screen, sprites are not limited in this way, and are able to occupy any set of pixels in the Game Window. This means that we have to calculate the sprite pixel differently. In the event of an overlap, the PXC selects which of the 4 sprite inouts to use in its calculations. These inputs are Sprite_Y, Sprite_X, Sprite_Math, and Sprite_Index. The PXC takes the X and Y values and computes the difference between the current screen pixel and the sprite location (Pixel Location - Sprite Position). The location of a sprite is considered to be the top left pixel of that sprite (pixel 0x00). The computed X and Y difference is which Sprite Pixel that is to be selected (if this unintuitive, refer to the *Sprite Calculations* section of this document). After it calculates this difference, it then needs to decide what math function to do based on the Sprite_Math input. The 8-bit input is split into 2 4-bit values, the first representing rotation, and the second representing a transformation. Some math is done (outlined in the *Sprite Calculations* section of this document), and out pops a pixel address. This address is passed to the Sprite ROM. 

The Sprite ROM is identical in behavior to the Tile ROM. Once the Pixel byte is selected, it is passed back to the PXC. Before the PXC actually decides what to output, it first uses its inputs to determine whether to output the High or Low Pixel inside the Pixel byte. Once it gets this, it determines if the current sprite pixel is transparent. If so, it passes the Tile Pixel instead. Once the pixel as cleared the PXC, it passes to the Pixel Buffer. The Pixel Buffer is needed to reduce the delay between calculation. The hardware is not quick enough to do all the calculations in time to display the pixel without significant delay. Instead, all pixel calculations are done for the next pixel. At this point, the data to be written to the Pixel Buffer is sitting and waiting to be written, while the Pixel Buffer outputs the current pixel value to the Palette Map.

The Palette Map is a CPU addressable block of memory capable of holding 16 colors at a time. The pixel data that has been passed around this whole time actually just holds which color is to be printed. Each of the 16-colors are each 8-bit, in the bit format of RRR-GGG-BB. Once the color has been selected, it is passed through a mux controlled by the TCU. When the TCU evaluates that it needs to display a pixel, it selects the output of the Palette Map. However, when the VGA signal is in BLANK, it instead flips the mux, outputting all zeros. 

After this mux is a Digital to Analog Converter (DAC) that splits these color values into their Red Green and Blue components, and then uses a resistor ladder to output a 0.7 - 0v voltage to the VGA port. 
