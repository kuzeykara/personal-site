---
title: "Building My Malware Analysis Lab (and Debugging INetSim DNS Failure)"
description: "A story of setting up a malware analysis lab and spending way too much time on a compatibility issue."
pubDate: 2026-03-24
tags: ["cybersecurity", "malware-analysis", "remnux", "flare-vm", "inetsim"]
draft: false
---

Hey internet, in this post I will share my experience - and struggles - trying to set up a malware analysis lab. A few days ago I attended my first security convention, [M0lecon2026](https://m0lecon.it/), organized by the amazing student team [pwnthem0le](https://pwnthem0le.polito.it/). While listening to the talks, my mind occasionally wandered off. I started thinking about what I actually want to focus on in the huge field that is cybersecurity. After some consideration, I realized that the answer was actually pretty clear. I wanted to work with malware! I was always fascinated by how people could do malicious things by using tools that were built with good intentions. I can say that I owe my fascination with malware to [@danooct1](https://www.youtube.com/@danooct1) and [@vxunderground](https://x.com/vxunderground) (honorable mention: [MalwareTech](https://malwaretech.com/)).

Naturally, after this realization I started researching what I can do to learn more about malwares. I did some "old-school" internet research, meaning no AI, and took notes about my findings. It was clear that I needed to set up a malware analysis lab. There were dedicated distros pre-configured for malware analysis with a lot of pre-installed tools. There were also external tools I could look into.  I then fed my notes to Cursor (Sonnet 4.6) and wrote a detailed prompt detailing my status and my approach. I asked it to do a deep internet research about my notes, create a detailed malware analysis lab setup guide based on its findings, while citing the sources and giving suggestions for further reading (articles, books, threads, etc.). It generated a Markdown file of about 800 lines. After reading it and asking for clarifications and more details for some parts, the line count increased to ~1000 lines.

For starters, I mounted my external SSD which was my main VM storage and verified Virtualbox was still working (Arch btw). The plan was to install two separate VMs, REMnux (Ubuntu-based) and FLARE-VM (Windows), and utilize a fully isolated internal network. Host-Only networking would create a `vboxnet0` interface on the host that bridges host and VMs. This means that malware could potentially reach the host, any routing mistake on the host could leak traffic to the LAN! Internal network avoids all of this. The VirtualBox kernel module handles traffic entirely in the hypervisor layer.

I was going to use INetSim to simulate the "Internet" for FLARE-VM to use.

| VM       | IP            | Role                                                        |
| -------- | ------------- | ----------------------------------------------------------- |
| REMnux   | `10.0.0.1/24` | DNS, HTTP, SMTP, FTP (INetSim); gateway for malware traffic |
| FLARE-VM | `10.0.0.2/24` | Analysis workstation; default gateway → `10.0.0.1`          |
## REMnux
"[REMnux](https://remnux.org/) is a Linux toolkit for reverse-engineering and analyzing malicious software."

The easiest way to set up REMnux is to download the prebuilt VA (Virtual Appliance) and import it into a hypervisor. In my case, there was a dedicated OVA for VirtualBox. I downloaded it and imported it without much hassle. I then configured it with the settings below:
```bash
VBoxManage modifyvm "REMnux" --memory 4096 --cpus 2 --vram 64 --clipboard-mode disabled --draganddrop disabled --usb off --audio-driver none --audio-enabled off
```
The memory and disk settings were stated in the docs as sufficient for most people. Clipboard is disabled because apparently some malware/tools "touch" the clipboard and disabling it entirely avoids it being a bridge to the host! The reasoning is the same for disabling drag and drop. Turning off USB was more apparent to me, since it is a common target for malware.

The first time I booted the VM I thought it was stuck but apparently it was normal for the first boot. *systemd* was waiting until it considers a network interface "online", which eventually timed out after ~2 minutes. After booting, I used Netplan to configure the network with the below settings:
```yaml
network:
    version: 2
    ethernets:
        enp0s3:
            dhcp4: false
            addresses:
                - 10.0.0.1/24
```

I then edited the INetSim config file, with the following settings:
```ini
# Bind to all interfaces
service_bind_address 0.0.0.0

# Respond to all DNS queries with this IP (REMnux itself)
dns_default_ip 10.0.0.1

# Start core services
start_service dns
start_service http
start_service https
start_service smtp
start_service ftp
start_service irc
start_service pop3
```

"INetSim is a software suite for simulating common internet services in a lab environment, e.g. for analyzing the network behaviour of unknown malware samples."

After this, I *thought* I was done with REMnux, and happily moved on to setting up FLARE-VM.

## FLARE-VM
[FLARE-VM](https://github.com/mandiant/flare-vm) is "a collection of software installation scripts for Windows systems that allows you to easily setup and maintain a reverse engineering environment on a VM."

FLARE-VM is essentially a Windows VM that is configured and is curated for malware analysis. It uses [Chocolatey](https://chocolatey.org/) and [Boxstarter](https://boxstarter.org/); which are a package management system (essentially zip files) and an automation tool for environment setups. I had to choose a supported Windows version, and I decided to follow Cursor's suggestion which was **Windows 10 Pro LTSC (Long-Term Servicing Channel)**. The reasoning was that Defender disabling process was simpler and more reliable, and FLARE-VM is fully supported and tested on Windows 10. Also "no automatic feature updates that can break your analysis toolchain mid-session". Perfect!

After obtaining the ISO (and verifying it), I created the VM "shell" on VirtualBox. I then modified it with the following command:
```bash
VBoxManage modifyvm "FLARE-VM" --memory 16384 --cpus 4 --vram 128 --graphicscontroller vmsvga --clipboard-mode disabled --draganddrop disabled --usb off --audio-driver none --audio-enabled off --boot1 dvd --boot2 disk --boot3 none --boot4 none
```

Since my host machine has 30GB of RAM, I decided to give this VM a lot of memory. Since this is the "main" VM I will use for analysis, I also increased its VRAM and CPU cores. I set the booting priority like this to then "attach" the ISO on the DVD to install Windows. The justification for the rest of the settings are the same as before.

After installing Windows 10 (and declining all the usual Windows prompts), I went on to disable Windows Defender completely. I booted into Safe Mode and edited the `Start` values of  the following registery keys: 
```
HKLM\SYSTEM\CurrentControlSet\Services\WdFilter
HKLM\SYSTEM\CurrentControlSet\Services\WdNisDrv
HKLM\SYSTEM\CurrentControlSet\Services\WdNisSvc
HKLM\SYSTEM\CurrentControlSet\Services\WinDefend
```

Then I Disabled Windows Update, Windows Firewall, and Windows Telemetry; and finally took a snapshot before installing FLARE. I followed the tutorial on its GitHub README for installation. The process was very easy, but it took ~2 hours to complete.

I then configured the static IP for it and set the Preferred DNS server and the Default Gateway as 10.0.0.1. It was time to test the internal network. I booted REMnux and ran INetSim. Then on FLARE I pinged 8.8.8.8 and received no response. Good. I pinged 10.0.0.1 and got responses back, great! Then I ran `Resolve-DnsName google.com` and got a *ResourceUnavailable* error. This little error sent me into an unnecessarily long and very frustrating debugging journey.

## DNS Not Working!
First things first, I checked the Windows side. Running `Get-DnsClientServerAddress -AddressFamily IPv4` showed that the Ethernet adapter had an empty `ServerAddress` field! DNS was not set, so I fixed it by running `Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "10.0.0.1"`. I tried to apply it with netsh but got "The service has not been started". I checked the services and saw that `Dnscache`, which is the Windows DNS Client, was stopped. It probably was caused by one of FLARE-VM's debloating scripts. I tried to enable it with `Set-Service` but I got "Access is denied". Tried it again on a high privilege (Administrator) Powershell but got the same result.

To enable it I had to edit the registry `HKLM\SYSTEM\CurrentControlSet\Services\Dnscache`. I set the Start value as 2 (Automatic) and rebooted. I tried Resolve-DnsName again and... not fixed. How? Why? Well, the problem had to be with REMnux.

INetSim was confirmed running. Its log stated `dns_53_tcp_udp started`, but `sudo ss -ulnp` showed nothing on port 53. Very frustrating. INetSim had all its TCP listeners up, but UDP/53 was completely unbound! Apparently the DNS child process was forking and immediately dying. Here is the INetSim log snippet:
```
...

Forking services...
* dns_53_tcp_udp - started (PID 1877)
Attempt to start Net::DNS::Nameserver in a subprocess at /usr/share/perl5/INetSim/DNS.pm line 69.

...
```

At that point it seemed like the main cause had to do something with compatibility. I decided to connect REMnux to the internet and run `remnux install` to get the latest system updates. Hopefully the issue was patched and everything would work. It wasn't patched. At least not entirely.

It turns out the root cause was a compatibility break between INetSim 1.3.2 and `libnet-dns-perl`. INetSim's `DNS.pm` calls `$server->main_loop`, a method that was deprecated in newer `Net::DNS` releases and redirected to wrap `start_server()` internally. But `start_server()` contains a subprocess guard it stores the PID at object construction time and `croak`s if the current PID differs. Since INetSim constructs the `Net::DNS::Nameserver` object in the parent process and then forks a child to run the DNS service, the child **always has a different PID**, and `Net::DNS` kills it every single time with `"Attempt to start Net::DNS::Nameserver in a subprocess"`. 

Debian's packaging team patched this in December 2025 (package `inetsim 1.3.2+dfsg.1-6`, patch `Remove-deprecated-Net-DNS-main_loop-call.patch`). The fix is a one-liner in `/usr/share/perl5/INetSim/DNS.pm` around line 69, courtesy of Cursor:
```perl
# Change:
$server->main_loop;

# To:
$server->start_server(0);
```

I applied the fix, but UDP/53 still stayed unbound, so my issue was not just an unpatched main_loop call. The subprocess guard fires regardless of whether you call `main_loop` or `start_server()` directly... At that point the simplest, most reliable solution was to just use a separate tool for DNS! Everything else was working fine, so why bother.

I decided to use `dnsmasq` as a standalone DNS server, redirecting all queries to `10.0.0.1`, while leaving INetSim to handle everything else (HTTP, HTTPS, SMTP, FTP, IRC, etc.).

## Conclusion
It took a whole day (a lot of hours) for me to fix this very avoidable and simple problem. I think I have learned my lesson: don't waste time trying to fix compatibility issues if there are easy alternatives. Be smart.

Thank you for reading, and see you soon!