---
title: "I Turned an Old Raspberry Pi 3B into a Live Aircraft Tracker"
date: 2026-10-28T11:04:23Z
draft: false
description: "How I reused a Raspberry Pi 3B and RTL-SDR Blog V3 to receive live aircraft transmissions, share ADS-B data and alert me to military and unidentified targets."
tags: [ads-b, raspberry-pi, rtl-sdr, radio, homelab]
---

The first military aircraft I watched appear live on my own map was **DESTRO41**, an RAF Eurofighter Typhoon. By then, the receiver had quietly recorded several other military aircraft without me noticing.

That moment made the project feel different from opening a commercial flight-tracking app. The aircraft on the screen had been received directly by a small antenna in my home, decoded by an old Raspberry Pi 3B and plotted on a map running on my own network.

I built the receiver with an [RTL-SDR Blog V3](https://www.rtl-sdr.com/buy-rtl-sdr-dvb-t-dongles/) and the prebuilt [ADSB.im](https://adsb.im/) image. It now runs headlessly over Wi-Fi, shares reception data with [adsb.lol](https://adsb.lol/) and sends me Gotify alerts when it detects military or unidentified aircraft.

## Hardware and cost

I already owned the Raspberry Pi 3B, microSD card, power supply and network equipment. The main purchase was an RTL-SDR Blog V3 kit for about **£51**. It included the USB receiver, a dipole antenna, coax and mounting hardware.

A stable 5 V supply matters on a Pi 3B, especially with a USB radio attached. I connected the SDR directly to the Pi and kept the antenna cable as short as practical. If I move the receiver into the loft, I will extend the mains side rather than depend on a long, poor-quality micro-USB lead.

## How ADS-B reception works

Many aircraft broadcast their identity, altitude, position and movement using Automatic Dependent Surveillance–Broadcast, or ADS-B. In the UK and much of Europe, the main frequency is **1090 MHz**.

The RTL-SDR only receives; it does not transmit. Software on the Pi decodes the broadcasts, plots aircraft on a local map and forwards reception reports to a feeder network. What appears on my map depends on what the antenna can hear, so aircraft altitude, terrain, buildings, antenna position and cable loss all affect the result.

## Preparing the antenna

At 1090 MHz, a quarter wavelength is about **6.9 cm**. I extended each half of the supplied dipole to roughly that length and mounted it vertically, with one element pointing up and the other down.

The receiver worked indoors, but antenna position made the biggest difference to range. A higher position with a clearer view of the horizon should outperform almost any software adjustment. Thick walls, metal objects and nearby electronics can all weaken reception, while long coax runs introduce additional loss at 1090 MHz.

## Installing ADSB.im

I used the prebuilt 64-bit ADSB.im image for Raspberry Pi models 2, 3, 4 and 5 instead of assembling the decoder, map and feeder containers manually. I checked the downloaded `img.xz` archive, carefully confirmed the microSD card's device name and then wrote the image to it.

That device check is worth taking seriously: writing an image destroys the existing contents of whichever disk is selected. I waited for buffered writes to finish and safely ejected the card before moving it to the Pi.

### The headless Wi-Fi fallback

I initially tried to preload the Wi-Fi configuration on the boot partition. It did not connect on the first boot, so ADSB.im created its temporary setup hotspot instead.

I joined that network from a phone, opened the setup page and entered the normal 2.4 GHz Wi-Fi details. The Pi then joined the home network without needing a monitor or keyboard. The fallback made the failed preconfiguration an inconvenience rather than a blocked installation.

## The first live results

Once the radio was detected, the dashboard began showing messages, positions and aircraft. The bias tee remained off because the bundled dipole is passive, and the decoder reported no dropped SDR samples.

![Sanitised ADSB.im dashboard showing receiver and feeder health](/images/posts/adsb/adsb-feeder-dashboard.webp)

*The ADSB.im dashboard confirmed that the decoder was receiving aircraft and sending data to the configured feeder. The private device name has been removed.*

I left the receiver running overnight in an ordinary indoor position. It stayed online without USB faults, container restarts or memory failures. During that initial run it decoded **more than 429,000 valid messages**, produced **more than 18,000 positions** and built **1,778 tracks**. Its best range was about **77 km (48 miles)**.

Those are not spectacular figures for a roof-mounted ADS-B antenna, but that was not the point of the first test. It proved that the old Pi, SDR, image, Wi-Fi connection and data feed all worked together reliably.

## The local aircraft map

The included tar1090 interface turns the received messages into a responsive local map. It shows aircraft identity, altitude, speed, track, signal source and distance from the receiver. Its range outline also makes the effect of walls and terrain visible: indoor reception is rarely symmetrical.

![Sanitised tar1090 map showing live aircraft and the receiver's coverage](/images/posts/adsb/tar1090-coverage-map.webp)

*Live aircraft received by the Pi. The receiver marker and surrounding area are deliberately concealed; the image reveals only the broad region.*

The map is available only on my local networks. Household devices can reach a restricted map view, while administration remains limited to a trusted management device.

## Sharing data and using MLAT

I configured the receiver to feed [adsb.lol](https://adsb.lol/). The normal ADS-B feed has remained healthy.

MLAT, or multilateration, is an additional process that estimates an aircraft's position by comparing the arrival time of its transponder messages at several receivers. It has occasionally disconnected and reconnected on the Pi 3B, but that does not stop direct ADS-B reception. The older Pi is adequate for this project, although its timing is less ideal than newer hardware for MLAT.

I kept the number of feeder services modest and left this RTL-SDR dedicated to ADS-B. A single tuner cannot listen to 1090 MHz aircraft transmissions and an unrelated radio band at the same time. I would use a second receiver if I wanted to monitor AIS, radiosondes or other signals continuously.

## Military and unidentified-aircraft alerts

Seeing DESTRO41 live made me check the receiver's stored history. It had recorded four military airframes in total at that point, including the Eurofighter, a Canadian military aircraft and two RAF training aircraft.

The live map later caught two UK military Eurofighters: **DESTRO41** (ZK379) and **DESTRO42** (ZK367). Both were shown at 14,000 feet and positioned through MLAT rather than a directly broadcast ADS-B position.

![Aircraft panels identifying DESTRO41 and DESTRO42 as military Eurofighters](/images/posts/adsb/military-typhoons.webp)

*Two Eurofighter Typhoons received by the Pi. The source screenshots were cropped to retain the aircraft identities while excluding the map and location-related telemetry.*

I added a small watcher that reads the receiver's aircraft data every 15 seconds and sends a [Gotify](https://gotify.net/) notification when it finds something interesting:

- aircraft marked as military trigger an immediate alert;
- unidentified targets trigger after remaining unidentified for 60 seconds;
- repeat notifications for the same aircraft and category are suppressed for 12 hours; and
- each alert includes the available ICAO address, callsign, registration, type, altitude and tracking source.

Here, **unidentified** has a precise and fairly ordinary meaning: the receiver has a transponder target but no callsign, registration, aircraft type or description for it. It does not mean the receiver has discovered an extraterrestrial craft. Sometimes the missing identity appears a little later, which is why the watcher waits before alerting.

## Network and security

The receiver sits on an isolated device network rather than the same network as my main computers. The router initially blocked management access across that boundary, so I added a narrow rule for one trusted computer instead of weakening isolation for every device.

I also:

- assigned the Pi a fixed private address outside the automatic allocation pool;
- created a non-root administrator account;
- enabled SSH public-key authentication;
- disabled password and root SSH login;
- restricted access with router and host firewall rules;
- enabled automatic security updates without automatic reboots; and
- confirmed that the receiver had no internet-facing port forwards.

The map needed to remain convenient on phones and tablets. I exposed a restricted, map-only path to the isolated household network instead of granting those devices general access to the Pi or trusted LAN.

Private addresses, hostnames, Wi-Fi details and the receiver's exact map location are intentionally absent from this article and its screenshots.

## Monitoring the receiver

A small health collector records a sample every five minutes. It checks receiver and feeder state, decoder statistics, Pi temperature, memory use, load, Wi-Fi, time synchronisation, container state and recent MLAT events.

The first version broke after I hardened SSH because it still expected root access. The public statistics remained available, but the private system checks failed. I changed it to use the restricted administrator account and limited `sudo` to the Docker checks that actually needed it.

That was a useful reminder: a security improvement is not complete until the monitoring and recovery paths have been tested again.

## Problems and fixes

| Problem | What fixed it |
|---|---|
| Preloaded Wi-Fi settings did not connect | I used the temporary ADSB.im setup hotspot. |
| The management computer could not reach the isolated Pi | I added one narrow router rule instead of removing network isolation. |
| The Pi's address changed | I assigned a fixed private address outside the DHCP pool and updated the related rules. |
| Root SSH access was still partly enabled | I checked the effective OpenSSH configuration, corrected the ordering and verified that root login was rejected. |
| Isolated tablets could not open the map | I added a restricted map-only route rather than general network access. |
| MLAT occasionally disconnected | I confirmed direct ADS-B remained healthy and that MLAT later reconnected. |
| Monitoring failed after SSH hardening | I moved it to the restricted account and narrowly scoped the required `sudo` access. |

## Is the Raspberry Pi 3B still enough?

For this job, yes. The Pi 3B is old, but it can run the decoder, local map, modest feeder workload, health checks and alert watcher without becoming the bottleneck.

Stable power, storage, Wi-Fi and antenna placement matter more than replacing it simply because a newer Pi exists. A newer model would provide more headroom and might improve MLAT consistency, but the current receiver does not need an upgrade to perform its main task. I would avoid loading it with unrelated, CPU-heavy services.

The same applies to replacing the RTL-SDR Blog V3 with a V4. For a dedicated 1090 MHz receiver, antenna height, cable loss, local interference and filtering are likely to matter more than changing between those dongles.

## What I would improve next

1. Move the antenna to a higher, safely secured position with a clearer view of the horizon.
2. Keep the coax short and away from large metal objects.
3. Compare message rate, aircraft count and maximum range before and after moving it.
4. Check temperature, voltage and Wi-Fi stability after relocation.
5. Consider a 1090 MHz filter or low-noise amplifier only if the new position reveals a problem they would solve.
6. Add a separate RTL-SDR if I want another radio service running continuously.

## Was it worth doing?

Yes. The Pi boots without a monitor, reconnects to Wi-Fi, detects the SDR, decodes aircraft, updates its local map, shares data and alerts me when it sees selected targets. It achieved all of that with hardware I already owned apart from the £51 radio kit.

The indoor test proved the complete system. The first live military detection made it memorable. The next meaningful upgrade is not a new computer or another software rebuild; it is simply giving the antenna a better view of the sky.
