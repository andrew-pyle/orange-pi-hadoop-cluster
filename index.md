# Single-Board Computer Hadoop Cluster
##### _Andrew Pyle | IFSC 7370 | April 2017_

![Orange Pi One](./images/orange-cover.JPG)

Let's set up a physical Hadoop 2 cluster with single-board Linux computers! I will be using 1 [Raspberry Pi 1 (Model B)](https://www.raspberrypi.org/products/model-b/) and two [Orange Pi One](http://linux-sunxi.org/Orange_Pi_One) boards. By the end of this journal, we will have a ethernet connected Hadoop 2 cluster of three single-board computers. This setup will not be powerful enough for production use, but it will demonstrate a simple implementation of the Hadoop 2 ecosystem for learning purposes.

##### Materials

I am using the following items to create the cluster:

* 1 [Raspberry Pi Model B](https://www.raspberrypi.org/products/model-b/)
* 2 [Orange Pi Ones](http://linux-sunxi.org/Orange_Pi_One)
* For each single-board computer
  * Sandisk Ultra 16 GB Class 10 micro SD card ([Amazon](https://www.amazon.com/gp/product/B010Q57SEE/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1))
  ([Speed Ratings Info](https://kb.sandisk.com/app/answers/detail/a_id/1996/~/difference-between-speed-class%2C-uhs-speed-class%2C-and-speed-ratings))
  * 5V 2A USB Power supply
  * USB-DC barrel plug cable ([Amazon](http://a.co/dzPdzaR))
* Network Interface
  * Ethernet switch and CAT5 cable
  _Note: USB WiFi Dongles could work, but are less reliable_
* Display and HDMI-DVI cable

##### Boot up the Orange Pis
###### 24 March 2017

My Raspberry Pi is currently in use for another project, so I will start this project setup with the Orange Pi One boards.

I purchased 2 boards from [AliExpress.com](https://www.aliexpress.com/item/Orange-Pi-One-H3-Quad-core-Support-ubuntu-linux-and-android-mini-PC-Beyond-Raspberry-Pi/32603308880.html), along with a Sandisk Ultra 16 GB Class 10 micro SD card from [Amazon](https://www.amazon.com/gp/product/B010Q57SEE/ref=oh_aui_detailpage_o00_s00?ie=UTF8&psc=1) for each.

![Orange Pi One](./images/orange-top.JPG)

Each Orange Pi One requires a 5V 2A power supply with a DC barrel plug ([4.0mm/1.7mm - center positive](http://linux-sunxi.org/Orange_Pi_One)). I used a [USB-DC barrel plug cable from Amazon](http://a.co/dzPdzaR) so I could use a USB power supply that I had lying around the house.

![USB-DC Barrel Plug Cable](./images/orange-cable.JPG)

###### SD Card Integrity

First, we need to ensure the integrity of the SD cards.
> Many scam artists try to market small flash disks as large ones by changing the appearance of the disk. We will use a software test to verify the size of the disk we have. I am using [F3 by Digirati](http://oss.digirati.com.br/f3/) (GPL v3). It runs on Windows, OS X, Linux, and other operating systems. The website includes a very straightforward instructions regarding it's use.

_(Note: I am using MacOS Sierra for the following commands, so if you are using a non-Unix-based OS, note that your commands will be different.)_

Download and unzip the program, then compile it for your system.

```bash
$ cd f3-6.0

$ ls
LICENSE		f3fix.c		f3write.h2w	libutils.c	version.h
Makefile	f3probe.c	libdevs.c	libutils.h
README.md	f3read.1	libdevs.h	log-f3wr
changelog	f3read.c	libprobe.c	utils.c
f3brew.c	f3write.c	libprobe.h	utils.h

$ make
cc -std=c99 -Wall -Wextra -pedantic -MMD -ggdb   -c -o utils.o utils.c
cc -std=c99 -Wall -Wextra -pedantic -MMD -ggdb   -c -o f3write.o f3write.c
cc -o f3write utils.o f3write.o -lm
cc -std=c99 -Wall -Wextra -pedantic -MMD -ggdb   -c -o f3read.o f3read.c
cc -o f3read utils.o f3read.o
```

F3 uses two commands to verify the flash memory: f3write and f3read. The f3write command writes blocks of data to the disk, and f3read reads the data written. In our case, the output will show if the card's claimed capacity matches its usable capacity.

Run f3write on the disk's location in the file system. This can take a while. (Note: if you are using macOS as I am, the location is at /Volumes/your_disk by default.)
```bash
$ ./f3write /Volumes/"NO NAME"
Free space: 14.83 GB
Creating file 1.h2w ... OK!                         
Creating file 2.h2w ... OK!                         
Creating file 3.h2w ... OK!                         
Creating file 4.h2w ... OK!                         
Creating file 5.h2w ... OK!                         
Creating file 6.h2w ... OK!                         
Creating file 7.h2w ... OK!                         
Creating file 8.h2w ... OK!                         
Creating file 9.h2w ... OK!                         
Creating file 10.h2w ... OK!                         
Creating file 11.h2w ... OK!                        
Creating file 12.h2w ... OK!                        
Creating file 13.h2w ... OK!                        
Creating file 14.h2w ... OK!                        
Creating file 15.h2w ... OK!                        
Free space: 0.00 Byte
Average writing speed: 9.79 MB/s
```

>_Note: I received the following error when executing f3write the first time. I just retried the f3write command, and execution was successful. I am not sure what caused the error, and a quick Google search did not return any help. I will just keep the info in mind._
>```bash
>$ ./f3write /Volumes/"NO NAME"
>Free space: 14.83 GB
>Creating file 1.h2w ... 0.29% -- 10.00 MB/s
>
>f3write: Write to file /Volumes/NO NAME/1.h2w failed: Input/output error
>```

And run f3read to verify the data written.
```bash
$ ./f3read /Volumes/"NO NAME"
                  SECTORS      ok/corrupted/changed/overwritten
Validating file 1.h2w ... 2097152/        0/      0/      0
Validating file 2.h2w ... 2097152/        0/      0/      0
Validating file 3.h2w ... 2097152/        0/      0/      0
Validating file 4.h2w ... 2097152/        0/      0/      0
Validating file 5.h2w ... 2097152/        0/      0/      0
Validating file 6.h2w ... 2097152/        0/      0/      0
Validating file 7.h2w ... 2097152/        0/      0/      0
Validating file 8.h2w ... 2097152/        0/      0/      0
Validating file 9.h2w ... 2097152/        0/      0/      0
Validating file 10.h2w ... 2097152/        0/      0/      0
Validating file 11.h2w ... 2097152/        0/      0/      0
Validating file 12.h2w ... 2097152/        0/      0/      0
Validating file 13.h2w ... 2097152/        0/      0/      0
Validating file 14.h2w ... 2097152/        0/      0/      0
Validating file 15.h2w ... 1733888/        0/      0/      0

  Data OK: 14.83 GB (31094016 sectors)
Data LOST: 0.00 Byte (0 sectors)
	       Corrupted: 0.00 Byte (0 sectors)
	Slightly changed: 0.00 Byte (0 sectors)
	     Overwritten: 0.00 Byte (0 sectors)
Average reading speed: 22.27 MB/s
```
Great! no files are listed in the 'corrupted' column above, so the microSD cards should be good.

If your disk shows files in the 'corrupted' category or a much smaller capacity than advertised, ask for your money back. See F3's website for more specifics on the program's use and functionality.

###### Flash the OS
The next step is to write the operating system for the orange pi to a microSD card, called "flashing" the card.

We will be installing the [Armbian distribution](https://www.armbian.com/orange-pi-one/) of Debian Jessie. It is a solely CLI image, which means that it uses a text-based command-line interface. They host a Ubuntu Xenial image with CLI only and GUI desktop interface versions as well.

>Shenzhen Xunlong CO., Limited, the makers of the Orange Pi, host their own [OS images](http://www.orangepi.org/downloadresources/) for the Orange Pi One, but they are the same images as for another model, the Orange Pi PC. I have read that the image is not fully supported by the hardware of the One, while Armbian's image is customized for the One. [See linux-sunxi](http://linux-sunxi.org/Xunlong_Orange_Pi_One_%26_Lite) for details.

Download and unzip the OS image. I used [The Unarchiver](http://unarchiver.c3.cx/unarchiver) for mac to unzip the .7z file. The OS image file is highlighted.
![alt text](./images/armbian-unzip.png "Unzipped Armbian Orange Pi One OS image")
Armbian offers a [GPG signature](https://docs.armbian.com/User-Guide_Getting-Started/) for authentication of the OS image download, if you want to check that out. The [process](http://notes.jerzygangi.com/the-best-pgp-tutorial-for-mac-os-x-ever/) is quite involved however.

##### 26 March 2017
We are will be using [Etcher](https://etcher.io) to flash the microSD card. It is a simple interface available for many OSes.

![Etcher Screenshot](./images/etcher.png)

Just choose the Armbian image and the microSD card. The card was auto selected for me. My Armbian image was named as follows. Ensure that you have the correct filename.
```
Armbian_5.25_Orangepione_Debian_jessie_default_3.4.113
```

Etcher gave a success message, so we can now insert the microSD card into the Orange Pi and begin the first boot.

![Etcher Success Screenshot](./images/etcher-success.png)

Now, just load the microSD into the Orange Pi board, connect a USB keyboard and display, and connect power. I'm using a HDMI-DVI connection.

###### Error!
At this point, we should see a red power light and the Pi should boot up. However, I see no lights at all. This is a problem!

After measuring some voltages with a multimeter, it seems that the issue is the USB-DC power cable. I can measure 5V from the power supply, but no voltage from the cable connector. I will get another cable and try to boot from the Armbian image again.

##### 29 March 2017
Two new cables from Amazon ([this one](http://a.co/gDyAZoq) and [this one](http://a.co/ftKETZe)) arrived today. I will try to boot the Armbian image again. I am skeptical that this will work, since it seems like a cable is the least likely piece of the computer system to fail. 

Let's connect the Orange Pi as above, and try to boot!

![Orange Pi Boot trial with new USB-DC power cable](../images/new-cable.JPG)

LIGHTS! It does seem as if the cable was bad. (I'll try to get my money back.) I am getting a green light and blinking red light. The DVI monitor showed an error message and then a blank screen, so that means that the resolution coming from the Orange Pi was unusable by the monitor. That also means that I have no idea what is going on with the Orange Pi Boot process. A quick Google search didn't turn up anything specific, so I will try to boot with another power supply.

>The Orange Pi is extremely picky about the power supply. Even a supply marked 5V 2A will fail frequently. It seems that the voltage must be extremely stable over all amperages, and the amperage must not fall far below 2A at any time during operation. It seems that the best solution for the Orange Pi power supply is to buy the  5V 3A power supply from [AliExpress](https://www.aliexpress.com/store/product/orange-pi-orange-pi-plus-Power-Adapter-5V-2A-Power-Supply-Micro-USB-Charger/1553371_32248555041.html). They have both US and European versions.

I use the [Plugable 4-port USB Hub](http://plugable.com/products/USB2-HUB4BC) to power my Raspberry Pi with no issues, so I will give that a try as the power supply.

![Orange Pi boot trial with Plugable USB Hub](../images/plugable-boot.JPG)

It works! I got a brief look at the boot process before the monitor went blank due to resolution issues (probably), and the green light went solid with occasional flashes. The red light started solid and then went out. Thankfully SSH is enabled by default on the Orange Pi, or else I'd be unable to change the resolution to match the monitor therefore unable to enable SSH.

Since the Orange Pi booted up with network connected, it received an IP Address by DHCP, and I can check that at my router's homepage. There are other options to ascertain the IP address of the devices on your home network, like [Fing](https://www.fing.io/) for iOS and Android). 

SSH into the Orange Pi One. The default password for root on Armbian is '1234'.

```bash
$ ssh root@[ip_address]    # Replace [ip_address] with the ip address of your Orange Pi
```
A nice login screen is displayed:

```bash
$ **********************login screen
```

Follow the initial boot prompts from Armbian, like changing the root password, making a new user account, and setting the display settings. The h3disp program is provided to set the display settings. 

```bash
$ **********************h3disp
```

My monitor is 1280x1024 @ 60 hz with a DVI adapter. I am not sure why the DVI adapter makes a difference, but it has its own '-d' flag. I will use the following command:

```bash
$ h3disp -m 1280x1024 -d ***********Context
```

After rebooting, the monitor shows the correct image, but it still shows an error message. After trying several other formats (all worse outcome than the above setting), I found that if I push the menu button on the monitor before it goes blank, it will interrupt the sleep timeout process, and after closing the menu, the error is gone and the monitor works properly. I can't explain this one, but it seems to work!

![Monitor menu trick](./images/monitor-trick.JPG)

##### 30 March 2017

# TODO:
* Configure GitHub Pages: http://kbroman.org/simple_site/pages/overview.html
* Pics:
  * Brighten Pics 
  * Raspbery Pi
  * Monitor error & trick
  * see above blanks
