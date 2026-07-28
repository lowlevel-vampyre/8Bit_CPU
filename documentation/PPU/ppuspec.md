# PPU Specifications
This document holds specifications of the PPU, as well as the description of how it behaves and what constraints it operates under. This document acts as the primary descripter for the PPU when designing it in RTL Design


## Technical Specs:
- Pixel Clock: 25.175 MHz
- Display Output: VGA, 640x480 @59.94Hz
- Full Resolution: 800x525
- Virtual Resolution: 320x240 @59.94Hz
- Game Window: 256x240
- Virtual Tile Area: 32 tiles wide, 30 tiles tall (each tile 8x8 pixels)
- Unique Tiles: 64
- Unique Sprites: 16
- Sprite Features: Rotate, Flip (x and y)
- Concurrent Sprites: 4
- Color: 8-bit color (RRR-GGG-BB), 16 concurrent-color palette
- Memory Mapped VRAM: 960 byte Tile Map, 16 byte Object Attribute Memory, 16 byte Palette Map

# Unit Descriptions

## Timing Control Unit:
The Timing Control Unit, or TCU, is the primary controller of data flow inside the PPU. It makes its decisions based on the Horizontal Counter (HC) and the Vertical Counter (VC), and asserts signals to produce the VGA output within the timing spec of the 640x480 @59.94Hz output. The TCU wears a lot of hats, but its primary job is calculating which pixel to display:

##### Path of a Pixel:
The TCU selects its pixels based on the HC and VC. These count each pixel on the screen, and the TCU uses them to to select which pixel exactly needs to be displayed. The TCU does all pixel calculations for the *next* pixel. It stores each fully calculated pixel in the Pixel Buffer, as you will learn about later. To select an actual pixel, the TCU first uses bits 9-4 of the HC, and bits 9-4 of the VC to index the Tile Map. Each virtual tile on the screen corresponds to a memory location in the Tile Map, which holds the Key for which tile is to be displayed at that location. 

Once the key is selected, it is passed to the Tile ROM, which has 3-inputs: Tile_Sel, Y_Pixel_Sel, and X_Pixel_Sel. Each tile is stored inside the Tile ROM as a series of bits. Each individual pixel is represented by 4 bits, meaning each byte in the Tile ROM holds the data for 2 pixels. Each 8x8 tile then is 256-bits long. To select a pixel, the 2-bit X_Pixel_Sel (bits 3-2 of HC) are used to index which of the 4 pixel bytes are to be selected. The 3-bit Y_Pixel_Sel (bits 3-1 of VC) are used to determine which row of pixels to select. Finally, the 6-bit Tile Sel is used to index each sprite. All these values are concatenated together to form a full 11-bit address, which selects the full 2048-bytes address each pixel byte. The Pixel byte is then output to the Pixel Calculator (PXC).

The Pixel Calculator (PXC) has many inputs that it used to determine which pixel to output. Its primary job is to determine if one of the 4 visible sprites overlaps with the current tile. It does this by first comparing is X and Y inputs (from the HC and VC, respectively), to the 4 Sprit_X and 4 Sprite_Y registers inside the Object Attribute Memory (OAM) that the PXC has a direct connection to. If there is an overlap, it needs to display the sprite pixel over the Tile Pixel. 

While each tile is constrained to a specific Virtual Tile on the screen, sprites are not limited in this way, and are able to occupy any set of pixels in the Game Window. This means that we have to calculate the sprite pixel differently. In the event of an overlap, the PXC selects which of the 4 sprite inputs to use in its calculations. These inputs are Sprite_Y, Sprite_X, Sprite_Math, and Sprite_Index. The PXC takes the X and Y values and computes the difference between the current screen pixel and the sprite location (Pixel Location - Sprite Position). The location of a sprite is considered to be the top left pixel of that sprite (pixel 0x00). The computed X and Y difference is which Sprite Pixel that is to be selected (if this unintuitive, refer to the ***Sprite Calculations*** section of this document). After it calculates this difference, it then needs to decide what math function to do based on the Sprite_Math input. The 8-bit input is split into 2 4-bit values, the first representing rotation, and the second representing a transformation. Some math is done (outlined in the ***Sprite Calculations*** section of this document), and out pops a pixel address. This address is passed to the Sprite ROM. 

The Sprite ROM is identical in behavior to the Tile ROM. Once the Pixel byte is selected, it is passed back to the PXC. Before the PXC actually decides what to output, it first uses its inputs to determine whether to output the High or Low Pixel inside the Pixel byte. Once it gets this, it determines if the current sprite pixel is transparent. If so, it passes the Tile Pixel instead. Once the pixel as cleared the PXC, it passes to the Pixel Buffer. The Pixel Buffer is needed to reduce the delay between calculation. The hardware is not quick enough to do all the calculations in time to display the pixel without significant delay. Instead, all pixel calculations are done for the next pixel. At this point, the data to be written to the Pixel Buffer is sitting and waiting to be written, while the Pixel Buffer outputs the current pixel value to the Palette Map.

The Palette Map is a CPU addressable block of memory capable of holding 16 colors at a time. The pixel data that has been passed around this whole time actually just holds which color is to be printed. Each of the 16-colors are each 8-bit, in the bit format of RRR-GGG-BB. Once the color has been selected, it is passed through a mux controlled by the TCU. When the TCU evaluates that it needs to display a pixel, it selects the output of the Palette Map. However, when the VGA signal is in BLANK, it instead flips the mux, outputting all zeros. 

