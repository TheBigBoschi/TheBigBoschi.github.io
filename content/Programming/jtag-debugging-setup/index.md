---
date: 2026-06-18
draft: false
title: "Board assembly and JTAG debugging"
---

After sending the board off for fabrication I ordered the components, just as I received them I started hearing all the talk about the 3 euro for item tax, which I barely avoided. Lucky me I guess!

## Assembly

The assembly was quite straightforward, no major problems were found and so far everything is working as expected. The board powers on from the battery and the solar charger works as expected.

The assembly was mostly done with a soldering iron, with the ESP32 itself soldered using a hot air station, with touchups done again with a soldering iron. 

{{< carousel aspectRatio="1-1" images="{Assembly Photos/closeup.jpg,Assembly Photos/reflow.jpg,Assembly Photos/top.jpg,Assembly Photos/back.jpg}" >}}

The problems began when I tried to load the software and launch the debugger.
Initially the silliest of the mistakes happened: I swapped the RX and TX lines. This became apparent after probing with the oscilloscope the lines, and realizing that the TX pin of the ESP was being driven by the programmer.
Crimping a new 6 pins connector with the TX and RX lines crossed solved the issue. 

From then on I was able to program the thing, but not reliably.

After probing around with a multimeter I realized that the battery holder holding my 18650 was quite tight, making it so that when inserting the battery if not done properly the battery would barely make an electrical connection, making the connection unreliable.

Fully inserting the battery made the programming work reliably. To have a quick indication of the 3.3V rail (and everything before it) working correctly, I added a led to the main voltage rail. This fixed all my frustations from that point forward.  

## JTAG debugging setup

Keep in mind that a specific entry in the udev rules have to be added before anything else.
As a start, create the file:
```bash
sudo nano /etc/udev/rules.d/99-openocd.rules
```
Then populate it with the following lines:
```bash
# FTDI devices (common debuggers like ESP-Prog)
SUBSYSTEM=="usb", ATTRS{idVendor}=="0403", MODE="0666", GROUP="plugdev"

# Espressif built-in USB-JTAG / USB-Serial controllers
SUBSYSTEM=="usb", ATTRS{idVendor}=="303a", MODE="0666", GROUP="plugdev"
```
then add your user to the plugdev group, and reload the udev subsystem. Sometimes rebooting the system is necessary for the modification to take effect.
```bash
sudo usermod -aG plugdev $USER
sudo udevadm control --reload-rules
sudo udevadm trigger
```
replug the board and the debugger and you should be good to go! Keep in mind that you may still be able to program it (using FTDI UART), but you wont be able to debug it (Using FTDI), or at least that was my experience before updating the udev rules. 

It's important to select the right target to be debugged. To do so, in the command bar (the one on the top of the screen in VScode) type ">ESP-IDF: Set Espressif Device Target" and select your esp model (in my case an ESP32), and after that select "ESP32 chip (via ESP-PROG)" or else it won't work. 


After sorting out these nuances I was finally in business, being able to program and debug the microcontroller.
I am not sure what I was expecting, but it's basically as debugging on a PC, being able to do instruction by instruction execution, and being able to view registers contents and assembly code on the fly, with the added inconvenience of having to manage two cores, and sometimes this can cause wifi disconnection if it's not managed correctly.

![alt text](<debugging.png>)

What left me perplexed at first is that the com port numbers printed for the UART and the JTAG did not line up properly with the ones listed in VScode, it's probably relatd to the fact that they talk about COMs port on the silkscreen, while im seeing them as devttyUSB ports in VScode.

## First tests

I'm slowly developing the software, for now I am more intrested in checking all the GPIOs work as intended, to switch the 5V supply and the 3.3V aux one (and the voltage divider to gauge the battery voltage too).

Guess what? GPIO 35 (what I wired as 5V enable) is an input only pin! I have to use some other spare pin to switch the 5V line. This will get corrected in the revision 2 of the PCB, for now I'm using 3.3V EN to switch the 5V regulator on and off.

At least GPIO 22 (3.3V enable) works as expected, same for the msofet used to switch the power. With the 3.3V rail switched off and a 10K resistor as a load the voltage on the rail is still 0.5V, I assume it's related to some leaks on the line, and it should not be a problem. if this consumes too much power I will have to revise the resistor network to bias the transistor (or change transistor type interely).

The Vsense switch (the transistor array used to switch power to the voltage divider to measure the battery voltage) work as expected instead, with its voltage dropping to 0 when not enabled.

I briefly checked the MPPT charge controller and it appeared to be working as expected, same thing for both the step up converter, but I'm a bit worried about it's behavior when the wifi gets turned on, and the current consumption will rapidly switch from basically 0 to 500mA.

For now I consider this a success.

The next step? finish the software and try out [the ESP Trace component!](https://developer.espressif.com/blog/2026/06/introducing-esp-trace-component/)

## References

[ESP-IDF JTAG debugging overview](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/jtag-debugging/index.html)
[ESP-IDF VSCode debugger setup](https://docs.espressif.com/projects/vscode-esp-idf-extension/en/latest/debugproject.html)