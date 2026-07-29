# Pixel Processing Unit Specification Sheet
This document holds specifications of the Pixel Processing Unit (PPU), as well as the description of how it behaves and what constraints it operates under. This document acts as the primary descripter for the PPU when designing it in RTL Design


## Technical Specs:
- Pixel Clock: 25.175 MHz
- Display Output: VGA, 640x480 @59.94Hz
- Full Resolution (including blanks): 800x525
- Virtual Resolution: 320x240
- Game Window: 256x240
- Virtual Tile Area: 32 tiles wide, 30 tiles tall (each tile 8x8 pixels)
- Unique Tiles: 64
- Unique Sprites: 16
- Sprite Features: Rotate, Flip (x and y)
- Concurrent Sprites: 4
- Color: 8-bit color (RRR-GGG-BB), 4x16 color palettes
- Memory Mapped VRAM: 960 byte Tile Map, 16 byte Object Attribute Memory, 64 byte Palette Map

<br>

# Unit Descriptions
The following are useful descriptions of the behavior and ports of each unit in the PPU. For added clarity, you can reference the PPU Datapath while reading.

## Timing Control Unit:
The Timing Control Unit, or TCU, is the primary controller of data flow inside the PPU. It makes its decisions based on the Horizontal Counter (HC) and the Vertical Counter (VC), and asserts signals to produce the VGA output within the timing spec of the 640x480 @59.94Hz output. The TCU wears a lot of hats, but its primary job is delegating the flow of data.

##### Inputs: 
- X
- Y
- PPU_WE
- PPU_Write_Addr

##### Outputs: 
- V_INC 
- V_RST 
- H_RST 
- Y_Pix_Sel
- X_Pix_Sel
- Tile_WE 
- Tile_Addr (8-0)
- Sprite_WE 
- Sprite_Addr (3-0)
- Color_WE 
- Color_Addr (5-0)
- BLANK 
- Status (7-0)
- V_Sync 
- H_Sync
  
### Path of a Pixel:

##### Step 1: Counters
The TCU selects its pixels based on the Horizontal Counter (HC) and the Vertical Counter (VC). These count each pixel on the screen, and the TCU uses them to to select which pixel exactly needs to be displayed. The TCU does all pixel calculations for the *next* pixel. It stores each fully calculated pixel in the Pixel Buffer, as you will learn about later. To select an actual pixel, the TCU first uses bits 9-4 of the HC, and bits 9-4 of the VC to index the Tile Map. Each virtual tile on the screen corresponds to a memory location in the Tile Map, which holds the Key for which tile is to be displayed at that location. 

##### Step 2: Tile Selection
Once the key is selected, it is passed to the Tile ROM, which has 3-inputs: Tile_Sel, Y_Pixel_Sel, and X_Pixel_Sel. Each tile is stored inside the Tile ROM as a series of bits. Each individual pixel is represented by 4 bits, meaning each byte in the Tile ROM holds the data for 2 pixels. Each 8x8 tile then is 256-bits long. To select a pixel, the 2-bit X_Pixel_Sel (bits 3-2 of HC) are used to index which of the 4 pixel bytes are to be selected. The 3-bit Y_Pixel_Sel (bits 3-1 of VC) are used to determine which row of pixels to select. Finally, the 6-bit Tile Sel is used to index each sprite. All these values are concatenated together to form a full 11-bit address, which selects the full 2048-bytes address each pixel byte. The Pixel byte is then output to the Pixel Calculator (PXC).

##### Step 3: Calculating the Pixel Output
The Pixel Calculator (PXC) has many inputs that it used to determine which pixel to output. Its primary job is to determine if one of the 4 visible sprites overlaps with the current tile. It does this by first comparing is X and Y inputs (from the HC and VC, respectively), to the 4 Sprit_X and 4 Sprite_Y registers inside the Object Attribute Memory (OAM) that the PXC has a direct connection to. If there is an overlap, it needs to display the sprite pixel over the Tile Pixel. 

