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
<img src="./photos/2nd%20rev%20macro.jpg" alt="1st rev" style="width:45%"/>
</p>