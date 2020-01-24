# VLB Cirrus Logic GD5429 Linear Address Mod

This project replaced the 74F32 quad-OR gate at U22A with a PAL16L8 (or equiv.) at U22.

## Reason
In the manufacturer-built configuration, linear addressing mode is essentially disabled for computers with >12MB RAM.  This is because the GD5429 chip itself only decodes A23-A2.  Address lines higher than A23 (> 16MB) are left to external decoding logic.  
	
## Lessons Learned
- Importance of A23

  VLB address line A23 is key when programming your desired address into the PAL.  The GD542x chip uses register SR7\[7:4\] to determine the base address of the linear mapping; those bits map directly onto A23 through A20.  A23 is also brought out onto the PAL, which allows you to make the decision whether or not the current cycle's address is for the GD5429 or not.  If we instruct the software to use a base address with A23 == 1, we can use A23 to determine if the current address is either (a) below the 16MB mark, or (b) at our desired framebuffer address.  If we did not use A23, the GD5429 will respond to accesses at (LinearFramebufferAddress MOD 16) which will certainly not work.
	
- Details of UADDR#

  The UADDR# input into the GD5429 must be asserted during any memory access destined for the chip.  This includes:
`		A0000h-BFFFFh for the traditional VGA/EGA/CGA space,
		C0000h-C7FFFh for the VGA BIOS, and
		The linear address mapping (I chose 04A00000h)`
	That means we can program the PAL to assert UADDR# when (A\[23-27\] == 0) or (A23 == 1 AND A26 == 1).  This solves the issue of the GD5429 responding to "wrapped-around" physical addresses > 16MB, yet still allowing access to the < 1MB window.
	
- Driver maps 4MB of physical memory into linear window

  A detail found during disassembly of the Windows 3.1 driver (5429.drv).  The LinearAddr= configuration element in the \[CL_WinAccel\] section of the SYSTEM.INI file specifies the base address (in megabytes) of the linear framebuffer.  The driver indeed uses this as the base address in a call to DPMI/ConvertPhysicalToLinear, but specifies a size of 4MB (regardless of the installed memory size).  If you try to map the card at 78MB, the linear framebuffer is probably physically at 80MB, where VLB A23 == 0.  This is a problem because we use A23 to determine if the current cycle address is below 16MB, or above.  If you encounter an issue where you feel like the A22 address line is bad, this may be the issue.

- Other signals

  I am not sure why the ADS# signal is brought to the PAL.
  
  I believe that W/R# is brought to the PAL/'F32 simply to buffer it; the CPU has to drive it to the GD5429 as well as 4 separate '245s.
  
  I am also not certain why BS16#, OEL#, and OEH# are brought out.
	
- PAL programming

  I chose to use A26 (64MB) as the starting point of my testing; it seems that the driver likes to put the linear framebuffer at 74MB (4A0 0000h, or A26+A23+A21).  Also, my motherboard does not connect A27-30, so there was no other option.  The other pins (OEL#, OEH#, and W/R#) are simply assigned to their respective output pins- there was no need to tristate them or perform any other logic.  The BS16# signal is different; it cannot be actively driven (either high or low) during cycles where we are not the VLB target.  I simply assigned that output pin to 0 (asserted), then assigned the output enable of that pin to match the input from the GD5429.

## Steps Taken

  First, I verified that all pins on the 'F32 and the PAL socket were connected per the '94 schematic.
I removed the 0 ohm shunts at R3, R4, and R14. They bridged BS16#, OEL#, and OEH#.
I removed the 470 ohm pull-up resistor at R45.  It is for the VLB BS16# signal:  I believe that the designer looked at the Cirrus Logic schematic and thought that the 1K pull-up here and at R40 were connected to the same line.  He took the liberty of replacing 2 "parallel" resistors with one.
I replaced the motherboard-facing resistor with a 1K, and the GD5429-facing with a 2.2K.
I cut the 'F32 off the board, and removed the remaining pin stubs.
Then I soldered a 20-pin DIP socket to the board, and verified all connections.
Once testing was completed, the ATF16V8 was inserted and the project was completed.
