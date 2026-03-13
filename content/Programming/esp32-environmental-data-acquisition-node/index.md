---
date: 2026-01-20
draft: true
title: "Esp32 Environmental Data Acquisition Node"
---
## Overview
In the previous post I defined what the HW architecture should look like, now I'm turning to the software.

The flow here is pretty simple:
Read the data -> send it over the WiFi to a server (local or not) -> Do some backend magic and save it to a DB. For now I'm concentrating over the first two steps.

Even before thinking about the SW on the ESP I realized that a pretty important piece here was the protocol with wich my data would move on my network and get to my server.

I started by laying out my requirements:
* Low overhead
* Ease of implementation
* Bidirectional
* Safe
* OK with less-than-ideal signal/internet reception
* Scalable (Just for fun!)

Usually the go-to in the web realm the REST protocol is the go to for web application, but this is not a web app and it has drawbacks in my use case, first of all that i cant achieve bidirectional communication easily from my network to a server on the cloud since my network (as most are actually) is behind a NAT.

In this case I choose to use a MQTT server to move the data, since it's extremely data-efficient (low overhead), the client implementation is maintained by espressif and well documented (easy to implement), I can easily send data to a topic while subsribe to another (bidirectiona data flow), has a qos parameter that ensure that my message gets delivered to the broker (reliabile with bad network), gives me the possibility to have TLS and to autenticate my client on the server.

It has two main drawback for me:
* while waiting for commands it keep the modem on
* It needs a MQTT broker to work

Regarding the first point, with the modem low power mode it should consume just 80mA while wating, and it should not impact too much my power budget if i keep it on for just 30-60 seconds every time that it's waiting for commands from the server.

Regarding the second, I consider the broker to be part of the backend services, and since i need a DB i can just run it togheter.

## Broker setup

I found out that the industry standard MQTT broker is mosquitto. It's lightweight, free and open source.
As always, docker rules, and I found a container running it.
In my setup, at least for now, the container is hosted on a x64 server reachable at 192.168.1.20 address.


To run it just save the following text in a file called compose.yaml:
```yaml
services:
  mosquitto:
    image: eclipse-mosquitto:2
    container_name: mosquitto
    restart: unless-stopped
    ports:
      - 1883:1883
      - 8883:8883
      - 9001:9001
    volumes:
      - /path/to/config:/mosquitto/config
      - /path/to/data:/mosquitto/data
      - /path/to/log:/mosquitto/log
    networks:
      - mqtt
networks:
  mqtt:
    driver: bridge
```
And then in the same folder as the file type
```bash
docker-compose up
```

It's important to specify the paths to route out in the compose file in order to maintain the config and the data even if the container gets updated.

In my code I'm autenticating the clients (but I'm not using TLS until I move the broker to a vps at least), to do so we have to create the users and specify theyr password. For now a there is a shared user for all the clients.

To do so log in the container and run the following command:
```bash
mosquitto_passwd /mosquitto/config/mosquitto.password esp32 //personalize esp32 with your username
```
It will then prompt for the user password.

To actually test the configuration we can use Curl to send and to receive the payload in two separate terminals.

First, to open the receiving terminal:
```bash
curl -N -u "esp32:password" mqtt://192.168.1.20:1883/esp/test --output  //password is the password that we have set before
```
And then, to send the message "hello" to the other terminal:
```bash
curl -u "esp32:password" -d "hello" mqtt://192.168.1.20:1883/esp/test
```
Notice that the URL is composed of the following parts:
* `mqtt://` Specifies the protocol tu curl
* `192.168.1.20` IP of the broker 
* `:1883` Port to send the messages. 1833 for unencripted data, 8883 for using TLS.
* `/esp/test` The topic in wich the payload will be sent to. You can send to multiple topics using the same connection. It's useful to post data and telemetry or warning in separated channels.

The received message will have the following format: `*topic**payload*` this is normal, it's just that curl it's not the right tool for the job here.

Once the broker is up and running we can move to a better tool to visualize the received payload, my personal reccomendation is MQTT Explorer, also available from the App center on Ubuntu. It's just plug and play, no fuss.

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






