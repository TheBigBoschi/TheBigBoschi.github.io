---
date: 2026-03-23
draft: true
title: "Esp32 Enviromental Data Acquisition Node: Software"
---

Server funzionante, autenticazione funziaaa.
per gestire megliio la cosa uso mqtt explorer, ho una gui e nasce per l'mqtt, uso snap, il .deb ha problemi
per qualche motivo mosquitto non mi aggiunge correttamente un secondo utente, come workaround ho copiato l'hash

Per il FW, l'idea è di fare il provisioning tramite libreria, poi se ho delle credenziali in memoria uso i metodi standard espressif, questo perche la libreria per qualche motivo ci mette dai 40 ai 90s secondi per connettersi

il reset delle credenziali per qualche motivo va ripetuto 2 volte, una dopo il reboot. da indagare.

https://github.com/terdia/mqttui da indagare. MQTT client con web GUI, containerizzato.

SPS30: porting libreria e lato HW devo mettere il level shifter. nvm, i2c, open drain e open collector non ha problemi a funzionare finche il VIH è abbastanza alto, e basta sia > 2.3V per funzionare quindi non ho bisogno uno shift register, la vmax è data dalle resistenze di pullup.

# inizio il porting della libreriaaa
Mannaggia.

uso i2ctools di espressif per giocare con l'i2c senza scrivere un programma.
i2c-tools> i2cset -c 0x69 -r 0x01 0x04  ferma la misurazione
i2c-tools> i2cset -c 0x69 -r 0x00 0x10 0x05 0x00 0xF6   0x0010 inizia le misurazioni, e accende la ventola. 0x0500F6 i primi 2 byte indicano scrivi uint16 per i valori, F6 è il crc polinomiale ecc

paragrafo sul cuntrollare scl e sda, ti sei messo le mani tra i capelli e poi avevi invertito i 2.

https://sensirion.com/media/documents/1E3CD1FF/6165AFE4/Sensirion_GF_AN_SFM-04_CRC_Checksum_D1.pdf

usa i2c master read e non i2c master read write. 

dal datasheet ce la procedura per accendere e usare il modulo

https://www.ti.com/lit/an/slva704/slva704.pdf?ts=1770857779462

i2c tools lavora a byte, non gestisce le word. quindi per scrivere 0x1103 (wakeup) devo usare 
    i2cset -c 0x69 -r 0x11 0x03
altrimenti perde pezzi

per impacchettare il componente: https://github.com/espressif/example_components

# Sleep

Ho modellato il sistema come una SM, devo gestire edge cases.

## Modem sleep mode
hai fatto SM per definire i vari stati
Quando sei in attesa di istruzioni dal server usi la ESP32 Modem Sleep Mode per ridurre i consumi. Questo fa si che il device si svegli solodurante l'intervallo DTIM.

NFB: usa lo scaling dinamico del clock (DFS)

modalita min -> si sveglia ogni tot per controllare
modalita max -> si veglia SOLO ogni DTIM per mantenere la connessione, puo perdere pachetti.
max mode: 
wifi_config_t wifi_config = {
    .sta = {
        .ssid = "MY_SSID",
        .password = "MY_PASS",
        .listen_interval = 10, // Wake up every 10 beacons
    }, NON SI FA A RUNTIME, SOLO PRIMA DELLA CONNESSIONE.

se metti listen_interval = n ti svegli ogni n dtim. ha senso solo in max config.
NB! hai DTIM pari a 1 sec, NON laori in max mode, solo in min.
per lavorare in min mode fai esp_wifi_set_ps(x); con WIFI_PS_NONE o WIFI_PS_MIN_MODEM (o WIFI_PS_MAX_MODEM) a runtime, dopo che il wifi è stato acceso. Lo fai quando non sei in provisioning, altrimenti alzi la latenza.

piu info ( e reference): https://github.com/espressif/esp-idf/tree/v5.5.3/examples/wifi/power_save

## Sensor polling 

Mentre polli i sensori lavori con frequenza dinamica. attivi il dynamic scaling con https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/power_management.html

## deep sleep

Reference: 
    https://github.com/espressif/esp-idf/tree/v5.5.3/examples/system/deep_sleep
    https://github.com/espressif/esp-idf/blob/v5.5.3/examples/system/deep_sleep/main/deep_sleep_example_main.c#L128

RTC wake signal: ESP_ERROR_CHECK(esp_sleep_enable_timer_wakeup(wakeup_time_sec * 1000000)); configura anche un GPIO? o resetti?
per mandare in sleep uso esp_deep_sleep_start();

svegliarsi da un deep sleep è uguale a fare un reset, l'unica differenza la ho consultando esp_sleep_get_wakeup_cause();
e per il fatto che quello che ho messo in rtc ram prima dello sleep rimane.







