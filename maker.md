# Maker Portfolio - Josh Piety
## Starblazer II - 3D Video Game for DOS & Windows
I was one of two lead developers on Starblazer II, a 3D space shooter video game inspired by games like Star Fox. Since the game was designed to run on very old hardware, and through DOS, the game engine had to be custom created.

<TODO\>

## SuperNova 8000 - Custom 8-bit computer constructed on breadboard and PCB

The SuperNova 8000 (SN8K) is an 8-bit computer, running on a W65C02S-14 processor (a variant of the 6502), alongside 32K of SRAM and 8K of programmable EEPROM. I designed the system architecture myself, as well as assembling it.

For easy prototyping, the first version was built on a breadboard. It incorporated these components:
* W65C02S-14 CPU
* W65C22 VIA (versatile interface adapter) used to communicate with peripherals
* 62256 SRAM chip, 32K
* 28C64B EEPROM chip, used for storing program code
* System clock at 1 MHz
* Discrete logic ICs to manage memory mapping

In order to program the EEPROM, as opposed to purchasing an expensive dedicated programmer, I designed one using an Arduino:

image

This allowed me to assemble 6502 assembly using `ca65`, flash it to the EEPROM and have it executed on the computer. In order to construct the device, I had to reference [datasheets](https://www.westerndesigncenter.com/wdc/documentation/w65c02s.pdf) on each of the ICs - for instance, this:

image

It took a few days to construct and debug the breadboard prototype, but it eventually worked:

image

The 8-bit register (labeled as "Flip flops") can be accessed by writes to the upper half of memory (addresses `$8000-$FFFF`, where the EEPROM is accessed through reads), and in this case holds the value `$55` that was written through code.

Real computation is possible with this design, too. I wrote a program that checks if numbers are prime:
```
.segment "ZEROPAGE"
; zp variables here
temp1: .res 1
temp2: .res 1
cur_prime: .res 1
.segment "RAM"
; ram variables here

.segment "ROM"
checkDivisible:
    ; checks if x is divisible by a
    ; returns ff if true and 00 if false
    ; preserves x and y
    ; clobbers a and flags, temp1 and temp2
    stx temp1
    sta temp2
checkDivisible_loop:
    clc
    adc temp2
    cmp temp1
    beq checkDivisible_yes
    bcc checkDivisible_loop
    lda #00
    rts
checkDivisible_yes:
    lda #ff
    rts
isPrime:
    ; checks if x is prime
    ; returns ff if true and 00 if false
    ; preserves x and y
    ; clobbers a and flags, temp1 and temp2
    txa
    and #01
    beq isPrime_odd
isPrime_composite:
    lda #00
    rts
isPrime_odd:
    lda #03
isPrime_loop:
    pha
    jsr checkDivisible
    bne isPrime_composite
    pla
    clc
    adc #02
    stx temp1
    cmp temp1
    bcc isPrime_loop
    lda #ff
    rts
findNextPrime:
    ; finds the next prime after cur_prime and puts it in cur_prime
    ; preserves y
    ; clobbers a, x and flags, temp1 and temp2
findNextPrime_loop:
    inc cur_prime
    ldx cur_prime
    jsr isPrime
    bne findNextPrime_finished
    jmp findNextPrime_loop
findNextPrime_finished:
    rts
nmi:
    jsr findNextPrime
    lda cur_prime
    sta ffff
    rti

irq:
    rti

reset:
    sei
    cld
    ldx #ff
    txs
    lda #02
    sta cur_prime
    sta ffff
endless:
    jmp endless

.segment "VECTORS"
.word nmi
.word reset
.word irq
```

A pushbutton was wired to the NMI pin (pin 6) of the 65C02 microprocessor, and as I signaled it through that button, the program worked as intended - the LED array stepped through all prime numbers between 0 and 255. 

However, the system was prone to wires coming out of the breadboard, so I decided to move future development to a printed circuit board. I used the KiCad EDA tool to design the schematic for the system and the physical PCB design:

image

image

After checking the design, I put in the PCB for fabrication at the fabrication service [OSH Park](https://oshpark.com/). I soldered the components used in the breadboard design into the PCB.

image

The design mostly worked; however, I had inadvertently mapped the 16 registers used by the VIA into space also mapped by the SRAM chip. If the system read from one of those addresses, it would cause a bus conflict and possibly damage hardware due to overcurrent. This necessitated a second revision (in which I also repositioned some of the connectors):

image

\<TODO\>


## Brimstone: Game engine for NES for 32KB development

The NES was a video game system that first released in 1983 in Japan. The first games developed for it were limited to 32 kilobytes of code space. I developed a game engine that ran under this same limitation, both to gain an appreciation for the history of computing and learn how to work under constrained environments, like those present in embedded systems. 

The most pressing issue is reducing the space footprint of game level data. If a standard game has 16 levels or stages, each level contains 8 "screens" worth of gameplay, and each screen is 16x16 "tiles", each one represented by a single byte, the game will use `16*8*16*16=32768` bytes for its level data, which leaves no space at all for program code! To get around this, I developed a custom compression algorithm.

Although in a game each level will span many "screens", game objects will:
* Often exist in very similar forms (rectangular blocks, or repeated patterns)
* Exist near each other (locality of reference) and slowly span the available space, as opposed to jumping around

Instead of using 8 bits to store X and Y coordinates for a game object, it is thus feasible to use 4 bits to store each as a signed displacement (-8 to 7) from the last object - a "command pointer" moving around the level keeps track of these positions and is updated by each game object. This means that the X and Y can be packed into a single "position" byte for each game object:

```
Position
XXXXYYYY
||||++++- signed Y offset
++++----- signed X offset
```

Each game object exists as one of the following types, including a two bit header and a six bit value:
```
Header byte
TTAAAAAA
||++++++- value/argument
++------- type field
```
The four/five types are:
* Type `00`: point. Some objects such as triggers only occupy a single point in space. The argument specifies what kind of object is placed here; there is a position byte following the header.
* Type `01a`: dynamic rectangle. This is a "catch-all" for more unique game objects that don't compress well as a series of rectangles, but take up a rectangular area. The argument field is left as zero, and is followed by a position byte and a size byte (with 4 bits apiece to width and height). After this is the content of the rectangular area: one byte per tile.
* Type `01b`: quick movement. Since displacements are from -8 to 7, moving, for instance, 15 tiles in one step between objects is impossible. If the argument field of a type `01` object isn't left as zero, each 3-bit half is interpreted as a signed displacement that's multiplied by 16 before being used to offset the "command pointer".
* Type `10`: static rectangle. Many game objects, like walls, background and floors, are comprised of the same tile repeated in a rectangular area. The argument field specified what kind of tile, and is followed by a position byte and a size byte.
* Type `11`: pattern playback. Often, the same object will show up many times. A list of "patterns", each being a compressed command sequence like this (essentially a subroutine), is included with the level data. The argument field specifies which pattern should be played back. It's followed by a position byte and a size byte - the width and height are interpreted as the number of times to loop the pattern in each axis.

In order to create the levels, I developed a GUI tool that supported this format natively:

image

image

With that done, I looked to developing other components of the game:
* Since the level data was stored compressed, decompression must happen at runtime on a very slow system, which also necessitated a very optimized decompression algorithm.
* The majority of the game code was written in [Millfork](https://github.com/KarolS/millfork), a new "middle-level" language specifically designed for this application. It was much more convenient than assembly language without sacrificing control over low-level parameters like memory layout, which on a constrained system is crucial. At the time I started this project, there were no good C compiler toolchains for this architecture. Components like decompression and frequently used macros were written in assembly.
* Video memory (VRAM) can only be accessed during the short time at the beginning and end of each frame when the video chip is not using it to draw to the screen ("vblank"). This meant I had to precompute the "updates" that needed to be made to the screen as the player moved through a level and uncovered new objects. As soon as this window of time started, the updates could be put into place.
* Since the hardware lacked any floating-point support, fixed point numbers of varying precision were used to implement physics and object collision detection.
* To save on processing power, a distinction in code was made between dynamic game entities that move and perform actions (termed "enemies") and static game objects that display a visual for a period of time (termed "effects"). That way, movement and collision detection routines aren't executed for these static objects.

The final engine uses about 8 kilobytes of space. This leaves the remaining 24 kilobytes for a prospective game and its data and objects. Of the available 2 kilobytes of RAM in the system, this engine uses about half of it, once again leaving the remainder for any game that leverages the engine.

images

## SNES Hardware Repair Project

After I received a non-functional Super Nintendo Entertainment System (video game console) from a friend, I set out to use my technical knowledge to repair it. The workings of the system are extensively documented, as are the games that run on it, so this was feasible.

When I initially tried to load a game using it - in this case, the 1994 game *Donkey Kong Country* - I was met with only a black screen, which could imply a catastrophic failure that can't be recovered. For instance, fried components might not exist on the market anymore, and transistor level damage is completely impossible to repair. However, given that the system had worked previously, I wanted to look further into it. Sure enough, the related *Donkey Kong Country 2* gave me different results:

image

Clearly, the CPU, RAM and graphics hardware have at least minimal functionality, and no essential components are totally destroyed. The conditions under which this "anti-piracy" screen is displayed can be found from a [disassembly of the game](https://github.com/p4plus2/DKC2-disassembly/tree/master): 
```
	REP #$20                    ;$808436   |\ Test that SRAM is present
	LDA.l sram_base	            ;$808438   | |
	INC A			            ;$80843C   | |
	STA.l sram_base	            ;$80843D   | |
	CMP.l sram_base				;$808441   | |
	BNE .prepare_anti_piracy	;$808445   |/ Otherwise trigger anti piracy
	DEC A					    ;$808447   |\ Restore byte modified
	STA.l sram_base				;$808448   |/
```
One of these conditions is that the save data RAM chip (SRAM) is present. The save data memory area is located at memory address `$B06000`, while the code above is located around memory address `$808436`. Therefore, a hardware fault in the upper address pins A22-A20 would allow code at `$808436` to execute, while accesses to memory at `$B06000` fail. Testing this hypothesis, I removed and cleaned the [cartridge connector](https://snes.nesdev.org/wiki/Cartridge_connector) (the adapter that the game uses to communicate with the console), and used a continuity tester to confirm that all pins were functional:

image

Unfortunately, this did not solve the problem. I attempted to use electrical tape to "mask" off pins on the connector to determine if a fault in any of these pins (here simulated using the electrical tape) resulted in the same effect as I observed, but it became clear that the issue was not a simple continuity issue, but instead something more difficult to resolve.

The next step was to observe the workings of the system at a CPU level and note any discrepancies. To this end, I purchased a logic analyzer - essentially an oscilloscope specifically intended for measuring 5V logic signals. Wires connected to the logic analyzer could be held in place by pressure from the connector, allowing me to "hook in" to the communication between the game and console:

image

I could now record the bytes that were being read, written and executed by the CPU (more specifically, a 65816 running at 3.58MHz) by logging the contents of the address bus. The logic analyzer was limited to eight digital channels, while the system had 24 address pins, 8 data pins and several other control signals that were necessary for discerning the operation of the CPU - clock and read/write signals, for instance. Thus, it was necessary to record data over several runs and match it up:

image

image

I determined that at TODO TIME after the RESET button was released, a given code sequence reliably executes, and I began my analysis there.

image

\<TODO\>
