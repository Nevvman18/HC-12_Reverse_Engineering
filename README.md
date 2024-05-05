# HC-12 - Reverse Engineering
HC-12 - one of the many good radio modules you can find on the market. *And the most mysterious*.

## Introduction
For one of my projects I bought a pair of them. Unfortunately, after about half a year, one of them either **went deaf or silent**, due to a broken radio IC.

I ended up buying another pair. Instead of importing from China, I got them from my local store, and **they were fake**! Bad reception, no config mode - these were the problems.<br>
After returning them, I received another 3 pieces from the same store in China. And they work *perfectly*.<br>
BUT, the new ones **didn't communicate with the old** ones!

## How could they be so different?
I started looking at them closely. Same **Si4463** radio IC, same PCB but a **different MCU!**<br>

### First? revision
Runs on a **HK32F030MF4P6** MCU. It's *like an STM32* family, but **cheaper** and more ***sketchy***.<br>
Specs: 32-bit Cortex-M0, 32MHz, 16KB flash, 2KB RAM (actually 4!) <br>
Looks powerful for the module, but unfortunately it isn't the most user-friendly (explained later on).

### Second revision
Here it gets better, because **everything is the same**, apart from the MCU.<br>
This time, it's **STM8S003F3P6**. Manufactured by STMicroeletronics, so it is **documented well** and it seems ***nicer to play with***.<br>
Specs: 8-bit 16MHz uC, 8KB flash, 1kB RAM.<br>

### Side-by-side comparison
<p align="center">
<img src="./photos/1st%20rev%20macro.jpg" alt="1st rev" style="width:45%"/>
<img src="./photos/2nd%20rev%20macro.jpg" alt="2nd rev" style="width:45%"/>
</p>

## Firmware extraction
### First revision
Firstly, I checked it's version with the built-in AT command `AT+VER`, which got me a response of **`HC-12_V2.6`**. The weird thing is that some modules respond with 'HC-12 \[version]', and some with 'www.hc01.com \[version]'.<br>

