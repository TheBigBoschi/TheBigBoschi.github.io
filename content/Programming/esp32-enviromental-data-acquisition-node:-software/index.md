---
date: 2026-03-23
draft: true
title: "ESP32 Enviromental Data Acquisition Node: Software"
---

Since I was trying out some new stuff for this particular project I didn’t do everything in the most structured way, I started with an implementation of the most tricky feature, and now I’m defining a functional diagram to properly define what my board should do, and the conditions necessary to stop one action and perform the other.

State Machine
Since my project has quite a number of different parts that have to orderly interact with each other, I opted to use the state machine pattern to model the interaction between the various modules.

From a high level my device has to:

Wake up
Connect to the network
Gather the data
Send the data
Get back to sleep
repeat forever
Pretty straightforward, right?

Now, how does the software know when to move from one state to the other, and to which state to go to? Here things gets a bit more tricky.

To better understand how the various states changes I started out by sketching up on a piece of paper the following diagram:

First power on

Timer wake up

Credentials present in NVS

Credentials not present in NVS

Credentials saved

Transmission needed

Transmission not needed

Connection established

Connection not established

SRV OK or timeout

Timer wake up

Power On

Check network configuration

Wi-Fi Provisioning

Read sensors

Wi-Fi connection

Send Data and Wait For Feedback

Error Handler

Stand By

As tedious as it was to code this diagram, I really think that having some kind of reference to refer to later on when I’m implementing the state machine will make everything a breeze.

This is more verbose than what is really needed, but by implementing all the states listed above as functions I can make the code clean and easily maintained, for example by changing just the error handling function, while if that was implemented inside the Wi-Fi connection function I would scramble a bit to find it.

State Machine implementation
https://gemini.google.com/share/763f97e2d4b3

Sleep
Ho modellato il sistema come una SM, devo gestire edge cases.

Modem sleep mode
hai fatto SM per definire i vari stati Quando sei in attesa di istruzioni dal server usi la ESP32 Modem Sleep Mode per ridurre i consumi. Questo fa si che il device si svegli solodurante l’intervallo DTIM.

NFB: usa lo scaling dinamico del clock (DFS)

modalita min -> si sveglia ogni tot per controllare modalita max -> si veglia SOLO ogni DTIM per mantenere la connessione, puo perdere pachetti. max mode: wifi_config_t wifi_config = { .sta = { .ssid = “MY_SSID”, .password = “MY_PASS”, .listen_interval = 10, // Wake up every 10 beacons }, NON SI FA A RUNTIME, SOLO PRIMA DELLA CONNESSIONE.

se metti listen_interval = n ti svegli ogni n dtim. ha senso solo in max config. NB! hai DTIM pari a 1 sec, NON laori in max mode, solo in min. per lavorare in min mode fai esp_wifi_set_ps(x); con WIFI_PS_NONE o WIFI_PS_MIN_MODEM (o WIFI_PS_MAX_MODEM) a runtime, dopo che il wifi è stato acceso. Lo fai quando non sei in provisioning, altrimenti alzi la latenza.

piu info ( e reference): https://github.com/espressif/esp-idf/tree/v5.5.3/examples/wifi/power_save

Sensor polling
Mentre polli i sensori lavori con frequenza dinamica. attivi il dynamic scaling con https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/power_management.html

deep sleep
Reference: https://github.com/espressif/esp-idf/tree/v5.5.3/examples/system/deep_sleep https://github.com/espressif/esp-idf/blob/v5.5.3/examples/system/deep_sleep/main/deep_sleep_example_main.c#L128

RTC wake signal: ESP_ERROR_CHECK(esp_sleep_enable_timer_wakeup(wakeup_time_sec * 1000000)); configura anche un GPIO? o resetti? per mandare in sleep uso esp_deep_sleep_start();

svegliarsi da un deep sleep è uguale a fare un reset, l’unica differenza la ho consultando esp_sleep_get_wakeup_cause(); e per il fatto che quello che ho messo in rtc ram prima dello sleep rimane.







