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
per gestire megliio la cosa uso mqtt explorer, ho una gui e nasce per l'mqtt, solo che una volta installato il .deb non va. Oh well!





