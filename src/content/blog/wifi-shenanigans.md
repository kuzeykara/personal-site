---
title: "Wi-Fi Shenanigans and Not Committing a Crime"
description: "An adventure about capturing handshakes, cracking passwords, and learning about local laws."
pubDate: 2026-05-04
tags: ["cybersecurity", "wireless-security", "wifi", "wpa2", "wpa3", "aircrack-ng"]
draft: false
---

Hey internet, today I want to talk about my short adventures with my new toy: an `Alfa AWUS036ACHM` Wi-Fi adapter. The idea to buy such a device came to my mind after my Wireless Security classes. I wanted to get my hands dirty because let's be real, theory is not as interesting as hacking stuff.

## Shopping!

I first checked if my Wi-Fi card supported Monitor Mode or Packet Injection and found out that it supported neither. This was bad news since I couldn't really do much of what I wanted to do, so I started exploring my options. Eventually I landed on this [repo by @morrownr](https://github.com/morrownr/USB-WiFi) on GitHub. After checking it out and doing more research, I decided to buy the aforementioned Alfa adapter. Availability, cost, and capabilities were the primary factors that led to this decision. It arrived in a few days and it was time to play!

## Hacking Loved Ones (with consent)

I found a (consenting) victim, took their phone (Samsung Galaxy A34) and turned on Hotspot. The protocol was set to WPA2, since apparently you can't do much against WPA3 (more on that later). I then connected to it with their laptop. I made them change the password so that I could try cracking it. I of course asked them to pick a very easy one so that I can use the famous `rockyou` wordlist and not have to wait for hours.

The plan had two parts essentially: capturing a WPA2 handshake and cracking the password. The first part of the plan was to use the `Aircrack-ng Suite` to enable monitor mode, scan the air and the target AP, deauthenticate a STA and capture the WPA2 handshake. The second part was to use either `aircrack-ng` from the Suite or another cracking tool like `hashcat`.

I began scanning the air and quickly found the MAC address and the channel of the hot-spot (AP), then managed to find the MAC address of the laptop connected to it (STA). Great! Time to launch a deauthentication attack and force them to reconnect so that I can capture the 4-way handshake. And... it doesn't work. I was getting almost no ACKs for the forged deauthentication packets I was sending. I had no idea why, so I started trying different ways and tools. I tried using `mdk4` and launched an authentication flood and EAPOL flood which did not work. The authentication flood showed mostly `No Response` and the EAPOL flood `EAP Start: 0` which apparently meant the injections were not working. This led to a frustrating spree of research and testing, and the reason behind it caught me off guard!

## Don't Do Crime, Kids
After lots of back-and-forth with Gemini and trial and error, I figured out the reason was that there were regulatory restrictions for the 5GHz radio band. 

`"The 5GHz band is shared with highly critical systems, including: Terminal Doppler Weather Radar, Maritime radar systems, Military tracking and communications."`

It should be harmless if you are not next to an airport or a military base, but you can't take chances.

The linux kernel itself was actually enforcing this by loading the legal parameters for your location and applying a flag called `NO-IR` (No Initiating Radiation) if the database says a 5GHz channel requires DFS (Dynamic Frequency Selection). This means your Wi-Fi card is forbidden from transmitting *anything* on that channel unless an AP explicitly gives it permission and the channel is clear.

Apparently modern Wi-Fi adapters also have regulatory limits hardcoded into their firmware. They will often just drop injected packets on DFS channels to prevent the manufacturer from being fined.

The reason why the 2.4GHz band doesn't have the same restrictions is because it is an **ISM (Industrial, Scientific, and Medical) band** - so a non-critical junk band.

There is a way to bypass these restrictions however, and it is surprisingly easy! You can move to a country without these restrictions and change your country code accordingly with `sudo iw reg set COUNTRY_CODE`. If you did that, hypothetically, everything would work.

I decided to avoid committing a crime, so I didn't forge my country code of course!

## 2.4GHz Shenanigans
From this point on, we are moving on to the 2.4GHz band to avoid the restrictions. 

And immediately, I saw improvements with `mdk4`. The AP was actually freezing and denying connections (mdk4 a), also EAP starts and associations were successfully being injected (mdk4 e). Though it was not actually kicking the already connected STAs, which was what I actually needed. 

I went back and tried `aireplay-ng` deauth, and it worked! It was sending forged deauthentication packets saying "disconnect me please, I am the laptop" and the AP removed the laptop from the network. When it reconnected, I captured the 4-way WPA2 handshake. I then used `aircrack-ng` to crack it using the classic `rockyou` wordlist. It took about 15 minutes to crack the password, which was "asdfgh00". Very weak indeed!

Below is the full attack flow:
```bash
sudo airodump-ng wlan0mon #scanning the air
sudo airodump-ng -c <Channel> --bssid <BSSID> -w wpa2_capture wlan0mon #scanning the target AP and waiting to capture the handshake
sudo aireplay-ng -0 10 -a <BSSID> -c <Client_MAC> wlan0mon # deauth
aircrack-ng -w <wordlist> -b <BSSID> <Capture_File.cap> # pwd cracking
```
## Feeling More Secure Now (WPA3)
After successfully hacking my loved ones, I decided to move on to WPA3 security. After some research and testing on my router, it seemed like the above techniques are completely useless against WPA3. The possibility of these types of attacks was actually the motivation to move on to an actually secure Wi-Fi protocol.

To prevent forging disconnects, WPA3 makes Protected Management Frames (PMF) mandatory. This ensures these critical packets cannot be spoofed. Also to prevent offline password cracking, WPA3 replaces handshakes with Simultaneous Authentication of Equals (SAE).

PMF requires a cryptographic signature called the MIC (Message Integrity Check), which is generated between the AP and each STA using their shared session keys. Even if an attacker knows the password, they can not forge a management frame (like a deauth packet). Each connection has its own temporary session key.

SAE is based on a method called the **Dragonfly Key Exchange** - which is a very cool name! During the SAE exchange the actual password is never transmitted over the air. Instead, the password is used to mathematically generate a "curve element". Since there is no verifiable hash sent over the air, an attacker cannot capture a packet and brute-force it offline. To guess a WPA3 password, the attacker must attempt a full SAE handshake with the router for *every single guess*. Modern routers should instantly detect an online attack like this and block the attacker's MAC address after just a few failed attempts.

## To Be Continued...
The next steps for me are to try an Evil Twin attack, and try Transition Mode attack techniques. I might dedicate a full blog post on WPA3 attack angles.

In conclusion: don't do crime, and use WPA3. See you soon!