While each tile is constrained to a specific Virtual Tile on the screen, sprites are not limited in this way, and are able to occupy any set of pixels in the Game Window. This means that we have to calculate the sprite pixel differently. In the event of an overlap, the PXC selects which of the 4 sprite inputs to use in its calculations. These inputs are Sprite_Y, Sprite_X, Sprite_Math, and Sprite_Index. The PXC takes the X and Y values and computes the difference between the current screen pixel and the sprite location (Pixel Location - Sprite Position). The location of a sprite is considered to be the top left pixel of that sprite (pixel 0x00). The computed X and Y difference is which Sprite Pixel that is to be selected (if this unintuitive, refer to the ***Sprite Calculations*** section of this document). After it calculates this difference, it then needs to decide what math function to do based on the Sprite_Math input. Some math is done (outlined in the ***Sprite Calculations*** section of this document), and out pops a pixel address. This address is passed to the Sprite ROM. 

##### Step 4: Comparing Pixels
The Sprite ROM is identical in behavior to the Tile ROM. Once the Pixel byte is selected, it is passed back to the PXC. Before the PXC actually decides what to output, it first uses its inputs to determine whether to output the High or Low Pixel inside the Pixel byte. Once it gets this, it determines if the current sprite pixel is transparent. If so, it passes the Tile Pixel instead. 

Sprite Pixels have bits in Sprite_Math that allow them to access more colors, so if a sprite pixel is to be output, those bits are concatenated to the output. If a tile is printed instead, the pixel is zero padded.

Once the pixel has cleared the PXC, it passes to the Pixel Buffer. The Pixel Buffer is needed to reduce the delay between calculation, and to provide and instant response time. All calculations are not done for the current output pixel, but the *next* one. The hardware is not quick enough to do all the calculations in time to display the pixel without significant delay. At this point, the data to be written to the Pixel Buffer is sitting and waiting to be written, while the Pixel Buffer outputs the current pixel value to the Palette Map that was calculated in the previous cycle.

##### Step 5: The Palette Map
The Palette Map is a CPU addressable block of memory capable of holding 64 colors at a time in 4 different palettes. Though there are lots of colors, tiles can only access the first palette. The remaining 3 are reserved for sprites. The pixel data that has been passed around this whole time actually just holds which color is to be output. Each of the 64 colors are  8-bit RGB values, in the bit format of RRR-GGG-BB. Once the color has been selected, it is passed through a mux controlled by the TCU. When the TCU evaluates that it needs to display a pixel, it selects the output of the Palette Map. However, when the VGA signal is in BLANK, it instead flips the mux, outputting all zeros. 

After this mux is a Digital to Analog Converter (DAC) gets the wire holding each color value, split into their Red Green and Blue components, and then uses a resistor ladder to output a 0.7 - 0v voltage to the VGA port. 

### Other Timing Control Unit Behaviors:
The Timing Control Unit (TCU), On top of controlling the flow of pixel data, also controls the Counters, the CPU V-Blank Pins, and the Memory Write decoding for VRAM. 

##### Enforcing Black Pixels:
For the counters, the TCU is responsible incrementing the VC when then HC is finished, and for clearing each counter when it has reached the end of its count (800 for HC, 525 for VC). When the counters are in BLANK (H-Blank or V-Blank), the TCU makes sure that the VGA port only gets black pixels. 

##### Status Information:
The TCU also passes a status byte to the CPU. The status byte signals the CPU when the PPU is in either V-Blank (bit 7) or H-Blank (bit 6), and also gives a 6-bit frame counter (bits 5-0). Since there are 60 frames displayed per second, a 6-bit counter is pretty close to a per-second counter (time drift of 6% or 0.06 seconds). Given the inaccuracy, it is not a good idea to use this as an actual timer, but instead a rough estimate for effects of short duration (instead, use the timer interrupt!).

<br>

## Pixel Calculator:
The Pixel Calculator, or PXC, is the controller to decide which pixel to output. It has a direct connection to the Object Attribute Memory, and uses this data to decide when to output tile data or when to output sprite data. 

##### Inputs: 
- X
- Y
- Tile_Pixel_Byte
- Sprite_Pixel_Byte
- 4xSprite_X
- 4xSprite_Y
- 4xSprite_Math
- 4xSprite_Index

##### Outputs: 
- Sprite_Index
- X_Sprite_Pix_Sel
- Y_Sprite_Pix_Sel
- Pixel_Data
  
