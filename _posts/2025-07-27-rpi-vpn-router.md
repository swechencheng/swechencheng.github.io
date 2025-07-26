---
title: "Super cheap: A Raspberry Pi VPN router for IoT devices"
date: 2025-07-27T16:51:19+01:00
categories:
  - blog
tags:
  - Embedded Linux
  - Raspberry Pi
  - Networking
  - Geofence
  - Tailscale
  - VPN
  - robot vacuum cleaner
  - IoT device
---

# **DISCLAIMER**

- Educational & Hobby Use Only

  The content on this blog is provided solely for educational, informational, and hobby purposes. I have no intention of infringing upon or circumventing any manufacturer’s terms of service, license agreements, or intellectual property rights.

- No Warranty

  All information is provided “as is,” without warranty of any kind, express or implied. I make no representations or warranties regarding the accuracy, completeness, reliability, or suitability of the information for any purpose.

- Use at Your Own Risk

  Any actions you take based on the information presented here are strictly at your own risk. I shall not be held liable for any damages, losses or other liabilities that may arise from your use of, or reliance on, this blog.

- Non‑Commercial

  This blog does not offer any commercial services, paid consultancy, or legal advice. All findings are shared freely and without charge, and no part of this material may be sold, redistributed, or repackaged for commercial gain.

- Respect Manufacturer Terms

  You are solely responsible for ensuring that any experiments or modifications you undertake do not violate the manufacturer’s terms of service, local laws, or regulations in your jurisdiction.

- No Legal Advice

  Nothing on this blog constitutes legal or professional advice. If you require legal guidance regarding geofencing, device usage, or related topics, please consult a qualified attorney.

# Background

When we visited our family in another country, my wife purchased a really advanced robot vacuum cleaner which can also mop the floor as well as cross the door threshold.
Moreover, it has AI voice control, e.g., I can ask it to just clean a specific area such as near the cat litter box.
This kind of robot is not available in any European market, due to the fact that tech companies in Europe mostly don't do rat race like some other top countries do. So we usually buy some smart devices when we do family visit and take the smart devices with us back to Sweden.

Here is the **problem**: This brand new shiny robot cleaner is Geo-fenced! And we cannot use it here at our nice home!

# What is the solution?

Since most robots usually does not use real GPS signal for geofencing (the reason could be cost reduction for manufacturer, and GPS works bad inddor), I think it should be IP based geofencing.
Then it is easy to solve the problem with a VPN.

But I cannot install a VPN on a robot, since I don't want to break the official firmware.
So I have to make a VPN router to let the robot connect to, and the robot's network traffic should be automatically directed through the VPN.

