# Investigating longevity and TLS behaviours in consumer IoT devices
#### by Nic Watson
#
This repository contains configuration files and setup instructions for reproducing the procedure and results of my research project, _[Investigating longevity and TLS behaviours in comsumer IoT devices](https://report-hub.scs.carleton.ca/projectPage/2805/)_, completed in 2024 as a capstone Honours project for the Bachelor of Computer Science (Cybersecurity stream) program. 

An updated, full-text version of the report is forthcoming on Overleaf.

## Abstract

Many residential "smart" Internet of Things (IoT) products rely on cloud services, but product designers appear to give little thought to the long-term life of devices after cloud endpoints are retired. This contributes to consumer dissatisfaction and a growing global e-waste problem. One factor affecting longevity is the Transport Layer Security (TLS) architecture used for device-cloud connections. Given the parsimonious nature of IoT platforms, it is difficult to upgrade in-service devices to meet evolving security needs. Additionally, X.509 certificates used for TLS authentication expire and must be periodically renewed, or clients may be unable to connect.

This study contributes to ongoing work on the IoT longevity problem by applying black-box analysis to TLS behaviours of selected consumer IoT devices, to determine possible future bugs and failure states. Experiments found a diversity of TLS versions and implementations in use, and differing approaches to certificate validation. Furthermore, by subverting Network Time Protocol usage, I prompted devices to accept false future time references. This elicited poorly-defined behaviours and uncovered evidence that TLS stacks in some devices are susceptible to the "2038 problem." I use these data to discuss tradeoffs between longevity and security, and suggest changes to design practices to mitigate longevity challenges.

## General setup notes

I configured each IoT device to connect to a 2.4 GHz 802.11g Wi-Fi access point (AP) hosted by a Raspberry Pi 4 Model B (2018), running Raspberry Pi OS "Legacy" (64-bit), with kernel version 6.1, based on Debian 11 "bullseye." The AP was configured for WPA2-PSK security. I installed the following additional software packages, which are available through the default apt repositories.
* __hostapd v2.9__ for wireless AP configuration and management.
* __dnsmasq v2.85__, to allocate IP addresses to wireless clients using the Dynamic Host Allocation Protocol (DHCP); and to provide Domain Name System (DNS) lookup services to clients.
* __iptabes-persistent__ v1.8.7 as a convenience for making network routing rules created with the operating system's native iptables utility persist through reboots.
* __Wireshark v3.4.10-0+deb11u1__ to capture and inspect network packets that IoT devices exchange with 
* __chrony v4.0__ to provide configurable NTP services to wireless clients.

Since the Raspberry Pi 4 has both wireless 802.11 (wlan0) and wired Ethernet (eth0) interfaces, I connected the Pi's Ethernet port directly to my Internet gateway router, and used an iptables rule to bridge the interfaces, allowing wireless clients to access the Internet uplink as if the Pi were itself an ordinary home network router (see figure, below). To reduce network complexity and make it easier to distinguish relevant experimental data in packet traces, I avoided using the remote SSH option for an interactive shell to the Pi, instead opting for a traditional physical interface with a wired USB keyboard and mouse, and a monitor connected to the Pi's HDMI output. 

![iot_setup](https://github.com/user-attachments/assets/e8a36313-85b2-47ec-a182-627a36c6fe50)

Furthermore, as the diagram above shows, I operated my user control interface (UCI) equipment—a smartphone running Android 11—on a separate wireless network, hosted by a separate physical device, except during some setup operations when the vendor's UCI app required a direct connection to the same Wi-Fi network that the device was going to use. The diagram depicts the smartphone reaching the Internet through a separate gateway from the one used by the Raspberry Pi; in fact, they were the same physical gateway, but this makes no functional difference, since—except during device setup—communications between the device and UCI are always indirect and mediated by Internet cloud services. I have shown this connection separately in the diagram for clarity, to emphasize that the device-to-cloud and UCI-to-cloud connections are conceptually separate.

Contrary to recommendations of the Raspberry Pi hobbyist blogosphere,  I used the Debian 11-based edition of Raspberry Pi OS (branded as "Legacy" in 2024), rather than the latest based on Debian 12 "bookworm." The newer edition ships with the Network Manager service, which provides a low-hassle, novice-friendly method to set up the device as a wireless AP. However, Network Manager includes its own built-in version of dnsmasq to provide DHCP and DNS services. This minimal, integrated dnsmasq service is not amenable to the kind of configuration customizations required to interfere with DNS lookups, as needed for my experiments (see 3.2.2, below). Making custom entries in the operating system's hosts file is also not a suitable alternative, because (unlike dnsmasq) it does not support wildcard subdomains.

However, installing dnsmasq as a full, separate service will interfere with the operation of Network Manager. I encountered these issues in my early experimentations, and chose to sidestep them by installing the older OS version, wherein Network Manager is disabled by default, and the traditional methods of using hostapd and dnsmasq remain effective.

## About NTP spoofing

To validate TLS certificates, one of the factors that the client (e.g. IoT device) must check is the certificate's expiry date. For this, the device needs some sense of the current date and time, but most consumer IoT devices do not have reliable internal clocks. Instead, many of them contact a public Network Time Protocol (NTP) server to receive a current timestamp. This provides a convenient way to trick an IoT device into using the wrong date and time.

To simulate an IoT device operating in the future, receiving a legitimate NTP time signal reporting that future date, I configured the Linux service chrony to run as an NTP server daemon. Chrony may be set to a local mode, in which case it uses only the system clock as its time reference and does not contact any upstream servers; it can further run in manual mode, allowing the user to set the system clock on the command line.

To prepare for a time manipulation experiment, I ensure that the target device is powered off. Then I start the chrony service, set it to local and manual mode, and enter a date and time past the expiry of the certificate I have chosen to target (e.g. "1 Jan 2045 00:00"). This ensures that network packets addressed to the Raspberry Pi on the standard NTP port 123 receive a response from chrony with the doctored time. The next step is to ensure that devices use my NTP server instead of a real one on the Internet. Before making an NTP request, a device will likely issue a Domain Name System (DNS) request, in which it asks a network DNS service to provide the IP address corresponding to a given FQDN. This is because it knows the FQDN, but not the IP address, of a suitable NTP server, and it needs the IP address to actually connect. By checking the previously-collected baseline packet trace for DNS queries, I can find the FQDN(s) of the NTP server(s) with which the device intends to communicate. I then enter these FQDNs as mappings in my dnsmasq file, directing them to the IP address of the Raspberry Pi itself.
 
Finally, I start a new Wireshark capture, restore power to the device, and allow it to rejoin the network. This time, it will end up asking my own NTP server for the time, and it will receive and accept the false future time that I have configured. The fake NTP response will show up in the Wireshark capture, providing confirmation in its timestamp field that the manipulation has worked. The screenshot below shows a capture experiment in which an IoT device (smart plug) accepted the Pi's local network address as the IP address for uk.pool.ntp.org, and then accepted the fake NTP server's claim that the date was July 11, 2064. The diagram (below the screenshot) compares the flow of DNS and timestamp information between the normal scenario with no interference, and the NTP-spoofing setup.

![ntp_spoof_wireshark](https://github.com/user-attachments/assets/311e213d-99e6-4cda-b551-bb4c6f5efa85)

![ntp_spoof_diagram](https://github.com/user-attachments/assets/86bca108-fb9c-4740-b932-d6b5c6098f90)

Note that a device could defeat this particular technique, as I have implemented it, by using a secure DNS protocol such as DNSSEC or DNS over TLS, or with a pre-configured IP address. However, the more general class of attack still works, with minor modification, as long as the device uses unencrypted, unauthenticated NTP.

## Setup instructions

This repository provides configuration files for the software tools used in my experiments. In some cases, these are complete files, which can replace the default ones on the target system. In other cases, I provide only content that must be _added_ to an existing file, in which case the version in this repository has the suffix `.add` in its name (copy/paste its contents into the existing conf, or append with `cat`). Finally, for smaller changes, I simply provide the necessary information inline as part of this readme.

### Basic setup

Software configuration for a basic Raspberry Pi wireless access point setup on Raspberry Pi OS based on Debian 11 "bullseye" is adapted from [Fromaget, n.d](https://web.archive.org/web/20240827164651/https://raspberrytips.com/access-point-setup-raspberry-pi/). This is one of many hobbyist blog articles offering substantially similar instructions (see e.g. [Pounder, 2022](https://web.archive.org/web/20240211044549/https://www.tomshardware.com/how-to/raspberry-pi-access-point)).

#### WLAN Country

For regulatory compliance, the radio interface must be configured to use the legally-designated frequencies in the country of use, i.e. Canada for these experiments. (IoT devices used in experiments were also acquired in Canada.) The country can be set using the _raspi-config_ interactive configuration tool (run as superuser, i.e. `sudo raspi-config`), under `Localisation Options > WLAN country`.

#### Configuration: hostapd in /etc/hostapd/hostapd.conf

The hostapd service handles physical- and link-layer configuration for the Wi-Fi access point. My explanatory comments in the provided config are based on information gleaned from the [Linux Wireless Wiki](https://web.archive.org/web/20240802051742/https://wireless.wiki.kernel.org/en/users/Documentation/hostapd) and [Gentoo Linux Wiki](https://web.archive.org/web/20240716224113/https://wiki.gentoo.org/wiki/Hostapd).

For hostapd v2.9, one must also modify `/etc/default/hostapd` and add the following line:
```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

This tells the startup scripts where to find the configuration file. Note that future versions of hostapd will likely require a different procedure in place of this step. 

Finally, hostapd needs to be unmasked and enabled as a system service:
```
sudo systemctl unmask hostapd
sudo systemctl restart hostapd
```

#### Configuration: dnsmasq in /etc/dnsmasq.conf

The dnsmasq daemon provides DHCP and DNS services to connect clients. Once it is installed, the configuration file should already exist as `/etc/dnsmasq.conf`. Some additional lines need to be added to the end of the file. See `/etc/dnsmasq.conf.add` in this repository (the contents of that file can be copy/pasted in the existing conf, or appended with `cat`). Annotations are my own, based on documentation from the [Debian Wiki](https://web.archive.org/web/20240731092448/https://wiki.debian.org/dnsmasq) and [ArchWiki](https://web.archive.org/web/20240822145001/https://wiki.archlinux.org/title/Dnsmasq).  

#### Configuration: dhcpcd in /etc/dhcpcd.conf

I also configured the Raspberry Pi's own DHCP client, which runs as dhcpcd ("DHCP client daemon") and is included in the OS distribution. A few lines must be added to the conf file. See/append `/etc/dhcpcd.conf.add` in this repository.

#### Configuration: Network bridge

In my setup, the Raspberry Pi serves IoT devices on wlan0, and accesses upstream Internet on the wired eth0 interface. In order to pass traffic through and let IoT devices access the internet, I had to configure forwarding of network packets between these interfaces.

First, I uncommented the line in /etc/sysctl.conf to enable packet forwarding:
```
net.ipv4.ip_forward=1 
```

Next, I added an iptables rule to the Network Address Translation (NAT) table using the command:
```
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

I used _iptables-persistent_ to make the rule persist when the Raspberry Pi was rebooted. This is accomplished by issuing the following command, after installing iptables-persistent:
```
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

This uses the built-in utility iptables-save to print all rules to standard output which is then piped to the text file `/etc/iptables/rules.v4`. (The use of `tee` has the effect of printing the output to the terminal at the same time.) The _iptables-persistent_ utility will automatically restore rules from this file at boot, and is equivalent to manually using `iptables-restore < /etc/iptables/rules.v4` every time.

#### (Re)starting network daemons

After altering configuration files, I (re)started hostapd and dnsmasq:
```
sudo systemctl restart hostapd
sudo systemctl restart dnsmasq
```
Rebooting the system will also restart these services automatically.

#### Wireshark usage

In order to intercept packets on wlan0, Wireshark should be launched as superuser.

### Setting up the NTP manipulation technique

The framework for providing clients with fake NTP responses consists of two components:
*	_dnsmasq_ replies to the client's DNS query for an NTP server, giving the Raspberry Pi's own IP address (192.168.42.10) as the response.
*	The chrony daemon (_chronyd_) responds to the client's NTP query with a timestamp from its own reference source.

#### Configuring dnsmasq for NTP manipulation

Address mappings for each time server FQDN a client may use need to be added to `/etc/dnsmasq.conf`, so that dnsmasq replies with the Pi's IP address. The FQDNs are determined by first allowing the devices to run normally, and examining the Wireshark packet traces for DNS requests from client devices regarding specific NTP servers.

Each NTP server domain name gets its own line in the configuration file, using the `"address="` directive. Prefixing a domain name with a dot will match any subdomain. For the devices used in this study, address mappings added to the configuration file are as follows:
````
address=/.ntp.org/192.168.42.10
address=/ntp.aliyun.com/192.168.42.10
address=/time.windows.com/192.168.42.10
address=/.nist.gov/192.168.42.10
````
After modifying dnsmasq.conf, it may be necessary to restart dnsmasq (`sudo systemctl restart dnsmasq`). After making changes here, I also power-cycled connected IoT devices to force them to issue new DNS requests.

#### Configuring chrony for NTP manipulation

The configuration file is `/etc/chrony/chrony.conf`. To set up this file, first I commented out three lines that _chrony_ uses to find time sources (see below). Lines can be commented out by prefixing them with '#'.

```
# Use Debian vendor zone.
#pool 2.debian.pool.ntp.org iburst

# Use time sources from DHCP.
#sourcedir /run/chrony-dhcp

# Use NTP sources found in /etc/chrony/sources.d.
#sourcedir /etc/chrony/sources.d
```

This ensures that I have full control over chrony's sense of time.

Next, I added the following three lines to the file (annotations are my own, based on chrony's official documentation ):

```
# allow all : let any client receive NTP services from chrony on this machine
allow all

# local : chrony will act as an NTP server on a local network, without needing upstream synchronization with an accurate time source.
# Option:  stratum 2  : the stratum number tells clients how many network hops the NTP server is from a direct connection to an accurate time source. Here we rather improbably claim to be stratum 2. NTP clients use this to find the best possible time source but this should not make any practical difference to IoT devices in these experiments.
# Option:  distance 0.1  : the distance is a metric that the NTP server reports to clients as a rough measure of how far (in seconds) it is expected to deviate from an accurate time source. A low number like 0.1 means that clients are unlikely to reject the server as too inaccurate.
local stratum 2 distance 0.1

# manual :  enables using the chronyc settime directive to manually set the clock
manual
```

After changing configuration, I restarted the chrony service: `sudo systemctl restart chrony`

#### Using chronyc to change the reported time
_chronyc_ is chrony's command interface. Invoking the program with no arguments will launch an interactive terminal. It must be launched as superuser.

In the interactive terminal, I changed the date and time as needed, using the _settime_ command. UTC time references are entered in the format:
```Month Day, Year HH:MM:SS ```

For example:  `Aug 3, 2024 16:40:00`

Other formats may also work. Including a time of day is optional (the time reference will default to midnight); if a time of day is given, including seconds is also optional. 

Using _settime_ has the effect of changing the system clock. If the system time zone is set, the system clock will automatically adjust for time zone offset. It may take some moments for the system interface to show the clock change. However, chrony will immediately start sending the new time to NTP clients, which means I can change the time in the middle of an ongoing connection that I am monitoring through Wireshark. There is no need to quit chronyc or restart the chrony service after using _settime_.


