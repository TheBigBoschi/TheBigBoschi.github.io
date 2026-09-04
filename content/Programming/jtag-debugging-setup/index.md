---
date: 2026-06-18
draft: false
title: "Board assembly and JTAG debugging"
---

After sending out the board off for fabrication I ordered the components, and just as I received them I was hearing all the talk about the 3 euro for item tax, which I barely avoided. Lucky me I guess!

## Assembly

The assembly was quite straightforward, no major problems were found and so far everything is working as expected. The board is able to be powered on from the battery and the solar charger works as expected.

The soldering was done mostly with a soldering iron, with the ESP32 itself soldered using a hot air station, with touchups done again with a soldering iron. 

{{< carousel aspectRatio="1-1" images="{Assembly Photos/closeup.jpg,Assembly Photos/reflow.jpg,Assembly Photos/top.jpg,Assembly Photos/back.jpg}" >}}

The problems began as I was trying to load the software and launch the debugger.
Initially the silliest of the mistakes happened: I swapped the RX and TX lines. This became apparent after probing with the oscilloscope the lines, and realizing that the TX pin of the ESP was being driven by the programmer.
Crimping a new 6 pins cable with the lines crossed solved the issue. 

Now I was able to program the thing, but not reliably.

After some more probing around with a multimeter I realized that the battery holder holding my 18650 was quite tight, making it so that when inserting the battery, if not done carefully, the battery would barely contact the positive contact, making the power connection unreliable.

Pressing the battery fully in made the programming work reliably. To have a quick indication of the 3.3V rail (and everything before it) working correctly, I added a led to the rail. This fixed all my frustations from that point forward.  

## JTAG debugging

It's important to select the right target to be debugged. To do so, in the command bar (the one on the top of the screen in VScode) type ">ESP-IDF: Set Espressif Device Target" and select your esp model (in my case an ESP32), and after that select "ESP32 chip (via ESP-PROG)" or else it won't work.


After sorting out these nuances i was finally in business, being able to program and debug the microcontroller.
I am not sure what I was expecting, but it's basically as debugging on a PC, being able to do instruction by instruction execution, and being able to view registers and assembly code on the fly, with the added inconvenience of having to manage two cores, and sometimes this can cause wifi disceonnection if it's not managed correctly.

![alt text](<debugging.png>)

What left me perplexed at first is that the com port numbers printed for the UART and the JTAG did not line up properly, it's probably relatd to the fact that they talk about COMs port on the silkscreen, while im seeing them as devttyUSB ports.

## First tests

I'm slowly developing the software, for now I am more intrested in checking all the GPIOs work as intended, to switch the 5V supply and the 3.3V aux one (and the voltage divider to gauge the battery voltage too).

Guess what? GPIO 35 (5V enable) is input only! I have to use some other spare pin to switch the 5V line. This will get corrected in the revision 2 of the PCB, for now I'm using 3.3V EN to switch the 5V regulator on and off.

At least GPIO 22 (3.3V enable) works as expected, same for the msofet used to switch the power. With a 10K resistor as load the voltage drop to 0.5V, I assume it's related to some capacitance on the line, and it should not be a problem, if this consumes too much power I will have to revise the resistor network to bias the transistor (or change transistor type interely).

The Vsense switch (the transistor array used to switch power to the voltage divider to measure the battery voltage) work as expected instead, with its voltage dropping to 0 when not enabled.

Success at last!
