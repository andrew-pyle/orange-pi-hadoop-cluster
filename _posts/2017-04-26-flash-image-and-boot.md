---
layout: post
title:  "Flash Image and Boot"
date:   2017-04-26
---

#### 26 March 2017
We will be using [Etcher](https://etcher.io) to flash the microSD card. It is a simple interface available for many OSes.

![Etcher Screenshot](./images/etcher.png)

Just choose the Armbian image and the microSD card. The card was auto selected for me. My Armbian image was named as follows. Ensure that you have the correct filename.
```
Armbian_5.25_Orangepione_Debian_jessie_default_3.4.113
```

Etcher gave a success message, so we can now insert the microSD card into the Orange Pi and begin the first boot.

![Etcher Success Screenshot](./images/etcher-success.png)

Now, just load the microSD into the Orange Pi board, connect a USB keyboard and display, and connect power. I'm using a HDMI-DVI connection.

##### Error!
At this point, we should see a red power light and the Pi should boot up. However, I see no lights at all. This is a problem!

After measuring some voltages with a multimeter, it seems that the issue is the USB-DC power cable. I can measure 5V from the power supply, but no voltage from the cable connector. I will get another cable and try to boot from the Armbian image again.

#### 29 March 2017
##### New Power Cables
Two new cables from Amazon ([this one](http://a.co/gDyAZoq) and [this one](http://a.co/ftKETZe)) arrived today. I will try to boot the Armbian image again. I am skeptical that this will work, since it seems like a cable is the least likely piece of the computer system to fail.

Let's connect the Orange Pi as above, and try to boot!

##### Boot Up

LIGHTS! It does seem as if the cable was bad. (I'll try to get my money back.) I am getting a green light and blinking red light. The DVI monitor showed an error message and then a blank screen.
> I wish I'd gotten a video here of the flashing red light, but I didn't have the camera ready. I don't know how to reproduce the error, except maybe to erase the microSD card and try an initial boot again, which would be wasting the work already done at this point.

This all means the resolution coming from the Orange Pi is unusable by the monitor. This also means that I currently have no idea what is going on with the Orange Pi Boot process. A quick Google search didn't turn up anything specific, so I will try to boot with another power supply.

>The Orange Pi is extremely picky about the power supply. Even a supply marked 5V 2A will fail frequently. It seems that the voltage must be extremely stable over all amperages, and the amperage must not fall far below 2A at any time during operation. It seems that the best solution for the Orange Pi power supply is to buy the  5V 3A power supply from [AliExpress](https://www.aliexpress.com/store/product/orange-pi-orange-pi-plus-Power-Adapter-5V-2A-Power-Supply-Micro-USB-Charger/1553371_32248555041.html). They have both US and European versions.

I use the [Plugable 4-port USB Hub](http://plugable.com/products/USB2-HUB4BC) to power my Raspberry Pi with no issues, so I will give that a try as the power supply.

![Orange Pi boot trial with Plugable USB Hub](images/orange-power.JPG)

It works! I got a brief look at the boot process before the monitor went blank due to resolution issues (probably), and the green light went solid with occasional flashes (the light is on the lower right corner of the board, by the GPIO pins). The red light started solid and then went out. Thankfully SSH is enabled by default on the Orange Pi, or else I'd be unable to change the resolution to match the monitor therefore unable to enable SSH.

Since the Orange Pi booted up while connected to the network, it received an IP Address by DHCP. I can check that at my router's homepage.
>(There are other methods to ascertain the IP address of the devices on your home network, like [Fing](https://www.fing.io/) for iOS and Android).

SSH into the Orange Pi One. The default password for root on Armbian is '1234'. Add the RSA fingerprint to the list of known hosts if asked. This registers the Orange Pi hardware with your computer.

```bash
$ ssh root@[ip_address]   # Replace [ip_address] with the IP
                          # address of your Orange Pi
```
```bash
root@192.168.0.108 password:    #1234
```
A nice login screen is displayed upon SSH connection.

```bash
  ___                               ____  _    ___             
 / _ \ _ __ __ _ _ __   __ _  ___  |  _ \(_)  / _ \ _ __   ___
| | | |  __/ _  |  _ \ / _  |/ _ \ | |_) | | | | | |  _ \ / _ \
| |_| | | | (_| | | | | (_| |  __/ |  __/| | | |_| | | | |  __/
 \___/|_|  \__,_|_| |_|\__, |\___| |_|   |_|  \___/|_| |_|\___|
                       |___/                                   

Welcome to ARMBIAN 5.25 stable Debian GNU/Linux 8 (jessie) 3.4.113-sun8i   
System load:   0.03            	Up time:       3 min		
Memory usage:  10 % of 494Mb  	IP:            192.168.0.108
CPU temp:      35°C           	
Usage of /:    7% of 15G    	
```

##### Configure System

Now that we can boot the Orange Pi up, lets get the system configured for use.

Follow the initial boot prompts from Armbian, like changing the root password, making a new user account, and setting the display settings.

The h3disp program is provided to set the display settings.

```bash
root@orangepione:$ sudo h3disp
Usage: h3disp [-h/-H] -m [video mode] [-d] [-c [0-2]]

############################################################################

 This is a tool to set the display resolution of your Orange
 Pi by patching script.bin.

 In case you use an HDMI-to-DVI converter please use the -d switch.

 The resolution can be set using the -m switch. The following resolutions
 are currently supported:

    480i	use "-m 480i" or "-m 0"
    576i	use "-m 576i" or "-m 1"
    480p	use "-m 480p" or "-m 2"
    576p	use "-m 576p" or "-m 3"
    720p50	use "-m 720p50" or "-m 4"
    720p60	use "-m 720p60" or "-m 5"
    1080i50	use "-m 1080i50" or "-m 6"
    1080i60	use "-m 1080i60" or "-m 7"
    1080p24	use "-m 1080p24" or "-m 8"
    1080p50	use "-m 1080p50" or "-m 9"
    1080p60	use "-m 1080p60" or "-m 10"
    1080p25	use "-m 1080p25" or "-m 11"
    1080p30	use "-m 1080p30" or "-m 12"
    1080p24_3d	use "-m 1080p24_3d" or "-m 13"
    720p50_3d	use "-m 720p50_3d" or "-m 14"
    720p60_3d	use "-m 720p60_3d" or "-m 15"
    1080p24_3d	use "-m 1080p24_3d" or "-m 23"
    720p50_3d	use "-m 720p50_3d" or "-m 24"
    720p60_3d	use "-m 720p60_3d" or "-m 25"
    1080p25	use "-m 1080p25" or "-m 26"
    1080p30	use "-m 1080p30" or "-m 27"
    4kp30	use "-m 4kp30" or "-m 28"
    4kp25	use "-m 4kp25" or "-m 29"
    800x480	use "-m 800x480" or "-m 31"
    1024x768	use "-m 1024x768" or "-m 32"
    1280x1024	use "-m 1280x1024" or "-m 33"
    1360x768	use "-m 1360x768" or "-m 34"
    1440x900	use "-m 1440x900" or "-m 35"
    1680x1050	use "-m 1680x1050" or "-m 36"

 Two examples:

    'h3disp -m 1080p60 -d' (1920x1080@60Hz DVI)
    'h3disp -m 720i' (1280x720@30Hz HDMI)

 You can also specify the colour-range for your HDMI-display with the -c switch.

 The following values for -c are currently supported:

    0 -- RGB range 16-255 (Default, use "-c 0")
    1 -- RGB range 0-255 (Full range, use "-c 1")
    2 -- RGB range 16-235 (Limited video, "-c 2")

############################################################################
```

My monitor is 1280x1024 @ 60 hz with a DVI adapter. I am not sure why the DVI adapter makes a difference, but it has its own '-d' flag. I will use the following command:

```bash
root@orangepione:$ h3disp -m 1280x1024 -d
```

After rebooting, the monitor shows the correct image, but it still shows a resoloution error message. After trying several other formats (all worse outcome than the above setting), I found that if I push the menu button on the monitor before it goes blank, it will interrupt the sleep timeout process, and after closing the menu, the error is gone and the monitor works properly. I can't explain this one, but it seems to work!
