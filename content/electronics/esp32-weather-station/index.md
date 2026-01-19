---
date: 2026-01-18
draft: false
title: "ESP32 data acquisition board: HW requirements"
---

## Overview

I'd like to create a project to act as a proof of concept for a complete data acquisition and visualization pipeline, as one day I'd like to set up a distributed network gathering data across the valley where I live and through the use of LoRa transmit it to a locally running DB for further analysis and visualization.

In order to do so I need:

* A physical board gathering data
* Some kind of daemon that receives and stores the data
* A visualization utility

## Hardware

As this is a small test, I want to start out easy, trying to leverage as much as possible my already existing infrastructure, to avoid buying hardware that may not even be necessary later.

For this reason I decided to start out logging weather data and to transmit it to my local server over WiFi. For this purpose an ESP32 is the perfect choice, as it allows me to easily connect to the network and talk with my server, without the need for any gateway in between.

The ESP32 ticks all the other boxes, such as:

* Low power consumption, both in low power mode (using the ULP Coprocessor) and in deep sleep (5 µA)
* Integrated RTC
* Ability to synchronize the time automatically through an NTP server
* ADC/PWM/Timers/I²C/SPI/UART support to connect to any sensor
* Lots of memory to temporarily store the data if the network is not available

## Data gathering sensors

The following components have been chosen for data gathering, keeping an eye to the power budget. Another bonus is that everything is connected using I²C, simplifying the connections.

* Brightness/UV: LTR390
* Temperature and humidity: AHT20
* Pressure: BMP280
* Particulate (PM 1 to 10): SPS30
* SCD30: CO₂ sensing

## Power Requirements

The ESP32 should take less than 5 seconds to connect to WiFi after being woken up from deep sleep. To stay on the safe side, we consider 10 seconds to wake up, connect to the network, and send the data. We assume that during these 10 seconds the board is constantly transmitting (worst case scenario in terms of power consumption) and that, according to its datasheet, it consumes 239 mA. Assuming this happens once every hour, just for the data acquisition and transmission part of the project the power consumption is around 3.3 V * 0.239 A * 10 / 3600 = 2.2 mW.

This calculation doesn't count the deep sleep power consumption, but since the board consumes so little in that state (around 5 µA), it is negligible and it won't be counted to make the calculations easier to follow.

The sensors are a bit more tricky, mainly because the SPS30 needs some warm-up time before each reading.
Let's go in order:

* LTR390: around 250 µA while measuring, less than 1 µA while dormant
* AHT20: around 1 mA while measuring, less than 1 µA while dormant
* BMP280: around 1 mA while measuring, less than 5 µA while idle
* SPS30: around 60 mA at 5 V while measuring
* SCD30: around 5.9 mA while measuring

They all need some warm-up time, specifically:

* 10 ms for the LTR390
* 100 ms for the AHT20
* 2 ms for the BMP280
* 8 to 30 s for the SPS30, so we assume 60 s to stay on the safe side
* 120 s for the SCD30

Since the SPS30 is measuring a physical property that doesn't change infinitely fast and since it is the most energy-hungry device, we can try to understand how little we can run it to get meaningful data, in order not to worsen data quality, while using as little energy as needed.

