---
title: "Remote control a power supply"
date: 2025-03-02T18:26:46+01:00
categories:
  - blog
tags:
  - hardware
  - power supply
  - serial port
  - Linux
  - Python
---

When working with embedded devices, I need to control, e.g. voltage and current on some test point. I got a good power supply Aim-TTi MX100TP, which I can have very accurate control over three outputs.
Here is the [manual for MX100TP].

Moreover, I found out that MX100TP has a remote control interface. This is great news to me since I am used to setting up everything in the lab to which I can have remote access.
I started with LAN interface. However, after some usage, I found out that the LAN interface is not stable depending on LAN/subnet/firewall situation. It gets impossible to get back to manual control of MX100TP when it goes into remote mode since it has "interface locking". This sucks once it gets stuck in LAN mode and not reachable. And I have to manually power-toggling it to get back the control.

Since I don't have full control of the firmware of MX100TP, I decided to switch to a USB/serial interface. Because I usually have a RaspberryPi gateway, which I can hook up with various hardware if interfaces allow. I have full control over the RaspberryPi, and I can completely configure the RaspberryPi OS however I need.

I started searching around to see if there exists any tool/script that can talk to the power supply over a serial line to MX100TP. Surprisingly I found only one [C# implementation].

I am not so familiar with building and running C# on Linux since it requires Mono framework, and it feels a bit overhead for me.
Thus, I created this little Python script:

<cite><a href="https://github.com/swechencheng/power-supply-serial">power-supply-serial</a></cite>

The usage is simple:
```bash
python3 ps_serial.py /dev/<serial>
```

To make life a bit easier, I also created a udev rule in my `/etc/udev/rules.d/95-usb-serial.rules` for the MX100TP:
```
SUBSYSTEM=="tty", ATTRS{idVendor}=="103e", ATTRS{idProduct}=="04ba", ATTRS{serial}=="425410", SYMLINK+="usbserial.powersupply"
```

This creates /dev/usbserial.powersupply so that I don't have to remember the serial port number. And the command becomes:
```bash
python3 ps_serial.py /dev/usbserial.powersupply
```

Now you can send the commands to MX100TP according to the command list in the manual, e.g.:
```
Enter a command (or 'quit' to exit): OP1?
Response: 0
Enter a command (or 'quit' to exit): OP1 1
Response:
Enter a command (or 'quit' to exit): OP1?
Response: 1
```
It also remembers command history which you can use up down arrow to resume a previous command.

Hope this little script can help to make your life a little bit easier!

[manual for MX100TP]: https://resources.aimtti.com/manuals/MX100T+MX100TP_Instruction_Manual_EN_48511-1610_8.pdf
[C# implementation]: https://stackoverflow.com/a/75055922/23622431