The main MCU is a HK32 family uC. It is programmed using **SWD** interface. After a bit of research, I stumbled upon some [blog](https://nerdralph.blogspot.com/2020/12/trying-to-test-ten-cent-tiny-arm-m0-mcu.html) [posts](https://nerdralph.blogspot.com/2021/01/trying-to-test-ten-cent-tiny-arm-m0-mcu.html) featuring the programming adventure.<br>
For programming it I wanted to use my ST-Link V2 probe. It turns out that during connection there occurs some sort of verification process, which doesn't allow me to read the firmware.<br>

I carefully attached the probe to the HC-12 (it has 2 test points on the back side of the board) and started debugging.<br>

Using pyOCD I successfully connected to the core of the HK32:
```console
(.venv) monkey@computer:~$ pyocd cmd -v -t stm32f051
0000530 I Target type is stm32f051 [board]
0000539 I DP IDR = 0x0bb11477 (v1 MINDP rev0) [dap]
0000584 I AHB-AP#0 IDR = 0x04770021 (AHB-AP var2 rev0) [discovery]
0000587 I AHB-AP#0 Class 0x1 ROM table #0 @ 0xe00ff000 (designer=555 part=600) [rom_table]
0000589 I [0]<e000e000:SCS v6-M class=14 designer=43b:Arm part=008> [rom_table]
0000590 I [1]<e0001000:DWT v6-M class=14 designer=43b:Arm part=00a> [rom_table]
0000591 I [2]<e0002000:BPU v6-M class=14 designer=43b:Arm part=00b> [rom_table]
0000595 I CPU core #0: Cortex-M0 r0p0, v6.0-M architecture [cortex_m]
0000597 I 2 hardware watchpoints [dwt]
0000599 I 4 hardware breakpoints, 0 literal comparators [fpb]
Connected to STM32F051 [Lockup]: B55B5A1A00000000392CF301
pyocd> rw 0x0 64
Transfer failed: Memory transfer fault (read) @ 0x00000000-0x0000007f
pyocd> unlock
0065045 W T bit in XPSR is invalid; the vector table may be invalid or corrupt [cortex_m]
Error: target was not halted as expected after calling flash algorithm routine (IPSR=3)
pyocd> reg flash
Flash.ACR @ 40022000 = 00000000
Flash.KEYR @ 40022004 = 00000000
Flash.OPTKEYR @ 40022008 = 00000000
Flash.SR @ 4002200c = 00000000
Flash.CR @ 40022010 = 00000080
Flash.AR @ 40022014 = 00000000
Flash.OBR @ 4002201c = fffffffa
Flash.WRPR @ 40022020 = ffffffff
pyocd> wr Flash.KEYR 0x45670123
writing 0x45670123 to 0x40022004:32 (KEYR)
pyocd> wr Flash.KEYR 0xCDEF89AB
writing 0xcdef89ab to 0x40022004:32 (KEYR)
pyocd> rw 0x0 64
Transfer failed: STLink error (18): AP error
pyocd>
```

I was able to read status, reset, halt, change registers, but any operations of writing and **flash reading were unsuccessful**. The issue has to be with the ST-Link. Segger J-Link should work, but I don't own one.<br>

I wanted to somehow extract the firmware. I saw [this writeup](https://github.com/rumpeltux/hc12) (and also other things [there](https://itooktheredpill.irgendwo.org/2020/hc12-hacking/)) on someone reversing same HC-12 modules. *rumpeltux* found out, that with sending various bytes over serial port he was able to get the firmware from the device. Tried it with my version, but it didn't work, throwing assignment errors.<br>

**My firmware extraction attempt from 1st module revision was unsuccessful.**

### Second revision
This time it **went better**. STM8, being supported by my debuggers, is programmed by **SWIM** interface. This module responds to `AT+VER` with **`www.hc01.com HC-12 v2.6`**.<br>
I assumed that it won't go so easily, because in the previously mentioned post, *rumpeltux* had to power glitch the uC to be able to extract the firmware.<br>
I connected the STLink with the STM8 using 2 test points on the back:
<p align="center">
<img src="./photos/2nd%20rev%20swim%20interface.jpg" alt="2nd rev test points" style="width:30%"/>
<img src="./photos/2nd%20rev%20mcu%20macro.jpg" alt="2nd rev STM8 uC" style="width:30%"/>
</p><br>

For the firmware extraction attempt, I used the official ST Visual Programmer.<br>
First attempt - `Unable to read bytes [...]`. I thought that again it was the Read-Out Protection byte. But wait, clicked the button second time and... **I was able to read the whole firmware!** For known reasons I won't release the original firmware, but I will analyse it later on.<br>
On the **OPTION BYTE** page the first register was set like this: **`ROP --- Read Out Protection OFF`**. I don't know if I was lucky with this batch. Maybe this version isn't properly secured? I didn't check the other modules, because of the need to solder the connections. If anyone wants, I can check that.


## Firmware analysis
Note that I only have the `v2.6` firmware.<br>

### PROGRAM MEMORY
Uploading the main firmware to [binvis.io](https://binvis.io/) reveals that only the first 200 bytes contain interesting characters in a mostly human-readable format.<br>
Some interesting parts: <br>
* At address **0x00008080** there is `HC-12_V2.3` string. But the module calls that it is v2.6, like below:
* At address **0x000080F1** there is `HC-12 v2.6...www.hc01.com` which is a string shown on the UART output.
* At address **0x0000808E** there is `20210319` which looks like a compilation timestamp.
* At address **0x000081F8** there is `How are you..Long time no see`. I don't know what that could be, it never shows up during operation.
* There are also some bytes like `OK` `ERROR` `OK+B00..`.

The rest of the *PROGRAM MEMORY* is a compiled machine code. I want to decompile it in Ghidra? in future, because there seem to be plug-ins for disassembling STM8 firmware files. The bad news is that *rumpeltux* decompiled his module's `v2.4` firmware and said that re-using it is *not a viable option*.

### DATA MEMORY
This is the EEPROM memory that contains program variables, like transmitter mode, frequency, baud rate. I observed how they change with trying different settings.<br>
For example, byte **0x00004001** stores the **radio channel**, `0x02` being channel 1 and `0xFE` being channel 127.<br>
I manually set the channel byte to `0xFF`, to which the module responded with `OK+RC254`. I can also set this channel via AT command, though it exceeds the range of 1-127 mentioned in the datasheet. Weird? Setting byte to `0x02` occured in a `OK+RC127`, maybe the decimal values of channel bytes are divided by 2 to get it in a configuration number. Will check communication on these channels later.

## Final thoughts
These radio modules seem like a mystery. Tons of versions, clones and unconfirmed information can be found. In future I will try to decompile the firmware and analyse the module more.<br>
There seem to be custom firmware projects on github, like these:
* [*rumpeltux's* custom firmware](https://github.com/rumpeltux/hc12fw) - I will check later
* [AX.25 packet radio firmware](https://github.com/al177/hc12pj) - compiled it quickly and it works (at least the serial port)

## Useful links
* [*rumpeltux's* blog](https://itooktheredpill.irgendwo.org/2020/hc12-hacking/) post about HC-12 reverse engineering
* [stm8flash](https://github.com/vdudouyt/stm8flash)
* [eevblog discussion](https://www.eevblog.com/forum/microcontrollers/$0-25-hk32f030m-(cortex-m0-32mhz-16kb-2kb)/) about the HK32F030M uC
* [Ralph Doncaster's](https://nerdralph.blogspot.com/2020/12/trying-to-test-ten-cent-tiny-arm-m0-mcu.html) HK32F030M programming attempt
* [Reprogramming a HC-11 CC1101 433MHz Wireless Transceiver Module](https://mvdlande.wordpress.com/2016/09/03/reprogramming-a-hc-11-cc1101-433mhz-wireless-transceiver-module/)
* [Hackaday post](https://hackaday.com/2018/05/05/fail-of-the-week-never-assume-all-crystals-are-born-equal/) about differences in HC-12 batches
* [Hackaday post](https://hackaday.io/project/27319-432-434mhz-range-spectrum-analyzer-based-on-hc-12) about DIY HC-12 radio spectrum analyzer