For me another **key point** is: I don't want to pay any VPN service provider, since I **DON'T TRUST** any 3rd party VPN service provider from the geofencing country. So I need to choose a good enough one.
Comnon VPN solutions such as [Wireguard](https://www.wireguard.com/) and [OpenVPN](https://openvpn.net/) are already censored and blocked in the geofencing country. So I have to choose alternatives.

[Zerotier](https://www.zerotier.com/) could be the one. And I have been using it a lot for my other devices for many years. It's a perfect private networking overlay. But I found some guys saying that Zerotier is not stable in the geofencing country.
They mention that [Tailscale](https://tailscale.com/) is a good solution. And one can set it up on any device, Linux, Windows, MacOS, iOS, Android, and even an *Apple TV*... Okay fair enough. Let's go ahead with Tailscale then.

To enable Tailscale VPN, one have to setup an [exit node](https://tailscale.com/kb/1103/exit-nodes).
And one need to run Tailscale client on the other device which want to have VPN access.

Normally, people would install a Tailscale exit node (such as [OpenWRT](https://openwrt.org/)) in the geofencing country. But I am not physically feasible to do this. So I decide to choose a VPS cloud provider and use a VPS instance to run Tailscale as a exit node.
Actually it is super cheap to use Alibaba as a VPS cloud provider. They offer **3 month free trial** with an economic VPS instance in the geofencing country, and then charge people only around **$10-20 USD every year** after free trial is over.

I also need a Tailscale VPN router here at my home. Unfortunately I am not the rule maker at home, so I cannot change our nice router into OpenWRT because we want to stick with factory stock firmware.
And I also want to keep everything cheap and tidy. Luckily I found out that I have an old long-forgotten Raspberry Pi 3B slumbering for years in a neglected corner of the study.
That's it! Since I only need to route a robot which is an embedded device, there is no requirement for a high speed network required by streaming or gaming.

So here would be the setup:
1. A modem provided by ISP to give Internet access.
2. A home router which is for normal usage. Several devices connect to the router.
3. A remote Tailscale exit node which runs on a VPS in the geofencing country to provide the VPN access to the Internet in that country.
4. A RPi 3B VPN router which runs Tailscale and uses the Tailscale exit node in the other country. This RPi also runs [hostapd](https://w1.fi/hostapd/) where the robot connects to, and forward all the traffic from host AP to the Tailscale exit node, so that my luxury robot will "think" itself as in the geofencing country.

{% mermaid %}
graph TD
    Internet[🌐 Internet]
    ISP[📡 ISP Modem]
    HomeRouter[🏠 Home Router]
    RPiRouter[🥧 RPi 3B VPN Router<br/>Tailscale + hostapd]
    TailscaleVPS[☁️ Tailscale Exit Node<br/>VPS in Other Country]
    
    %% Regular home devices
    Laptop[💻 Laptop]
    Phone[📱 Phone]
    TV[📺 Smart TV]
    
    %% VPN connected devices
    VPNRobot[📱 \(VPN\) Robot cleaner]
    VPNIoTDevice[📱 \(VPN\) IoT Device]
    
    %% Internet connections
    Internet --- ISP
    ISP --- HomeRouter
    HomeRouter --- RPiRouter
    
    %% Regular device connections
    HomeRouter --- Laptop
    HomeRouter --- Phone
    HomeRouter --- TV
    
    %% VPN device connections to RPi AP
    RPiRouter --- VPNRobot
    RPiRouter --- VPNIoTDevice
    
    %% Tailscale tunnel
    RPiRouter -.->|Tailscale Tunnel| TailscaleVPS
    TailscaleVPS -.->|Exit to Internet<br/>from Other Country| Internet
    
    %% Styling
    classDef vpnDevices fill:#e1f5fe,stroke:#0277bd,stroke-width:2px
    classDef infrastructure fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef tunnel stroke:#ff6f00,stroke-width:3px,stroke-dasharray: 5 5
    
    class VPNRobot,VPNIoTDevice vpnDevices
    class RPiRouter,TailscaleVPS infrastructure
    class RPiRouter tunnel
{% endmermaid %}

# Detailed (or not so?) guide to setup everything

## First, build my own VPS Tailscale exit node

Run a VPS instance and [install Tailscale](https://tailscale.com/kb/1347/installation).
Follow [this guide](https://tailscale.com/kb/1103/exit-nodes) to setup exit node.
One might find out that DNS will be "broken" after enabling Tailscale.
So pay attention to the [common problems](https://tailscale.com/kb/1188/linux-dns#common-problems).

Make sure I don't see any `Tailscale health check` when I run:
```bash
sudo tailscale status
```

I cannot be very detailed here because everyone has their own preference towards a cloud VPS service. And there are tons of guides out there on the Internet, so I decide to skip this chapter.

## Second, build my RPi VPN router

Here I start with a [RaspberryPi OS **Lite**](https://www.raspberrypi.com/software/operating-systems/).
Because My RPi 3B is too old, I want the system to be very small and efficient.

Then remove the NetworkManager... (You can ask AI why?)

```bash
sudo apt remove network-manager
```

![](https://user-images.githubusercontent.com/1140756/124652877-7e713e00-de51-11eb-9973-09be5be1f069.jpg)

NetworkManager is heavy, bloaty, and sometimes has conflict with [Tailscale DNS race condition](https://github.com/tailscale/tailscale/issues/2334).
Again I want the system to be very efficient.

I use `systemd-networkd` and `systemd-resolvd` instead. Install them if not:

```bash
sudo apt install systemd-networkd systemd-resolvd
```

Then install `hostapd`:

```bash
sudo apt install hostapd
```

Now I can specify my `/etc/hostapd/hostapd.conf`:

```
# Interface and driver
interface=wlan0
driver=nl80211

# Network name and mode
ssid=MyVpnAP
hw_mode=g
channel=6 # Less likely to overlap with other WiFi
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0

# Security
wpa=2
wpa_passphrase=MyStrongEnoughPassword
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP

# Country code (important for location spoofing)
country_code=CN # Oops, now everyone know which geofencing country it is...
ieee80211d=1
ieee80211h=1
```

Install the dnsmasq to manage the DHCP and DNS on `wlan0`:

```bash
sudo apt install dnsmasq
```

And I want to avoid `systemd-resolved` to manage `wlan0`, which will cause conflict.
So I specify `/etc/systemd/network/10-wlan0.network`:

```
[Match]
Name=wlan0

[Network]
Address=192.168.4.1/24
IPForward=yes
DHCPServer=no
```

And specify `/etc/dnsmasq.conf`:

```
# Don't let dnsmasq poll /etc/resolv.conf or other resolv
no-poll

# Upstream DNS Servers
server=100.100.100.100 # Tailscale
server=223.5.5.5       # Alibaba
server=1.1.1.1         # Cloudflare

# Only for wlan0 (which used by hostapd)
interface=wlan0
bind-interfaces
listen-address=192.168.4.1

# Override the default route supplied by dnsmasq, which assumes the
# router is the same machine as the one running dnsmasq.
dhcp-option=3,192.168.4.1
# Enable the integrated DHCP server
dhcp-range=192.168.4.2,192.168.4.50,255.255.255.0,24h

# Slightly bigger cache size than default 150
cache-size=1000
```

Now my host AP will have a gateway IP address `192.168.4.1`,
and my clients which connect to the AP will have IP address from `192.168.4.2` to `192.168.4.50`.
I force the DNS to be a Tailscale DNS or fall back to Alibaba's DNS so the host AP can be a really nice target geo.

`dnsmasq` is convenient for me to do very specific setup on host AP.
Thus host AP can be **"geo-fencing"** as the target country.

And I let `systemd-resolved` to manage the rest of the network interfaces.

Remember to add this `/etc/systemd/network/10-eth0.network` for Ethernet cable:

```
[Match]
Name=eth0

[Network]
DHCP=yes
```

It's better to reboot my RPi once to apply those settings before proceed with Tailscale.

After reboot, [install Tailscale](https://tailscale.com/kb/1347/installation).
Bring Tailscale online and solve [common problems](https://tailscale.com/kb/1188/linux-dns#common-problems).
My Tailscale should [use the exit node](https://tailscale.com/kb/1103/exit-nodes?tab=linux#use-the-exit-node) which I have set up in the previous chapter.

Make sure I don't see any `Tailscale health check`:

```bash
$ sudo tailscale status
100.69.181.99   rpi-3b-tailscale     charles.cc.zhang@ linux   -
100.117.23.101  izbp15dsdfqo80wert7lm2z-1 charles.cc.zhang@ linux   active; exit node; direct 114.215.188.166:41641, tx 60592228 rx 322398812
```

Here I have `rpi-3b-tailscale` which stands for my RPi 3B, and `izbp15dsdfqo80wert7lm2z-1` for my VPS in the geofencing country.

Now it is time to setup the traffic forwarding from host AP to Tailscale.

Install iptables:

```bash
sudo apt install iptables iptables-persistent
```

Setup kernel parameters in `/etc/sysctl.conf`:

```
net.ipv4.ip_forward=1
net.ipv4.conf.tailscale0.forwarding=1
net.ipv4.conf.wlan0.forwarding=1
```

And apply:
```bash
sudo sysctl -p
```

Add firewall rules:

```
sudo iptables -A FORWARD -s 192.168.4.0/24 -o tailscale0 -j ACCEPT
sudo iptables -A FORWARD -i tailscale0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT

sudo iptables -t nat -A POSTROUTING -s 192.168.4.0/24 -o tailscale0 -j MASQUERADE

# This is very important to do TCP MSS clamping because Tailscale has a different mtu than wlan0
sudo iptables -t mangle -A FORWARD -o tailscale0 -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1240
```

Now I have finished all the configurations!

Oh well, save my iptables to make it persistent:
```bash
sudo netfilter-persistent save
```

Now, even if there is a power outrage, my RPi VPN tunnel will automatically come back after a reboot.

## Test the outcome

I can use my mobile to connect to my host AP `MyVpnAP`, then just visit `ifconfig.me`.
And I get the correct IP which is in the geofencing target country.

I even opened a specific geofencing live streaming APP from that country and stream a little bit. For the first half minute it was fine, but then it became choppy and the AP WiFi broke. 
Oh my poor RPi 3B...

But the AP immediately came back afterwards once I stop the streaming APP.
This RPi VPN router is not for heavy duty for sure.

Our luxury robot cleaner is now happily working, the only thing we have not let it try is to let it open its camera and automatically chase my nice little cat. I think this functionality is too heavy for my RPi VPN router.

So far I have **not** spent **a single penny** because I am reusing my old devices, and the VPS exit node is still on free trial.
After the trial period over, I am more than willing to pay Alibaba $10-20 a year for keep the VPS up and running as an Tailscale exit node.

# Caveat

For most robot's manufacturer's controlling APP, it is **strongly recommended** to also connect to the RPi VPN router's AP while you do the configuration towards your robot.
This is to make sure your real IP won't leak from your mobile APP to the manufacturer.

# A better approach

RPi router is not optimal for wireless.
[OpenWRT with Tailscale](https://openwrt.org/docs/guide-user/services/vpn/tailscale/start) is much better in every aspect, e.g. coverage and performance.
If OpenWRT is available, don't hesitate to replace the RPi with it!

Hope this blog finds someone who needs it. Cheers!
