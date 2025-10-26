---
title: Using Proxmox as a Home Server Part IV - HASS and ESPHome
description: This installment will focus on installing and configuring Home Assistant and ESPHome in a Fedora LXC container using docker-compose.
date: 2025-10-26T20:25:44.671Z
preview: /media/posts/pve-rack/Drive_Cables.jpg
draft: true
tags:
  - Guide
  - Network
  - Projects
  - Proxmox
  - Server
  - Virtualisation
  - ZFS
  - LXC
categories: []
toc: true
autonumber: true
math: true
hideBackToTop: false
hidePagination: false
showTags: true
fmContentType: Typo Post
slug: proxmox-home-server-part-iv-hass-and-esphome
keywords:
  - Guide
  - Home Server
  - LXC
  - Proxmox
  - PVE
  - ZFS
  - Container
---

This installment will focus on installing and configuring Home Assistant and ESPHome in a Fedora LXC container using docker-compose.

## Home Assistant and ESPHome Explained

Home Assistant (HASS) is a free, open source, home automation platform. Unlike some other platforms, it keeps all your data local unless you choose to send it somewhere else. Its an excellent way of combining multiple ecosystems and devices into one place, and enables a level of automation and monitoring that you couldn't have if you were using an individual service provider for each of your devices. If you have a smart home device, from a lightbulb to a camera to your washing machine, a local sensor providing information on sound levels or environmental conditions, or even your energy providers API, chances are someone, somewhere, has enabled its integration with home assistant. What that means is that you can turn on your air purifier when your particle counter detects high pollution levels, or have your porch light turn on when you arrive home in your car, or pretty much anything else you could want. Hell, I know someone who has it flash the lights in their house when someone rings their doorbell, because it was cheaper and easier than getting one designed for the hearing impaired.