#### Calculating which pixel to output:
The process for calculating which pixel to output is decently complicated, and requires many hops between VRAM units, and many arithmetic computations. The first step is determining whether a Sprite overlaps with the current pixel (If the Sprite Disable bit in Sprite_Math is on, this following step is skipped and the tile pixel is passed through). The way to do this is simple: A sprite is always an 8x8 image, and so for each axis (x & y) you subtract the sprite position from the pixel position. If the result is between 0 and 7, then you have an overlap (when calculating the overlap, the Shake bit from Sprite_Math is added to the Sprite X position). In the event of an overlap, the PXC calculates which pixel it needs to grab from the Sprite ROM. How it does this is detailed in the ***Sprite Calculations*** section. 

Once the Sprite Pixel Address is passed to the Sprite ROM, the resulting Pixel byte is returned to the PXC. Now the PXC has both pixel bytes, it splits them both into high and low pixels. Then, if the Sprite pixel is transparent, the tile data is output and padded with zeros to become 6 bits. Otherwise, the sprite pixel is output, and the output is padded with the values of Palette0 (bit 4) and Palette1 (bit 5). At this point, the PXC's job is complete, and the pixel is stored in the Pixel Buffer for the following cycle.

#### Sprite Calculations
Earlier I mentioned that the PXC does some funky calculations to determine which sprite pixel it wants to select. The PXC performs many functions, and each correspond to a bit in Sprite_Math, so I will just explain what each one does:

##### Basic Calculation:
The general format of any PXC sprite calculation starts with taking the Pixel position and comparing it against each of the 4 Concurrent Sprites stored inside the OAM. This is done by subtracting the position of the Sprite from the Pixel Position. To further explain with an example, say you wanted to output the top left pixel at the current screen pixel. Since the X and Y coordinates correspond to that top left corner, the screen coordinates and the sprite position should produce a zero sum after the subtraction, giving you the correct coordinates. Instead, say you wanted to output the 6th pixel on the 3rd rank/line. The X position of the screen pixel is 6 larger than the sprite location. The same logic applies to the Y position. You can then see how the difference between the position always indicates which pixel of the sprite needs to be printed.

##### Transformations:
It would be no fun, and a little tedious, if all the PPU could do was print static sprites. To save on space, and because it is conceptually easy to implement, we are adding the feature of sprite transformations. The 3 bits that select which sprite transformation to do are the X-Flip (bit 0), Y-Flip (bit 1), and Diag-Flip (bit 2). These work together to perform any rotation and transformation combination your heart could desire. The X and Y flips are simple. Once the pixel address is computed, the bits are simply flipped to for the axis you want to flip, so it starts its drawing from the opposite side. 

The Diagonal-Flip is almost as simple. To do this, we simply swap the X and Y coordinates as they are passed into the Sprite Map. This is a little hard to visualize exactly *why* this works, but I have a challenge to help out. Make a 4x4 grid on paper, and fill it them with some random shading. Make each cell unique. Then, make a separate 4x4, and walk through the first 4x4, copying the values from left to right, top to bottom, but putting the values from the rows inside the columns. What you will find is that your image is now rotated and mirrored. By then Applying a horizontal flip, you have a 90 degree rotation. All effects can be combined to achieve a full 360 degree rotation, as well as any flips needed:

- 90 degree rotation: Diagonal-Flip --> X-Flip
- 180 degree rotation: Y-Flip --> X-Flip
- 270 degree rotation: Diagonal-Flip --> Y-Flip

<br>The only complication is that these operations are not commutative, so the proper order is Diagonal Flip --> X-Flip and/or Y-Flip (the flips are commutative). 

##### Sprite Disable:
This simply disables the sprite temporarily. This can be used to make a sprite invisible, and modulating this each frame allows for an "Invisible" effect while still being actually visible. This does not work well while moving, and barely works well while still. Depending on what frequency you modulate at, you can get different effects. A 50% duty cycle will most likely work best for "Invisible" visuals.

##### Dither:
The Dither bit is an idea that could give a cool effect equivalent to invisibility. It changes its behaviors depending on if we are on an odd or even frame. On an odd frame, it disables the odd x pixel on odd y lines, and even x pixel on even y lines. The behavior is flipped for even frames. This ***should, hopefully*** produce a convincing invisibility effect. Or it could look like trash.

##### Shake:
The shake bit is always added to the Sprites X coordinates. This offsets the sprite by one without changing their position. Modulating this every frame allows a "shake" or "shiver" effect.

<br>

