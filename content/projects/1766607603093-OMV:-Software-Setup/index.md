---
title: "OMV: Software Setup"
date: 2025-12-24
draft: false
description: "a description"
tags: ["example", "tag"]
---

OMV Configuration
========

In the previous post, OpenMediaVault (OMV) was installed on our SBC. Now it is time to configure the system to perform all the required actions.

Adding the disks
--------

To mount drives other than the boot drive, they need to be added to fstab. However, it is recommended not to edit fstab manually. Instead, use the OMV web interface:

Navigate to Storage → Disks and verify that all drives are properly recognized by the system.

Then go to Storage → File Systems. From the toolbar, click Mount to mount an existing partition, or Create and Mount to initialize a new drive with a partition table and mount it.

This approach ensures that drives are correctly configured and minimizes the risk of misconfiguration or data loss. OMV’s official guide recommends following this method, as alternative approaches could interfere with system functionality, for example, preventing disks from spinning down when idle.

OMV-Extras Plugin
--------

Docker is not enabled in OMV by default. To enable it, the first step is to install [OMV-Extras](https://wiki.omv-extras.org/doku.php?id=misc_docs:omv_extras), which provides access to additional plugins. You can install it by running the following command in the terminal:

```bash
sudo wget -O - https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/install | bash
```
After rebooting, OMV-Extras will be installed. This extension allows you to download and install extra plugins for OMV, unlocking a wide range of additional functionality, including Docker support.

Enabling Docker
--------

Once OMV-Extras is installed, Docker can be enabled on the system. First, navigate to System → OMV-Extras, check the Docker repo option, and save the settings.

The interface between OMV’s GUI and Docker is provided by the [Compose Plugin](https://wiki.omv-extras.org/doku.php?id=omv7:omv7_plugins:docker_compose). To enable it, go to System → Plugins and install the openmediavault-compose plugin.

After installation, a Compose section will appear under the Services tab, where you can add, configure, and manage your Docker containers directly from the web interface.

Moving Docker and Containerd
--------

Before proceeding further, it is good practice to move the storage location of Docker and [Containerd](https://www.docker.com/blog/containerd-vs-docker/) in order to save space on the eMMC.

It is important to note that this step is not necessary for most users with more capable boards or for systems installed on an SSD.

During normal operation, Docker and Containerd pull images and build containers, which can consume a significant amount of storage. For example, running a single Emby container and an Immich container already used about 10 GB of flash. While this may seem manageable, on a small SBC with only 16 GB of eMMC storage, it is already too much.

To address this, I decided to move both Docker and Containerd to another disk.

Moving Docker is straightforward. Navigate to Services → Compose → Settings in the OMV web interface and choose the desired installation path.

To move Containerd, first stop the service:
```bash
 sudo service containerd stop
```
Then, modify the Containerd configuration file. I used Nano, the integrated text editor, to edit the file:

```bash
sudo nano /etc/containerd/config.toml
```
Within the file, modify the following line to point to your desired storage path:

```bash
root = "/var/lib/containerd" # your_path_here
```
After saving (Ctrl+S) and exiting (Ctrl+X), rename the existing Docker and Containerd folders to allow for rollback if needed:

```bash
 sudo mv /var/lib/containerd /var/lib/containerd.old
 sudo mv /var/lib/docker /var/lib/docker.old
```
Finally, restart the Containerd service to apply the new configuration:

```bash
 sudo service containerd restart
```
To ensure everything loads correctly, it is recommended to reboot the system. After that, Docker and Containerd will use the new storage locations.

Docker compose files
--------

OMV provides a graphical editor for Docker Compose files, which is perfect for the setup I had in mind. The editor can be found under Services → Compose → Files. Using the toolbar, click Add to create a new Compose file. By checking Show environment file, you can define an .env file for the container. This will be particularly useful for Immich, whose default Compose file references a separate .env containing paths and passwords for the media folder and local database.

Setting up Emby
--------

Emby is the media server application I chose, primarily because I had already used it on Windows to stream content to TVs and phones, and it works reliably with minimal configuration.

I opted for the [LinuxServer version of Emby](https://hub.docker.com/r/linuxserver/emby). The linked page provides an example Compose file, which allows the system to run out of the box. Minor adjustments are needed, such as updating the media folder paths and commenting out shared device lines that are not applicable to your hardware (mainly for hardware acceleration).

Setting up Immich
--------

Immich is a self-hosted photo and video management solution, essentially a self-hosted alternative to Google Photos. It supports metadata extraction and can automatically detect and group photos containing the same faces.

I followed the [recommended docker compose](https://docs.immich.app/install/docker-compose/) provided by Immich. The process is similar to Emby, with the addition of downloading and placing the .env configuration file in the correct location. Following the official guide ensures that all necessary volumes are exposed to the container correctly.

What's still missing
--------

Since this setup is primarily a mockup to test system stability using spare parts, several features remain to be implemented once the system is validated:

* Adding a container to download torrent files with a web GUI

* Installing Tailscale and creating a custom VPN, along with a reverse proxy using Nginx for secure remote access (essential without a public IP)

* Setting up automatic backups for both the OS and data disks

* Configuring cTerm for web-based access to both the system and containers

* Configuring Samba for file sharing

Impressions
--------

So far, I am not satisfied with the system’s stability.

When it works, it works flawlessly, even when running Immich with facial recognition, which is CPU-intensive and not easily handled by a small SBC. Performance itself is surprisingly good.

The main issue is stability. The system runs for a few days and then, during the night, it shuts down unexpectedly. The next morning, it may refuse to boot. I have reinstalled the system multiple times, tweaking configurations each time, yet the problem persists.

The next step will be to install the system on an SSD connected via SATA. I expect this may improve both speed and reliability. Currently, I am running a sanity check on the eMMC using:

```bash
armbianmonitor -c /path/to/test
```

I suspected the eMMC might be corrupted, and the test confirmed it.

Before migrating the system, I plan to copy fstab and key configuration files to preserve disk mounts and Docker Compose paths. Since all user data is stored on HDDs, this approach allows me to redeploy containers without modifying paths, ensuring that the system comes back online without data loss.

I must say, I truly appreciate the containerization concept. It makes this entire process far more manageable and resilient.