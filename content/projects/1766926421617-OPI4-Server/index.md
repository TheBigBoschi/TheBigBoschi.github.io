---
title: "OPI4 Server"
date: 2025-12-28
draft: true
description: "a description"
tags: ["example", "tag"]
---

ip: Network-Manager al posto di netplan

per qualche motivo hai un interfaccia di modifica con sudo nmtui-edit e sudo nmtui-connect

Docker: manca copose plugin. prima aggiungo il repo poi lo posso scaricare
curl -L https://github.com/linuxserver/docker-docker-compose/releases/download/1.28.5-ls32/docker-compose-armhf | sudo tee /usr/local/bin/docker-compose >/dev/null

ci sono cazzi con apparmor, un servizio necessario a docker.
sudo apt update
sudo apt install -y apparmor apparmor-utils

dovrebbe risolvere.
Spoiler: lo ha fatto.

Stai combattendo contr l'ip, stai usando nmtui

If eth0 is UNMANAGED (very common on Armbian) Check: 
nmcli device status

If eth0 is unmanaged, fix it: 

sudo nmcli device set eth0 managed yes 
sudo systemctl restart NetworkManager

l'ip lo setti tramite sudo nmtui

disk backup from another pc:
sudo dd if=/dev/sdd bs=4M status=progress | gzip -1 > "/media/boschi/Data/Backup SSD OPI4/251228 - Backup SSD - dd.img.gz"

img restore:
gunzip -c dd.img.gz | sudo dd of=/dev/sdd bs=4M status=progress

installa cockpit

cockpit docker manager: https://github.com/chrisjbawden/cockpit-dockermanager

se non ti lascia fare docker ps senza sudo fai sudo usermod -aG docker boschi per mettere l'utente nel gruppo
newgrp docker per applicare le modifiche

https://cockpit-project.org/applications per accdeere a tutti i plugin ufficiali

curl -sSL https://repo.45drives.com/setup | sudo bash
sudo apt-get update
sudo apt install cockpit-file-sharing   -> plugin gestione SMB e NFS

NO! https://github.com/adelolmo/hd-idle -> semplicemente sudo hd-idle spegne i dischi dopo 10 minuti di inattività

uso questo:
sudo hdparm -S 240 /dev/sdX per spindown

stato instantaneo dischi: 

sudo hdparm -C /dev/sdb
sudo hdparm -C /dev/sdc
sudo hdparm -C /dev/sdd


Tailscale:
https://tailscale.com/kb/1282/docker

per far funzionare tutti i servizi assieme devo metterli nello stesso file -> NON PIU VERO CON TAILSCALE SULL'HOST!

https://www.youtube.com/watch?v=DoxjmTe-gSs -> spiegazione nginx

installa tailsclae sull'host: curl -fsSL https://tailscale.com/install.sh | sh
lancia tailscale: sudo tailscale up

I container lavorano in maniera predefinta sulla connessione bridge, li loro sono in grado di accedere

Controllo log: systemctl

sudo journalctl --since 00:00:00 > log.txt 
estrae i log da mezzanotte del giorno corrente e li salva in log.txt
ho avuto un reboot alle 1.12 

ToDo: public smb share, understand why the disks dont shut off