After this mux is a Digital to Analog Converter (DAC) that splits these color values into their Red Green and Blue components, and then uses a resistor ladder to output a 0.7 - 0v voltage to the VGA port. 

##### Other Timing Control Unit Behaviors
The Timing Control Unit (TCU), On top of controlling the flow of pixel data, also controls the Counters, the CPU V-Blank Pins, and the Memory Write decoding for VRAM. For the counters, the TCU is responsible incrementing the VC when then HC is finished, and for clearing each counter when it has reached the end of its count (800 for HC, 525 for VC). When the Counters are in BLANK (H-Blank or V-Blank), the TCU makes sure that the VGA port only gets black pixels. In V-Blank, the TCU also flips a V-Blank pin that the CPU reads to know it is time to update VRAM. If the V-Blank pin is not active, the CPU should not write to VRAM. When the CPU does write to VRAM, the TCU acts as a controller to determine, based on the address, which of its internal memory devices to write to. Additionally, it is responsible for strobing the H-Sync and V-Sync signals on the VGA port. 

### VRAM
VRAM is split into 6 separate modules that hold some information for the PPU. The specifics of each are detailed below:

##### Tile Map
- Size: 960 bytes
- Writable: Yes
- Memory Mapped to Addresses 0x800 - 0xBBF
- Inputs: WD, ADDR, WE
- Outputs: TileSel
<br>Each byte in this map corresponds to a virtual tile on the screen. The byte holds the ID for 1 of the 64 possible tiles that can be drawn. At the moment, only the bottom 6-bits are used as a key. The top 2 bits are unused. The output of this module is fed to the Tile ROM Module. This module is CPU writable, and should only be written during VBLANK, as the contents are not being used. Modifying while the PPU is drawing the screen can create tearing.

##### Tile ROM
- Size: 2048 bytes
- Writable: No
- Inputs: X_Pix_Sel, Y_Pix_Sel, TileSel
- Outputs: Pixel_Byte
<br>This ROM holds the pixel data for 64 unique 8x8 tiles. The data is sorted sequentially, with each pixel taking up 4-bits of data to represent a 4-bit index of Palette Map. Each tile takes up 32-bytes of data ([8 * 8 * 4] / 8). Since each pixel is 4-bits, each byte stores the data for 2 pixels. It is simple to address each pixel and each line using bit-manipulation by understanding what each bit in the Tile ROM Address corresponds to. Bits 1-0 correspond to the Pixel Byte for each line. Bits 4-2 correspond to which each line of a Tile. And bits 11 - 5 correspond to a unique tile. This unit does not split the pixel byte, but simply outputs the selected byte.

##### Sprite ROM
- Size: 512 bytes
- Writable: No
- Inputs: X_Pix_Sel, Y_Pix_Sel, SpriteSel
- Outputs: Pixel_Byte
<br>This ROM holds the pixel data for 16 unique 8x8 sprites. It behaves and is architected exactly the same as the above Tile ROM.

##### Object Attribute Memory (OAM)
- Size: 16 bytes
- Writable: Yes
- Memory Mapped to Addresses 0xBC0 - 0xBCF
- Inputs: WD, ADDR, WE
- Outputs: 4x(Sprite_X, Sprite_Y, Sprite_Math, Sprite_Index)
<br>Each byte in this module is directly connected to the Pixel Calculator (PXC). There is a unique output for each of the 4 possible concurrent sprites. Each sprites requires 4 bytes of data. The first and second byte store the X and Y position of the sprite respectively. The actual position of a sprite is considered to be the top left pixel. The third byte holds the Sprite Math register, which is mapped like so:
    - Bit 0: X-Flip
    - Bit 1: Y-Flip
    - Bit 2: Diagonal-Flip
    - Bit 3: Sprite Disable
    - Bit 4: Palette0
    - Bit 5: Palette1
    - Bit 6: unused
    - Bit 7: Shake

<br>These each correspond to a different operation in the Pixel Calculator. This allows full rotation, transformation, invisibility, effects using different palettes, and a single pixel shake. I go into more detail on how these are utilized in the ***Pixel Calculator*** section. 

The fourth and final byte represents which of the 16 unique sprites to render. The sprite index is passed into the Pixel Calculator, then to the Sprite ROM

##### Pixel Buffer
- Size: 1 byte
- Writable: No
- Inputs: IN
- Outputs: OUT
<br>This Buffer is designed to hold the data of the previous pixel while the next pixel is being calculated. This ensures that the pixel is held high for the entirety of the Pixel Clock duration. When the clock oscillates, the data waiting to be saved is saved, and the output of the Pixel Buffer changes and quickly propagates to the VGA port

##### Palette Map
- Size: 64 bytes
- Writable: Yes
- Memory Mapped to Addresses 0xBD0 - 0xC0F
- Inputs: WD, ADDR, WE
- Outputs: TileSel
<br>Each byte in this map holds an 8-bit color. This is indexed by the 6-bits stored inside the Pixel Buffer. While only the bottom 16 bytes (aka Palette 0) are addressable by the Tile ROM. However, the Sprites have access to the other 3 palettes are able to be used by the sprites using the Palette0 and Palette1 bits in the Sprite_Math byte. This is a low latency translation unit to convert from a color code to an 8-bit color. 
