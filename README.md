# Custom-MCU
1. brew install qemu

2. Download https://developer.arm.com/open-source/gnu-toolchain/gnu-rm/downloads
	- Place folder in same directory as this project

3. ./install

## Debug (Reverse firmware to show LED_On)
1. qemu-system-arm -M versatilepb -m 128M -nographic -s -S -kernel main.bin
	- ./gcc-arm/bin/arm-none-eabi-gdb
		- (gdb) target remote localhost:1234
		- (gdb) file main.elf
		- (gdb) break main
		- (gdb) cont

2. Using Objdump && Radare2
	- First lets find the .data
	```
	➜  ./gcc-arm/bin/arm-none-eabi-objdump -s -j .rodata main.elf

	main.elf:     file format elf32-littlearm

	Contents of section .rodata:
 	 1031c 00101f10 4c50435f 4750494f 312d3e44  ....LPC_GPIO1->D
 	 1032c 41544120 3d200000 0a000000 31000000  ATA = ......1...
 	 1033c 4c454420 4f6e0a00 4c454420 4f66660a  LED On..LED Off.
 	 1034c 00
	```
	- r2 -aarm -b16 -d gdb://localhost:1234
		- pd 25 @ 0x1031c
			```
			0x00010328      312d           cmp r5, 0x31                ; '1'
			```
			- This is the value we have to change in order for the LED to do what we
			want
		- `dr r5=0x01` won't work
		- We have to rewrite the ASM
		```
		[0x0001016c]> s 0x00010328
		[0x00010328]> wa cmp r5, 0x30
		Written 2 bytes (cmp r5, 0x30) = wx 302d
		```
		- In gdb, to edit a register, it would be
			 - (gdb) set $r0 = 0x1 @
			 ```
			 0x00010834 <+132>:	mov	r3, r0
			 0x00010838 <+136>:	cmp	r3, #0
			 0x0001083c <+140>:	bne	0x10850 <main+160>
			 ```
		- If you run `pd 25 @ 0x1031c` again, you'll see its compare'd to '0' now.
		- Hit `dc`
			```
			(gdb) disas
			0x00010850 <+160>:	  bl	   0x10438 <mcu_sck_13_high>
   		0x00010854 <+164>:	  ldr	   r0, [pc, #76]	; 0x108a8 <main+248>
	 		=> 0x00010858 <+168>:	bl	   0x10084 <print_uart0>
			(gdb) i r
			r0             0x11098	69784
			(gdb) x/20xg $r0
			0x11098:	0x692033312f4b4353	0x000a484749482073
			0x110a8:	0x2044454c00000031	0x2044454c000a6e4f
			0x110b8:	0x000000000a66664f	0x0000000000000000
			0x110c8:	0x0000000000000000	0x0000000000000000
			0x110d8:	0x0000000000000000	0x0000000000000000
			0x110e8:	0x0000000000000000	0x0000000000000000
			0x110f8:	0x0000000000000000	0x0000000000000000
			0x11108:	0x0000000000000000	0x0000000000000000
			0x11118:	0x0000000000000000	0x0000000000000000
			0x11128:	0x0000000000000000	0x0000000000000000
			(gdb) x/x $r0
			0x11098:	0x692033312f4b4353
			(gdb) x/s $r0
			0x11098:	"SCK/13 is HIGH\n
			```
			- We should see MCU Layout of HIGH && `LED On` in the debug terminal
			```
																		+-----+
			+----[PWR]--------------------| USB |--+
			|                             +-----+  |
			|          GND/RST2 [ ][ ]             |
			|        MOSI2/SCK2 [ ][ ]   A5/SCL[ ] |
			|                              AREF[ ] |
			|                               GND[ ] |
			| [ ]N/C                     SCK/13[H] |
			| [ ]v.ref                  MISO/12[ ] |
			| [ ]RST                    MOSI/11[ ] |
			|               ....                   |
			|                                      |
			| [ ]A4/SDA    RST SCK MISO    TX>1[ ] |
			| [ ]A5/SCL    [ ] [ ] [ ]     RX<0[ ] |
			|              [ ] [ ] [ ]             |
			|               |   |   |             /
			|    LPC13     GND MOSI 5V   ---------
			\                         /          
			-------------------------
			```