Using [this site](https://charts.ecmwf.int/catalogue/packages/cams_air_quality_eea/products/plume_cams_eu_particulate_matter_2.5um_eea) I can see that the graph points are updated hourly. Thanks to that (and to the fact that the values move slowly) I decided to sample the air every 20 minutes, and to dial it back to once or twice an hour if the system consumes too much power.

For the other sensors we can safely sample them every 10 minutes and give them a warm-up period of 5 seconds, since they don't consume much power.

Now that we have done all our homework, we can finally estimate the power needed to run the sensors.

* SPS30: 5 V * 0.06 A * 1 / 60 * 3 = 15 mW
* SCD30: 3.3 V * 0.0059 A * 120 / 3600 * 6 = 3.9 mW
* All the other sensors: 3.3 V * 0.003 A * 5 / 3600 * 6 = 0.1 mW

After getting power to the battery, we still need to reduce the voltage to something that can be managed by the ESP32 since it needs a voltage between 2.3 and 3.6 V. Since we have high current bursts during WiFi usage, a switching voltage regulator is needed. This will increase power consumption slightly, but will probably make the system more efficient at high battery SOC where there will be less lost power.

I'm hoping to complete this project quickly using just off-the-shelf parts. A TPS63020 board was chosen: this is a buck-boost converter, so it is able to maintain a fixed output voltage even if the battery dips below 3.3 V.
Two of these boards will be used, one for the ESP32, and one to supply 5 V to the SPS30.

Since these are ready-made boards they are not terribly focused on efficiency, and from what I gathered even though the IC itself should have an operating quiescent current of around 25 µA the boards consume around 400 µA while nothing is connected.
Again, this results in around 3.7 V * 0.0004 A = 1.5 mW.

All considered, running the sensors as described above and sending the data to a server once an hour, plus the energy lost for the power supply, we would need around 2.2 mW + 15 mW + 3.9 mW + 0.1 mW + 1.5 mW = 22.7 mW on average.

## Power Source Selection

The idea is to have a self-contained unit, able to be deployed mostly anywhere. To do so a solar panel becomes necessary to generate power (or an [RTG](https://en.wikipedia.org/wiki/Radioisotope_thermoelectric_generator), but the cops wouldn't be happy about it).

To get an idea of the power that a solar panel could produce, we turn to this [useful site from the EU](https://re.jrc.ec.europa.eu/pvg_tools/en/tools.html).

It's good practice to design for the worst case. From this page, for my particular location, I can see that the month with less sun is December, and that for a panel at a 0° angle (i.e. lying flat on the ground) the irradiation is around 32 kWh/m² during that month.

Keeping in mind that a conservative efficiency for monocrystalline panels is around 15% and for polycrystalline ones around 13%, we know that a monocrystalline solar panel generates 32 * 0.15 = 4.8 kWh/m² each month, or 4.8 / (31 * 24) = 6.4 Wh/(m²*h), which means that a 1 m² panel would produce on average 6.4 W (5.5 W average for a 1 m² polycrystalline one).

To avoid losing energy, I decided to use an MPPT converter to charge the lithium battery. The CN3791 seems to be the perfect choice for this use case, as it is specifically designed to charge 1S lithium batteries from a solar panel, implements MPPT and has a constant current / constant voltage charging profile.

Looking at its datasheet it's not clear what efficiency it has in the real world, but usually a 90% efficiency is realistic for this kind of device.

So, to actually supply power to our little happy device we need 22.7 mW / 90% efficiency = 25.2 mW, which means that with an average power of 5.5 W/m² we would need a solar panel of 0.0252 / 5.5 = 0.0046 m², or a square panel with sides of about 65 mm.

Thermal derating considerations don't apply here, since during the summer the panels could get hotter, losing 10–15% of power, yet the solar radiation would be much higher, meaning that we still have plenty of power during the summer. On the bright side, during winter, with its shorter days and reduced irradiance, we may actually gain some efficiency (around 0.3–0.5% for each degree below 25 °C).

Since we want to have some headroom, panels are cheap and space is not really a concern, I opted to use a 170 x 120 mm 3 W solar panel, plenty more than what we actually need, able to keep the system running continuosly even with the battery mostly dead, or with a cloudy ski.

The last component not mentioned here is a TP4056 board with battery protection included. This board allows the system to be charged with USB and automatically disconnects the battery when the voltage gets too low, extending its lifetime. It's a worst case design: power should be plenty and a low voltage protection will be set in place, still it's nice to have some redundancy.

## Final Block Schematic

![Block Diagram](blockDiagram.png)

Since I'm building this project incrementally, the schematic above should be considered more of a guideline than a definitive design. The final implementation may change depending on what works in practice and on the components I have available to replace anything that does not.
