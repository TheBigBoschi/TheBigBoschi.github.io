---
date: 2026-01-20
draft: true
title: "Esp32 Environmental Data Acquisition Node"
---

Aggiungi la cartella elettronica o sw che sia dentro alle fonti per gli articoli recenti

dopo aver autenticato la rete setuppo il client mqtt. prima devo testarlo -> sul server lancio un istanza di mosquitto salvando i log.
dentro alla consolle lancio 

mosquitto_passwd /mosquitto/config/mosquitto.password esp32 //esp32 è lo user

ora chiede la pw. NB il -c  crea il file e lo sovrascrive se esistente.  https://mosquitto.org/documentation/authentication-methods/

NON HO ABILITATO TLS per ora non serve, e creare e sherare i certificati CA è sbatta.

Ho installato mosquitto in docker. per testarlo posso usare curl, visto che ho usato pw e id devo usare:
mqtt://[user:password@]broker[:port]/topic
curl -N -u "esp32:test" mqtt://192.168.1.20:1883/esp/test --output - //-N non buffera e printa subito a schermo, -u per il login, --output - printa a terminale.

i messaggi ricevuti sono del tipo *topic**payload*, è normale, è solo che curl non nasce per questo e va bene cosi.

per inviare: curl -d "ciao" mqtt://esp32:test@192.168.1.20:1883/esp/test // -d specifica di effettuare una richiesta post
oppure: curl -u "esp32:test" -d "hello" mqtt://192.168.1.20:1883/esp/test

Server funzionante, autenticazione funziaaa.
per gestire megliio la cosa uso mqtt explorer, ho una gui e nasce per l'mqtt, uso snap, il .deb ha problemi
per qualche motivo mosquitto non mi aggiunge correttamente un secondo utente, come workaround ho copiato l'hash

Per il FW, l'idea è di fare il provisioning tramite libreria, poi se ho delle credenziali in memoria uso i metodi standard espressif, questo perche la libreria per qualche motivo ci mette dai 40 ai 90s secondi per connettersi

il reset delle credenziali per qualche motivo va ripetuto 2 volte, una dopo il reboot. da indagare.

https://github.com/terdia/mqttui da indagare. MQTT client con web GUI, containerizzato.

SPS30: porting libreria e lato HW devo mettere il level shifter. nvm, i2c, open drain e open collector non ha problemi a funzionare finche il VIH è abbastanza alto, e basta sia > 2.3V per funzionare quindi non ho bisogno uno shift register, la vmax è data dalle resistenze di pullup.

## inizio il porting della libreriaaa
Mannaggia.

uso i2ctools di espressif per giocare con l'i2c senza scrivere un programma.
i2c-tools> i2cset -c 0x69 -r 0x01 0x04  ferma la misurazione
i2c-tools> i2cset -c 0x69 -r 0x00 0x10 0x05 0x00 0xF6   0x0010 inizia le misurazioni, e accende la ventola. 0x0500F6 i primi 2 byte indicano scrivi uint16 per i valori, F6 è il crc polinomiale ecc

paragrafo sul cuntrollare scl e sda, ti sei messo le mani tra i capelli e poi avevi invertito i 2.

https://sensirion.com/media/documents/1E3CD1FF/6165AFE4/Sensirion_GF_AN_SFM-04_CRC_Checksum_D1.pdf

usa i2c master read e non i2c master read write. 





