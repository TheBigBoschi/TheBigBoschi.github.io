---
date: 2026-01-18
draft: true
title: "ESP32 data acquisition board: HW requirements"
---

Overview
------

I'd like to set up a project to showcase a complete pipeline of data acquisition, in order to do so I need:

* A physical board gathering data
* Some kind of daemon that receives and stores the data
* A visualization utility

This is both a showcase and a proof of concept, as one day I'd like to set up a distribuited network gathering data across the valley where I live and trough the use of LoRa transmit it to a local-running DB for further analysis and visualization.

Hardware
------
As this is a small test, I want to start out easy trying to leverage as much as possible of my already existing infrastructure, to avoid buying hardware that maybe wont even be necessary later.

For this reason I decided to start out logging weather data and to ttransmit it to my local server over wifi. For the purpose an ESP32 is the perfect choice, as it allows me to easily connect to the network and talk with my server, whitout the need for any gateway in between. 

ESP32 ticks all the other boxes, such as:
* Low power consumption, both in low power mode (using the ULP Coprocessor) and in deep sleep (5uA)
* Integrated RTC
* Ability to sincronize the time automatically trough a NTP server
* ADC/PWM/Timers/I²C/SPI/UART support to connect to any sensor 
* Lot of memory to temporarily store the data if the network is not available

Data gathering
------

Currently (18/01/26) I'm in the phase of ordering the correct components. They 

* Brightness/UV: LTR390
* Temperature and humidity: AHT20
* Pressure: BMP280
* Particulate: SPS30
* SCD30: CO2 sensing

The following has been choosen for some data gathering, whitout exceeding the power budget. Another bonus is that everything is connected using I²C, simplifying the connections.

Power Requirements
------

The ESP32 should take less than 5 seconds to connect to a wifi after being woken up from deep sleep. to stay on the safe side we consider that number 10 seconds to wake, connect to the network and send the data. We assue that during this 10 seconds the board is constantly transmitting (worst case scenario in terms of power) and that as per it's datasheet, it's consuming 239mA. Assuming we do so every hour, just for the data acquisition and transmit part of the project the power consumption is around 3.3V * 0.239 A * 10/3600 = 2.2 mW

The sensor are a bit more tricky, mainly because the PMSA003 needs a bit of warm up time before each reading.
Lets go in order:
* LTR390: around 250uA while measuring, less than 1uA while dormant  
* AHT20: Around 1mA while measuring, less than 1uA while dormant.
* BMP280: Around 1mA while measuring, less than 5uA while in idle.
* SPS30: Around 60mA at 5V while measuring.
* SCD30: Around 5.9mA while measuring.

They all need some warmup time, specifically
* 10ms for the LTR390
* 100ms for the AHT20
* 2ms for the BMP280
* 8 to 30s for the SPS30, so we say 60 to stay on the safe side.
* 120s for the SCD30

Since the SPS30 is measuring a physical property that doesnt change infinitely fast and since is the most energy hungry device, we can try to understand how little can we run it to get meaningful data, in order not to worsen the data quality, but yet make it use as little energy as needed.

Using [this site](https://charts.ecmwf.int/catalogue/packages/cams_air_quality_eea/products/plume_cams_eu_particulate_matter_2.5um_eea) I can see that the graphs point are updated hourly, thanks to that (and the fact that the values move slowly) I decided to sample the air every 20 minutes, and to dial it back to once an hour if the system consumes too much power.
Instead for the other sensors we can safely sample them every 10 minutes and give them a warm-up period of 5 seconds since they dont consume much power.

Now that we have done all our homeworks, we can finally extimate the Wh needed to run the sensors.

* PMSA003: 5V*0.06A*1/60*3 = 15mW
* SCD30: 3.3V*0.0059A*2/60*6 = 3.9mW
* All the other sensors: 3.3V*0.003A*5/3600*6 = 0.1mW

after getting power to the battery, we still need to reduce the voltage to something that can be managed by the esp since it needs a voltage between 2.3 and 3.6 volts. Since we have high current bursts during the wifi usage a switching voltage regulator is needed. This will increase a bit the power consumption but will probably make the system more efficient at high battery SOC where there will be less lost power.

I'm hoping to complete this project quickly using using just off the shelf parts, a TPS63020 board was chosen, this is a buck-boost converter, so it's able to maintain a fixed output voltage even if the battery dips below 3.3V.
Two of these board will be used, one for the esp, and one to supply 5v to the SPS30.
Since these are ready made boards they are not terribly focused on efficiency, and from the little that i gathered even though the IC should have a operating quiescent current of around 25uA the boards consumes around 400uA while noting is connected.
Again, this will eat out around 3.7*0.0004 = 1.5mW.

All considered, running the sensor as described above and sending the data to a server once an hour, plus the energy lost for the power supply we would need around 2.2mW + 15mW + 3.9mW + 0.1mW + 1.5mW = 22.7mW

Power Source Selection
------

The idea is to have a self contained unit, able to be deployed mostly anywhere. To do so a solar panel becomes necessary to generate power (or an [RTG](https://en.wikipedia.org/wiki/Radioisotope_thermoelectric_generator), but the cops wouldn't be happy about it). 

To get an idea of the power that a solar panel could produce, we turn to this [useful site from the EU](https://re.jrc.ec.europa.eu/pvg_tools/en/tools.html).

It's good practise to design for the worst case, from this page, for my particular location, I can see that the month with less sun it's december, and that for a panel at a 0° angle (ie liying flat on the ground) the irradiation is around 32 kWh/m² during that month.
Keeping in mind that monocristalline have an efficiency of around 15% and polycristalline 13%, we now know that a monocristalline solar panel generates 32*0.15 = 4.8kWh/m² each month, or 4.8/(31*24) = 6.4 Wh/(m²*h), which basically means that a 1m² panel would produce on average 6.4W. (5.5 W average for a 1m² polycristalline one).

To avoid loosing energy, i decided to use an MPPT converter to charge the lithium battery. The CN3791 seems to be the perfect choice for our use case, it's specifically designed to charge 1S lithium batteries from a solar panel, implements MPPT and has a constant current function for the end of charge stage of the battery.
Looking at it's datasheet it's not clear what efficiency it has in the real world, but usually a 90% efficiency is realistic for this kind of devices. 

So, to actually supply power to our little happy device we need 20.5mW / 90% efficiency = 22.8mW, which means that with an average power of 5.5W/m² we would need a solar panel of 0.0227/5.5 = 0.0041m², or a square panel with the sides of 65mm each.

Thermal derating considerations dont apply here, since during the summer the panels could get hotter, losing 10-15% of power, yet the solar radiation would be much higher, meaning that we still have plenty of power during the summer. On the bright side, during winter, with its shorter days and reduced irradiance, we may actually gain some efficiency (around 0.3-0.5% for each degree below 25°).

And since we know that solar panels are characterized at 1000W/m² we also know that we would need a panel with a nominal power odf 0.0041m²*1000W/m²*0.15% = 0.615W panel. Assuming the specs given to us on Aliexpress are to be trusted.

Since we want to have some headroom and a 