# PiBkupDI
Process for backing up a Raspberry Pi as a Disk Image. Commands consolidated in single text file to save manual typing.


Condensed and reformatted from the article written by Avram Piltch - https://www.tomshardware.com/how-to/back-up-raspberry-pi-as-disk-image


This method allows you to clone your OS at a point in time, shrink it to a manageable size, and then restore it quickly if needed.

Dependencies:

* External storage device - Needs to have more storage space then your current drive, unless you first shrink your partition. 

* PiShrink - https://github.com/Drewsif/PiShrink

* Sudo/root access

The commands cheat sheet can be downloaded locally using: 
``` 
sudo wget https://github.com/geoscoutCJ/PiBkupDI/blob/main/commands.txt
```

## Background

There are a few ways to backup a Raspberry Pi. You can use Raspberry Pi OS’s SD Card Copier app, which is under the Accessories section of the Start menu, to clone your microSD card directly to another microSD card. But unless you need a second card right away, it’s a better idea to create a disk image: a file you can store on a PC or in the cloud, distribute to others and write to a new microSD card at any time, using Rufus or Raspberry Pi Disk Imager.

## Creating the Disk Image

1.  Format a USB Flash or hard drive as either NTFS (if you are using Windows on your PC and plan to read this drive on a PC) or EXT4 (for Linux). Make sure to give the drive a volume name that you remember (ex: “pibkup” in our case). You can also format the drive directly on the Raspberry Pi if you like. At this point, we will assume that the flash drive is larger than the existing SD card. If not, be sure to read: [How To Shrink a Partition on Raspberry Pi using the GUI](https://github.com/geoscoutCJ/PiBkupDI/edit/main/README.md#how-to-shrink-a-partition-on-raspberry-pi-using-the-gui)

2.  Connect the external drive to your Raspberry Pi.

3.  Install pishrink.sh on your Raspberry Pi and copy it to the /usr/local/bin folder by typing:
  
```
wget https://raw.githubusercontent.com/Drewsif/PiShrink/master/pishrink.sh
sudo chmod +x pishrink.sh
sudo mv pishrink.sh /usr/local/bin
```

4. Check the mount point path of your USB drive by entering:

```
lsblk
```

You’ll see a list of drives connected to the Raspberry Pi and the mount point name of each. Your USB drive will probably be mounted at /media/pi/[VOLUME NAME]. For example: /media/pi/pibkup. If your drive isn’t mounted, try rebooting with the USB drive connected or you can mount it manually by creating a directory and then mounting it. Example:
```
sudo mkdir /media/pi/pibkup
sudo mount /dev/sda1 /media/pi/pibkup
```

However, you can’t and shouldn’t do that if it’s already mounted.

5. Copy all your data to an img file. I prefer to use ```pv``` paired with ```dd``` so I get a progress bar but you can use either.

    - NOTE: If you are using a shrunk partition, this command will need to include the _count_ attribute. See [How To Shrink ...](https://github.com/geoscoutCJ/PiBkupDI/edit/main/README.md#how-to-shrink-a-partition-on-raspberry-pi-using-the-gui) for more information.

```sudo pv -tpreb /dev/mmcblk0 | dd of=[mount point]/myimg.img bs=1M```

or

```sudo dd if=/dev/mmcblk0 of=[mount point]/myimg.img bs=1M status=progress```

This is by far the longest portion in my experience. Taking quite while, depending on how much data you have, your Pi's processing power, and the read/write speeds of both set of storage media. More then an hour, sometimes quite a bit more, in my experience.

6. Navigate to the USB drive's root directory (the mount point).

```cd /media/pi/pibkup ```

7.  Use pishrink with the -z parameter, which zips your image up with gzip.

```sudo pishrink.sh -z myimg.img```

This is much faster than the pv/dd commands, generally taking between 10 minutes to an hour. When it is done, you will end up with a reasonably sized image file called myimg.img.gz. You can copy this file to your PC and then save it however you like. (Upload it to the cloud, send it to a friend, etc) 

## Writing Your Raspberry Pi Disk Image to a Card

Once you’re done, you’ll have a file with the extension .img.gz and you can write or “burn” it to a microSD card the same way you would any.img file you download from the web. 

The easiest way to burn a custom image is to:

1.  Launch Raspberry Pi Imager on your PC. Can be found here, if you don’t have it already: https://www.raspberrypi.com/software/

2.  Select Use custom from the Choose OS menu.

3.  Select your .img.gz file.

4.  Select the microSD card you wish to burn it to.

5.  Click Write.

## How to Shrink a Partition on Raspberry Pi using the GUI

By default, the dd file copy process makes an image out of ALL the space on your microSD card, even the unused space. For example, say you have a 64GB microSD card, but are only actually using 6GB of space. If you don’t shrink the rootfs partition, you will end up copying all 64GB over to your external drive, which will take a lot more time to complete and will require that you have at least 65GB of free space.

The solution is to shrink the rootfs partition of your microSD card down to a size that’s just a little bit bigger than the amount of used space. Then you can copy just your partitions over to the USB drive.

To do the shrinking, you’ll need a USB microSD card reader and a second microSD card with Raspberry Pi OS and GUI on it.

1. Put your source microSD card (the one you want to copy) in a reader and connect to your Raspberry Pi.

2. Boot your Raspberry Pi off a different microSD card.

3. Install gparted on your Raspberry Pi.

``` sudo apt-get install gparted -y ```

4.  Launch gparted from within the Raspberry Pi OS GUI. It’s in the System Tools section of the start menu.

5.  Select your external microSD card from the pull down menu in the upper right corner of the gparted window.

6. Unmount the rootfs partition if it is mounted (a key icon is next to it) by right clicking it and selecting Unmount from the menu. If the option is grayed out, it’s not mounted.

7.  Right click rootfs and select Resize / Move.

8.  Set the new size for the partition as the minimum size or slightly larger and click Resize.. Note that gparted may overreport the amount of used space (when we unmounted a partition with 4.3GB used, it changed to say 6GB were in use), but you have to go with at least its minimum.

9.  Click the green check mark in the gparted window and click Apply (when warned) to proceed.

10. Shutdown the Raspberry Pi.

11. Remove the source microSD card from the USB card reader and insert it into the Raspberry Pi to boot from.

12. Follow the instructions in the section above on creating a disk image. In step 5, you will need to use the ``` count ``` attribute. This will tell ``` DD ``` to copy only as many MBs as are in use. In our example, we had had a 16GB card, but after shrinking the rootfs down to 6.5GB, the card only had about 6.8GB in use (when you count the /boot partition). So, to be on the safe side (better to copy too much data than too little), we rounded up and set dd to copy 7GB of data by using count=7000. The amount of data is equal to count * block size (bs) so 7000 * 1M means 7GB. The command we would use looks like this:

```
sudo dd if=/dev/mmcblk0 of=[mount point]/myimg.img bs=1M count=7000
```
