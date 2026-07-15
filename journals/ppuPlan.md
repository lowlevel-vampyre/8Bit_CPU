# Graphics Outline

The CPU is only half of the battle for this project. The other half is the graphics. For this project, a playable game is important,
meaning we have to be able to control a display. This is my idea, which is subject to change, about how we will go about this.

The first thing to note is what kind of communication we plan to use. Because of simplicity, and the large amount of present documentation,
the VGA protocol is what we are targeting. This makes it easy for our GPU to output pixel data on a timer to the display (though I will say,
we are not using a framebuffer). This will be controlled by two timers in the GPU, one for the horizontal pixel, and the other for the
vertical pixel. The data to print for each pixel will be supplied by a combination of these modules: a 960-byte Tile Map (32 * 30 tiles), 
the 4-sprite OAM (Object Attribute Memory), a 2560-byte Image Buffer (80 images at 16 bytes per image), and finally a 4-Byte Color Buffer

These are architected like so:

---
##### Tile Map
- 1200 Bytes
- System RAM (0x800 - CAF)
- Each byte holds the ID of an 8x8 tile from the Tile Buffer, and each byte corresponds to a tile on the screen. The display is split up
  into 40 horizontal tiles, and 30 vertical ones. This is purely to match the VGA resolution of 640x480. Using the 8-bits per tile line,
  the final resolution works out to 320x240, which is half the resolution, making it easy to matchup the pixel. The PPU uses this to
  determine what pixels it should write to the screen. Since the tiles directly map to the display, the H and V counters control the
  addressing to the Image Buffer. The CPU will most likely fill this once, and make minor changes if there are any updates (like a new
  level or something else)
---
##### Object Attribute Memory (OAM)
- 16 bytes
- System RAM (0xCB0- CBF)
- This is a special chunk of memory that the PPU uses to determine if a sprite occupies this tile. The 16 bytes represent 4 sprites, each
  with 4 bytes denoting x-pos, y-pos, unused, and sprite ID. The unused spot is possibly for rotations, transforms, or other math. I am 
  not sure yet. The Sprite ID corresponds to a address in the Image Buffer. It does not have enough space for all available sprites, so
  while this computer supports up to 16 sprites per game, there can only be 4 sprites on screen at a time. This is where the CPU keeps
  track of player and mob data
---
##### Image Buffer
- 2560 Bytes
- Private Programmable ROM (Not CPU Addressable)
- This buffer stores the pixel data for each sprite and tile. It is split up where the first 2048 bytes are for tiles, and the remaining
  512 bytes are for sprites. Each image is 8x8 pixels, and each pixel uses 4-bits for color, so each byte in this memory file corresponds 
  to 2 pixels. This allows a maximum 64 tiles and 16 sprites, at 32 Bytes per image. The 4-bit color is mapped to the 16 colors in the 
  color buffer, allowing minor effects, like color shifting or flashing.
---
##### Color Buffer
- 16 bytes
- System RAM (0xCC0- CCF)
- This is a special chunk of memory that the PPU uses to determine the color to display. Here, full 8-bit colors are used. Theoretically, this
  implementation allows 256 colors... just only using 16 at a time XD. However, having this file be CPU Addressable means cool effects could
  be implemented, like flashing screen when a game is won, color-shifts for "night mode" or "super-duper hard mode", or whatever else. 
---


The way I plan this to work is this. At the end of a frame during V-Blank, the PPU copies data from some of CPU addressable portions of memory,
which is only the 16-bytes of OAM and the 16-bytes for colors, into super fast registers. This means the any updates to memory that the PPU
accesses will be see during the next frame.

For each screen tile, it checks the Tile Map for the particular tile it is currently on. Since each tile is sorted into an 8x8 grid, and 
each pixel is 4-bits of color, we will have to split each byte in half. For each screen pixel, the pixel address only ticks every other
clock cycle. And for each pixel in, if the horizontal counter is odd, we grab the high 4-bits. If it is even, we grab the low 4-bits. The
pixel data is output to the Pixel Comparator, which checks if the pixel should be shown, or if a sprite pixel should be shown instead. Then
that data goes into the address port of the color register, which outputs 8-bit data to the display, giving a pixel!

For future reference, sprite pixels are calculated differently than regular tiles. Because each tile image is mapped to a particular tile on
the screen, we can easily use the bottom 8-bits of our pixel location to index the x & y bits for each 8x8 image. However, sprites are not
constrained in this way, and require more work to interpret which pixels to display. When the pixel comparator computes whether a pixel is on
a sprite, it does a subtraction calculation: Pixel Location - Sprite Location (the sprite location in this case is the pixel at the top left 
of the sprites image). If this difference is inclusively between 0 and 7, we have an overlap! But instead of throwing this data away, we hold 
the difference for x & y, and pass that directly into the XPixel and YPixel values in our Sprite Buffer. Let me explain further why this is the
case. Say our location pixel on our sprite overlapped with the current pixel. In that case, the difference is 0, so we pass zero into our buffer, 
since what we want to do is print that top left pixel. If instead the x difference is 0 but the y difference is 7, we want to print the bottom 
left pixel. You can now see how the difference corresponds to each pixel.

Another thing you might be thinking about is how exactly to index each pixel individually. Since we are storing two pixels in a byte, we can't
actually do that, we would have to split them apart after the fact. But still, we have 4 bytes per line: how do we know which bytes to select?
Well, my idea is that since each line is 8 pixels, the binary number system is on our side. If we already know which x pixel and each y pixel
on a image, we can easily concatenate these to make a full index. Want to access the 8th pixel (x = 0b111) on the 3rd line (y = 0b010)? Well, 
we take the first take the upper 2 bits of the x-value (0b11 = '3') and the full y-value (0b010 = 2), and we put those values together 
(0b01011 = '11') to get the 12th byte, which contains the 23rd and 24th pixel. Then, we use the bottom bit of our x-value (0b1 = 1) to select
between the upper and lower 4-bits of the byte. Hooray, we finally have a usable pixel!

Once we have the 4-bits for color, we will put this into a temporary register to index the color registers on the next pixel. Since we are not
actually calculating the current pixel, but instead the next one, we need to hold that pixel for the duration of the pixel clock. So on each 
clock cycle, the temporary register gets the pixel data we calculated, and uses that to index the color register, which outputs to the display.

The CPU has info about where the PPU is at via memory that the CPU can read from. Two bytes report where the PPU is at. One is a byte
representing the current horizontal tile index, and one representing the vertical counter (excluding H-Blank and V-Blank). Another is 
a pseudo-status register, that holds if the PPU is in V-Blank, H-Blank, and also a bit to tell if the frame is odd or even. 