ESPHome allows you to create your own sensors or other devices based around the ESP family of microcontrollers, such as the [IKESP-Air](https://blog.calumday.co.uk/posts/2024-10-24-ikesp-air/) project from one of my earlier posts, and add them effortlessly to Home Assistant using a simple interface to create the firmware that runs them.

## Creating the Container and Installing Docker.

While its certainly possible to run Home Assistant OS in a virtual machine, I opted for the more lightweight LXC container approach again. Instead of running the full OS, we will be running docker in a Fedora container. Before we can do this, we create an LXC container similar to the one we created in [part II](https://blog.calumday.co.uk/posts/proxmox-home-server-part-ii-sshfs-nas/) of this series, though this time we use Fedora in place of Alpine. We can download the Fedora template (at the time of writing, based on Fedora 42) in the same way we did the Alpine one in part II. I like to use `HomeAssistant` as the hostname, but you can change this to you liking. Give it a Password, and an SSH key as we did earlier, and as we move through the configuration dialogue, I like to give it access to 2 CPU cores, 4GB ram, and a 32GB boot drive. All of this can be changed later, so if you find you need give it more or less resources later on, don't worry. Set a static IP as we did with the SSHFS container in part II, and remember it as you will need it later.

Once the container is created, we need to update it. At the same time, I like to make sure it stays up to date without me doing it manually. Thankfully, this is trivial in Fedora. First we update and install the DNF auto update plugin.

```sh
sudo dnf upgrade -y && sudo dnf install -y dnf5-plugin-automatic
```

Now we have the package installed, we need to configure it to reboot the container when needed.
```sh
touch /etc/dnf/automatic.conf && sudo dnf install -y dnf5-plugin-automatic && echo -e '[commands]\napply_updates = yes\nreboot = when-needed' | sudo tee /etc/dnf/automatic.conf && sudo systemctl enable --now dnf5-automatic.timer
```

That it, the system should now be up to date, and will periodically check for updates, and install them, rebooting if necessary. Now we can install docker and docker-compose, if they aren't already. I shamelessly borrowed the following command from [Alex Haydock's excellent blog](https://blog.infected.systems).
```sh
if command -v apt >/dev/null 2>&1; then source /etc/os-release && if [ "$ID" = "ubuntu" ]; then echo "Installing Docker for Ubuntu <26.04..." && sudo apt update && sudo apt install -y docker.io docker-compose-v2; else echo "Installing Docker for Debian 13+..." && sudo apt update && sudo apt install -y docker.io docker-compose; fi; elif command -v dnf >/dev/null 2>&1; then echo "Installing Docker for Fedora..." && sudo dnf install -y moby-engine docker-compose; elif command -v apk >/dev/null 2>&1; then echo "Installing Docker for Alpine..." && sudo apk --no-cache add docker docker-compose; else echo "No supported distro found. Doing nothing for safety."; fi

```

It will cleverly identify if you are on Ubuntu, Debian, Fedora, or Alpine, and install docker and docker-compose. If your OS isn't listed, it will simply give and error message informing you it cant install, and fail safely. On Fedora 42 this command should indicate that there is nothing to do, as docker and docker-compose should already be installed. If they are not, this will install them.

We can now reboot the container, and then its on to creating the Home Assistant and ESPHome docker-compose file.

## Using Docker-compose to Run Home Assistant and ESPHome

Using docker-compose to run Home Assistant and ESPHome is as simple as creating the compose file, adding some configuration files, making one small change when the container is running, and ensuring it starts again after a reboot. To start, we create the folder into which we will add the docker-compose file, and any other config files. It is appropriate to do this in the `/opt` directory, which exists to hold add-on software, or anything else that isnt a core part of the OS or installed using the OS package manager.
```sh
mkdir /opt/hass
```

Next we can add the docker-compose file:
```sh
touch /opt/hass/docker-compose.yaml
```

We then edit the file using nano:
```sh
nano /opt/hass/docker-compose.yaml
```

and add the following contents:
```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      - /opt/hass/data:/config
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped
    network_mode: host

  mosquitto:
    container_name: mosquitto
    image: "docker.io/library/eclipse-mosquitto:2"
    volumes:
      - /opt/hass/mosquitto:/mosquitto
    restart: unless-stopped
    network_mode: host

  esphome:
    container_name: esphome
    image: "docker.io/esphome/esphome:stable"
    volumes:
      - /opt/hass/esphome:/config
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped
    network_mode: host
    command: "dashboard --address '' /config"

  watchtower:
    image: 'docker.io/containrrr/watchtower'
    container_name: 'watchtower'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command: 'homeassistant mosquitto esphome'
```

Lets break this file down a little. We create 4 services, `homeassistant`, `mosquitto`, `esphome`, and `watchtower`. The `homeassistant` service will create a container of the same name, using the latest Home Assistant docker image from the official repo. It will then mount some directories on the Fedora host in the locations expected by HASS. The `restart` line tells docker that if the container should stop for any reason other than having been manually stopped by the user, it should be restarted. The `network_mode` line in each of these containers tells docker not to isolate the container from the host networking stack, allowing use to access the web interfaces of HASS and ESPHome using the IP address of the Fedora LXC container.

Next, the `mosquitto` service creates a container that runs the Mosquitto MQTT broker. This acts as a middle man between any of your devices that use the MQTT protocol to communicate and HASS itself. Thirdly, the `esphome` service starts an ESPHome container. Finally we have the `watchtower` service. This starts a container that uses Watchtower to monitor the other containers, and identify when there are changes to the images that they are based on. When an image is updated, Watchtower will gracefully stop the container and update it to use the new image. The final `commmand` line tells it what containers to monitor.

Now that we have the docker-compose file ready, we have to create a config for Mosquitto to use. First, we create the directory it expects the config file to be in, and then add the file itself.
```sh
mkdir /opt/hass/mosquitto/config && touch /opt/hass/mosquitto/config/mosquitto.config
```

Now we can use nano to edit this file:
```sh
nano /opt/hass/mosquitto/config/mosquitto.config
```

and add the contents:
```sh
persistence true
persistence_location /mosquitto/data/
log_dest file /mosquitto/log/mosquitto.log
listener 1883

## Authentication ##
allow_anonymous true
```

Crucially, this file is currently allowing unauthenticated access to the MQTT broker by any device that can pass traffic to it. This is probably a bad idea, so our next step is to add a password to it. We do this when the container is running, so having the `allow_anonymous true` line is essential for now as without it the container wont start.

We bring up the container by changinf to the `hass` directory:
```sh
cd /opt/hass
```

and then using the command:
```sh
docker-compose up -d
```
to start the containers.

Where the `-d` flag tells docker-compose to run in the background, which is essential if we want to either be able to enter the mosquitto container in the next step, or to have docker continue to run when we close the session.

In order to enable authentication for Mosquitto, we need to add a user and password. We do this by entering the running container.
```sh
docker exec -it mosquitto sh
```

Once we are working inside the container, we create the user and password we need:
```sh
mosquitto_passwd -c /mosquitto/config/password.txt hass
```

This will add a password file, and a user called `hass`. It will request you to enter a password, and confirm it. Once its done, you will be able to exit the container shell using the `exit` command.

You can check the user and password have been added:
```sh
cat /opt/hass/mosquitto/config/password.txt
```

This should give you an output of the username, and the hashed password. Now that we have set a username and password, we can edit the mosquitto.config file to use them, using the `nano /opt/hass/mosquitto/config/mosquitto.config` command we ran earlier and changing the file contents to match:
```sh
persistence true
persistence_location /mosquitto/data/
log_dest file /mosquitto/log/mosquitto.log
listener 1883

## Authentication ##
allow_anonymous false
password_file /mosquitto/config/password.txt
```

We can then restart the container to apply the changes.
```sh
docker-compose restart mosquitto
```

That completes the hard work. We should now have both the web interfaces for Home Assistant and ESPHome available at the IP of the Fedora container. HASS is accessed at http://X.X.X.X:8123, while ESPHome is at http://X.X.X.X:6052. Mosquitto should also be listening at X.X.X.X:1883.

## Onboarding HASS

The next step should be to access the HASS web interface as described above, and then fillow the [onboarding guide](https://www.home-assistant.io/getting-started/onboarding/) provided by the HASS project. After doing this, there is an additional step we need to take. We need to connect HASS to the Mosquitto MQTT broker. Navigate to the Settings page in the HASS web interface, and select the "Devices and services" option. Add a new integration, and search for "mqtt". Select the "MQTT" option, rather than any of the longer variations. On the next page, the broker can be set to either 127.0.0.1, or the IP address of the HASS container. The port should be set to 1883, and the username and password set to whatever you specified in the Mosquitto configuration. You should get a success message after clicking submit. You can test the broker is listening by following the instructions [here](https://www.home-assistant.io/integrations/mqtt/#testing-your-setup).

From here, you're ready to branch out and use HASS for whatever you'd like. There are far too many possibilities to guide you from here, but you should be able to find help and inspiration on the HASS forums, or on any number of other blogs and sites.