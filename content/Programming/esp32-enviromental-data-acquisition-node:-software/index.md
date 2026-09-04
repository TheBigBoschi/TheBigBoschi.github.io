---
date: 2026-03-23
draft: false
title: "ESP32 Enviromental Data Acquisition Node: Software"
---

Since I was trying out some new stuff for this particular project I didn’t do everything in the most structured way, I started with an implementation of the most tricky feature, and now I’m defining a functional diagram to properly define what my board should do, and the conditions necessary to stop one action and perform the other.

## State Machine
Since my project has quite a number of different parts that have to orderly interact with each other, I opted to use the state machine pattern to model the interaction between the various modules.

From a high level my device has to:

* Wake up
* Connect to the network
* Gather the data
* Send the data
* Wait for a ACK/NACK
* Get back to sleep
* repeat forever
Pretty straightforward, right?

Now, how does the software know when to move from one state to the other, and to which state to go to? Here things gets a bit more tricky.

To better understand how the various states changes I started out by sketching up on a piece of paper the following diagram:

{{< mermaid >}}
    graph TD;

    Start([Wake Up]) --> asd{Loss of power?}
    S1[Check network configuration]
    S2[Wi-Fi Provisioning]
    S3[Read sensors]
    S4[Wi-Fi connection]
    S5[Send Data and Wait For Feedback]
    S6[Stand By]
    S0 --> S1
    S1 -- No credentials in NVS --> S2
    S2 --> S3
    S1 -- Credentials in NVS--> S3
    S3 --> S4
    S4 --> S5
    S5 --> S6
    S6 --> S3

{{< /mermaid >}}

This is jsut an overview of what my code should do, there are some optimizations and edge cases to take care of but mostly this is how it should work.


ltr390 info: https://esphome.io/components/sensor/ltr390/
https://components.espressif.com/components/k0i05/esp_ltr390uv/versions/1.2.7/readme
NB!parla e implementa una funzione per normalizzare i valori letti rispetto a data/ora

BMP280: https://components.espressif.com/components/esp-idf-lib/bmp280/versions/1.0.7/readme
https://github.com/esp-idf-lib/bmp280/blob/ba9aaa92d7416740703b044a53462c0e9b8ee4e8/examples/default/main/main.c

NB pagine 28 del datasheet, parla di multipli indirizzi in base a un pin
Librerie esistenti vecchie, sembra che l'opzione migliore sia di creare un wrapper per fornire la vecchia firma della funzione che internamente usi la nuova

Aspetto aht40

https://components.espressif.com/components/k0i05/esp_bmp280/versions/1.2.7/readme

https://sensirion.com/media/documents/A88858C9/629626D4/Application_Note_Creep_Mitigation_SHT4x.pdf

Pirla! devi lasciare il tempo al sensore di completare la lettura prima di interrogarlo, altrimenti non risponde.

L'heather non è stato usato, non ho umidità critiche

BMP280: funzia. Pubblica snippet di codice come repo.

parla dello scope creep e di come ti ispira monitorare i terremoti, anche se vivi in mezzo alle alpi (stabili sismicamente)


Read sensor:
Iinizializzo I2C controller, apro 3/4 thread (uno per sensore) e sincronizzo con un semaforo.

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

## RTC
tempo aggiornato tramite sntp, come timer uso quelo da 8.5Mhz, piu corrente ma migliori prestazioni.
piu info a https://docs.espressif.com/projects/esp-idf/en/v5.0.3/esp32s2/api-reference/kconfig.html#config-rtc-clk-src

## MQTT
3 stream
* Data
* Telemetry
* Commands

i primi due inviano i dati al server, il terzo si mette in ascolto per feedback (ok/ack o resend data)
struttura: id/*node_id*/[Data/Telemetry/commands]
l'idea è con le wildcard su telegraf mi smazzo tutto il traffico in ingresso al db in maniera semplice, ma andando a spulciare in fase di debug la divisione per client e dopo per data/telemetria mi semplifica il debug

per smazzarmi il feedback uso il pattern producer consumer, MQTT_event_handler fa da produttore, passa i dati in una coda e esce. Creo un altro metodo che prende i dati, parsa, ritrasmette o termina, e setta un bit su cui il loop principale sta aspettando, poi esce.

Uso una coda, um generico taskNotify sarebbe stato abbastanza, la coda ha un leggero overhead ma mi da flessibilità, se due messaggi arrivano uno dopo l'altro prima che l'esecuzione del primo sia terminata non ho stati indefiniti.

definisco protocollo per ritrasmissione (comand da server)
* DATA_ACK: all good, go back to sleep
* RETRANSMIT n - N: retransmit all the data with utc time between the two times

Comandi da nodo:
* NO_DATA: dati del periodo non disponibili. Ritrasmetto pacchetto vuoto?
* altrimenti ritrasmette pacchetti mancanti

## JSON
the data has been packaged in json format, the schema has been optimized to reduce the payload size. Initially i planned to use an array of struct, but sending multiple arrays of data reduces drammatically the size, going from:

SISTEMARE! è cambiaro lo schema e uno è un array, l'altro un singolo pacchetto.
come UUID uso gli ultimi 3 byte del MAC, che sono univochi (i primi 6 indicano il produttore della scheda, sono tutti uguali sugli esp)

```json
{
  "device_id": "ESP32_ENV_NODE_02",
  "data_log": [
    {
      "sensor_data": {
        "time": 1718964000,
        "PM2p5": 12.30,
        "PM10p0": 18.70,
        "ambient_light": 450,
        "uvi": 3,
        "humidity": 55,
        "temperature": 24.50,
        "pressure": 1013.25,
        "payload_group": 101,
        "errors": 0
      },
      "telemetry": {
        "Vbatt": 4.12,
        "board_temp": 28.40,
        "rssi": 42,
        "payload_group": 101,
        "next_wakeup": 900
      }
    }
  ]
}
```

to

```json
{
  "device_id": "ESP32_ENV_NODE_02",
  "data_log": [
    {
      "time": 1718964000, "grp": 101, "err": 0, "next": 900, "rssi": 42,
      "pm25": 12.30, "pm10": 18.70, "lux": 450, "uvi": 3, "hum": 55, "temp": 24.50, "press": 1013.25, 
      "vbat": 4.12, "btemp": 28.40
    }
  ]
}
```