## VRAM
VRAM is split into 6 separate modules that hold some information for the PPU. The specifics of each are detailed below:

#### Tile Map
- Size: 960 bytes
- Writable: Yes
- Memory Mapped to Addresses 0x800 - 0xBBF
- **Inputs:** WD, ADDR, WE
- **Outputs:** TileSel
<br>Each byte in this map corresponds to a virtual tile on the screen. The byte holds the ID for 1 of the 64 possible tiles that can be drawn. At the moment, only the bottom 6-bits are used as a key. The top 2 bits are unused. The output of this module is fed to the Tile ROM Module. This module is CPU writable, and should only be written during VBLANK, as the contents are not being used. Modifying while the PPU is drawing the screen can create tearing.

#### Tile ROM
- Size: 2048 bytes
- Writable: No
- **Inputs:** X_Pix_Sel, Y_Pix_Sel, TileSel
- **Outputs:** Pixel_Byte
<br>This ROM holds the pixel data for 64 unique 8x8 tiles. The data is sorted sequentially, with each pixel taking up 4-bits of data to represent a 4-bit index of Palette Map. Each tile takes up 32-bytes of data ([8 * 8 * 4] / 8). Since each pixel is 4-bits, each byte stores the data for 2 pixels. It is simple to address each pixel and each line using bit-manipulation by understanding what each bit in the Tile ROM Address corresponds to. Bits 1-0 correspond to the Pixel Byte for each line. Bits 4-2 correspond to which each line of a Tile. And bits 11 - 5 correspond to a unique tile. This unit does not split the pixel byte, but simply outputs the selected byte.

#### Sprite ROM
- Size: 512 bytes
- Writable: No
- **Inputs:** X_Pix_Sel, Y_Pix_Sel, SpriteSel
- **Outputs:** Pixel_Byte
<br>This ROM holds the pixel data for 16 unique 8x8 sprites. It behaves and is architected exactly the same as the above Tile ROM.

#### Object Attribute Memory (OAM)
- Size: 16 bytes
- Writable: Yes
- Memory Mapped to Addresses 0xBC0 - 0xBCF
- **Inputs:** WD, ADDR, WE
- **Outputs:** 4x(Sprite_X, Sprite_Y, Sprite_Math, Sprite_Index)
<br>Each byte in this module is directly connected to the Pixel Calculator (PXC). There is a unique output for each of the 4 possible concurrent sprites. Each sprites requires 4 bytes of data. The first and second byte store the X and Y position of the sprite respectively. The actual position of a sprite is considered to be the top left pixel. The third byte holds the Sprite Math register, which is mapped like so:

**Sprite Math Bit Layout:**
    - Bit 0: X-Flip
    - Bit 1: Y-Flip
    - Bit 2: Diagonal-Flip
    - Bit 3: Sprite Disable
    - Bit 4: Palette0
    - Bit 5: Palette1
    - Bit 6: Dither
    - Bit 7: Shake

These each correspond to a different operation in the Pixel Calculator. This allows full rotation, transformation, invisibility, effects using different palettes, and a single pixel shake. I go into more detail on how these are utilized in the ***Pixel Calculator*** section. 

The fourth and final byte represents which of the 16 unique sprites to render. The sprite index is passed into the Pixel Calculator, then to the Sprite ROM

##### Pixel Buffer
- Size: 1 byte
- Writable: No
- **Inputs:** IN
- **Outputs:** OUT
<br>This Buffer is designed to hold the data of the previous pixel while the next pixel is being calculated. This ensures that the pixel is held high for the entirety of the Pixel Clock duration. When the clock oscillates, the data waiting to be saved is saved, and the output of the Pixel Buffer changes and quickly propagates to the VGA port

##### Palette Map
- Size: 64 bytes
- Writable: Yes
- Memory Mapped to Addresses 0xBD0 - 0xC0F
- **Inputs:** WD, ADDR, WE
- **Outputs:** TileSel
<br>Each byte in this map holds an 8-bit color. This is indexed by the 6-bits stored inside the Pixel Buffer. Only the bottom 16 bytes (aka Palette 0) are addressable by the Tile ROM. Sprites on the other hand have access to the other 3 palettes using the Palette0 and Palette1 bits in the Sprite_Math byte. This is a low latency translation unit to convert from a color-code to an 8-bit color. 
