---
title: "Nixie Clock: Hardware design"
date: 2022-10-18
description: "a nixie based clock"
thumbnail: "/featured.jpeg"
tags: ["example", "tag"]
series: ["Nixie clock"]
series_order: 1
---



<img src="casio.jpg" width="30%"  style="float:right; border-radius: 20px; padding: 1% 2%;"/>

As an electronics enthusiast, I’ve always been fascinated by the concept of timekeeping. It’s one of those challenges humanity has truly mastered. Take the iconic Casio F-91W, for example: it’s inexpensive (around 20€), has been around for decades, and can still maintain an accuracy of about ±15 seconds per month in normal conditions.
So when I found a few Nixie tubes in my scrap bin one day, I naturally started thinking about building a tabletop clock.
</div>
<br>

Nixie
========

<img src="nixie.jpeg" width="25%"  style="float:left; border-radius: 20px; padding: 1% 2%;"/>

Nixie tubes are relics of the 1960s. Inside each one, thin number-shaped wires are stacked on top of each other and enclosed in a glass bulb filled with a low-pressure noble gas (and sometimes a small amount of mercury to increase their lifespan).
A Nixie tube has one anode and ten cathodes—one for each digit. When you apply about 180 V through an impedance of roughly 20 kΩ between the anode and a chosen cathode, the corresponding digit glows. It feels like magic!
(Side note: Nixies don’t work like incandescent bulbs. They operate using the [glow discharge](https://en.wikipedia.org/wiki/Glow_discharge) effect).  

<br>
<br>
<br>

HV power supply
========

Since Nixie tubes operate at such high voltages, it's not possible to simply power them directly from a wall-wart supply, it would be impractical and unsafe for the end user.  

Instead, a standard 12 V barrel-jack wall-wart power supply was chosen. It’s inexpensive, easy to find, and doesn’t raise any safety concerns.

The schematic below shows the supply unit chosen to power the high-voltage Nixie supply. It was selected for its simplicity, reliability, and the fact that its current-mode PWM control makes it easy to set the peak current drawn by the converter. This limits the maximum load on the 12 V wall-wart supply and also allows the peak current to be adjusted later to match the inductor’s saturation current—useful when working with inductors salvaged from the scrap bin.

{{< figure
    src="HV_PSU.png"
    caption="courtesy of Mr. Moorrees from [threeneurons](https://threeneurons.wordpress.com/nixie-power-supply/)"
    >}}

The series of 100 kΩ resistors serves two purposes: it reduces the number of unique items in the BOM, and it allows the voltage divider to withstand 200 V, since a single 0805 resistor is only rated for about 150 V peak.

Special attention was given to the potentiometer, which is placed on the low side of the voltage divider. Because this component is susceptible to mechanical wear and failures, placing it on the low side ensures that an open-circuit fault pulls the output voltage down to a safe level of about 2.5 V. If the potentiometer were placed on the high side instead, an open-circuit failure could drive the output voltage high enough to damage both the Nixie tubes and the switching transistors.

Anode resistor sizing
========

A resistor must be added between the HV supply and each Nixie digit. How is its value chosen?

The key point is that Nixies exhibit hysteresis in their turn-on and turn-off voltages. This means the striking voltage (the voltage needed to start a discharge) is higher than the holding voltage (the voltage required to keep a digit lit).

For example, a digit strikes at around 170 V, after which the anode-to-cathode voltage drops to roughly 133 V once the digit is lit. The datasheet specifies a maximum safe current of 2.5 mA; a conservative limit of 2.0 mA was selected.

The resistor can be calculated using Ohm’s law:

{{< katex >}}
$$
 R = \frac{V_{striking} - V_{holding}} {I_{max}}
$$

If the HV supply is 170 V and the holding voltage is 133 V, the resistor value is:

$$
 R = \frac{170 - 133} {0.002} = 18,5K\Omega
$$

A practical value of 20 kΩ was chosen.

It is also important to check the resistor’s power rating, since the power dissipated is approximately:
$$
P = \frac{\Delta V^2} {R} = \frac{37^2V} {20K\Omega} = \frac{1369V} {20K\Omega} \approx 0.07W 
$$

A 1210 resistor (\(\frac{1} {4}W\)) was selected in order to have a bit more headroom in case of a partial nixie failiure.

Timekeeping
========

The "old" way
--------

Using discrete logic ICs is my favorite approach. It’s purely academic—modern microcontrollers consume less power, are more accurate for timekeeping, cost less, take up less space, and offer many more features. Still, as a design exercise, building something from discrete logic can be fun. Starting with a pile of inert ICs and ending up with a circuit that performs a very specific task efficiently, all thanks to your own design, is extremely rewarding.

After completing the design up to the logical level, I realized it wasn’t such a good idea for this project. I wanted the clock to retain time even when the power was off for extended periods, and with a very precise TCXO (Temperature Compensated Crystal Oscillator), the circuit would have consumed too much power, meaning it wouldn’t last long on backup power.

The "Modern" way
--------

As mentioned before, modern microcontrollers outperform old discrete logic in almost every metric. For this project, I wanted to pair a microcontroller with a discrete RTC. Compared to using the microcontroller’s internal registers as a RTC, this approach is slightly more expensive and requires more board space. However, it provides extremely accurate timekeeping while maintaining very low power consumption during backup operation.


RTC
--------

As an RTC, a [PCF2129](https://www.nxp.com/docs/en/data-sheet/PCF2129.pdf) was selected, mainly because it is inexpensive (around 2$ on LCSC) and has an accuracy of ±3 ppm, or roughly 95 seconds per year. It supports both I2C and SPI interfaces, selectable by connecting the IFS pin either to ground or V_BBS to select SPI or I2C, respectively.

Since this clock may remain unused for up to six months each year, I wanted the RTC to maintain time as long as possible on backup power. In the current configuration, while keeping the internal thermal compensation of the oscillator active, the IC consumes approximately 2.5 µA. This allows a pair of AAA batteries to keep the RTC running for over 20 years, far longer than necessary.

Microcontoller
--------

For the microcontroller, an [attiny1616](https://ww1.microchip.com/downloads/aemDocuments/documents/MCU08/ProductDocuments/DataSheets/ATtiny1614-16-17-DataSheet-DS40002204A.pdf) was chosen. It was inexpensive yet cabaple, and being based on the AVR architecture I am comfortable with it and its toolchain.

Compared to what I used before, this chip uses a UPDI programming interface, which requires fewer pins than the usual SPI interface found on most Arduino clones. It should also allow hardware-level debugging, although this feature is not currently implemented in the Arduino APIs.

{{< figure
    src="mainBoardSchematic.png"
    >}}

Cathode driver
--------

To light up the Nixie digits, a cathode driver is required, capable of pulling the cathode to ground and withstanding the 170 V of the high-voltage supply. Since the ATtiny1616 has a limited number of I/O pins, and I need around 40 pins to drive all the Nixie digits plus a few additional pins for auxiliary lights, I opted to use 74HC595 shift registers.

The [74HC595](https://assets.nexperia.com/documents/data-sheet/74HC_HCT595.pdf) is a serial-to-parallel shift register that requires only 4 microcontroller pins to control a theoretically unlimited number of outputs. I chose to use 3 shift registers for each pair of Nixies. This provides 3 × 8 output lines, of which 10 × 2 are used to drive the cathodes and 2 extra lines serve as spare channels.

It is important to note that the output enable (OE) pin allows all output pins to enter a high-impedance state, effectively turning off the digit. This feature is exploited to dim the tubes by connecting a PWM pin to the OE input.

To interface each shift-register output with the high-voltage cathodes, I used an MMBTA44, a 400 V transistor, for each cathode.


{{< figure
    src="digitSchematic.png"
    >}}

Each pair of digits has a circuit like this. The 74HC595 shift registers are perfect for this kind of application: you can simply daisy-chain them, output more bits of data, and you are ready to go. This makes the design modular, easily upgradable, and maintainable.

I then added some momentary buttons and connectors to provide user input and neatly organize all the unused pins. An extra header was added to provide easy access to the programming interface and the I2C data line. By leaving this header unsoldered, it is possible to quickly probe it and monitor the data being exchanged.

Initially, I also made provisions for a light-sensitive resistor to automatically dim the watch in the dark; however, testing showed that this feature was not particularly useful.

What would I do differently?
--------

After some years of continuous use, a few critical issues have emerged.
The main problem is related to the tubes themselves. Since they are IN-1 tubes and do not contain any added mercury, their lifespan is quite limited compared to other types of Nixie tubes.

Another critical point is the board layout. The idea of using a single board to host all components is nice—it makes the mechanical construction easier—but it is not ideal from a maintenance point of view. After a few tubes failed, I realized it would have been better to use a small daughterboard for each tube, hosting the shift register and the associated transistors. This would have made the entire assembly much easier to maintain and would have greatly simplified the wiring, since each daughterboard would require only a single ribbon cable.

A final point concerns the dimming. Currently, the tubes are dimmed using PWM, simply switching them on and off very quickly to achieve the desired brightness. A smarter approach would have been to filter the PWM signal to obtain an analog voltage and inject that into the feedback pin of the HV converter. This way, the supply voltage to the tubes would change to achieve dimming. Opinions differ on whether this extends tube lifespan, but if implemented correctly—in both hardware and software—it may reduce stress on the tubes while avoiding under-driving or overloading them.

PDF Schematic
--------

{{< button href="schematic.pdf" target="_self" >}}
Download
{{< /button >}}