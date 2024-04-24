# Automatically Mounting USB Drives on Debian 11 (Buster) (Linux, Raspberry Pi)

## Introduction
First, the problem.

I have an old i86 notepad that I've installed [Raspberry Desktop PI](https://www.raspberrypi.com/software/raspberry-pi-desktop/) on. This is a central library server for my projects plus has an external USB drive that holds backups. The library and backup are shared through NFS to various Raspberry PI Clients. All has been working fine for several months.

I had to perform a reboot and Debian 10 (Buster) came up in recovery mode. I finally got access and noted that the backup was not mounted. This seems to have been the problem with the boot to recovery mode.

I was using FSTAB to mount the backup. Initially when I plugged in the drive, it was assigned it to /deb/sdb1 which is what I was using in the mount. However, now it was listed as /dev/sdd1 (subsequently it showed also as /dev/sdc1).

Looking for answers, I found that the device for USB drives may change for several reasons including cable issues. The recommendation is to:

* Mount USB drives using their UUID's
* Do not use FSTAB to mount USB drives where to could interfere with the boot process (which is what I experienced). One thing I didn't know about was the NOFAIL option which does ["Do not report errors for this device if it does not exist."](https://superuser.com/questions/449922/fstab-on-boot-mount-when-device-is-plugged-in)

## Solution Research

Researched showed there were a number of solutions.

* Use SYSTEMD's [systemd.mount](https://manpages.debian.org/testing/systemd/systemd.mount.5.en.html). There is also an AUTOMOUNT variant which I discounted after reading [this post](https://unix.stackexchange.com/questions/570958/mount-vs-automount-systemd-units-which-one-to-use-for-what)
* Use SYSTEMD to run a shell script to mount the USB.
* Use [udev](https://wiki.debian.org/udev). I glanced at this but it was different paradym and I was familiar with SYSTEMD

There are others but I've lost the links for them.

For the server I decided to utilize systemd.mount for the server. I was already using systemd with shell  scripts for the clients. These have been included as an appendix

## Getting Started with systemd.mount

The Desktop PI runs Debian 10 (Buster) so I don't know if there will be changes in newer releases.

As noted in other posts, systemd.mount is not as well documented as other services. Some of what I learned was from examples I found online then trial and error. 

As a warning, at one point, I had a boot problem with systemd.mount configured. Not really sure what.

You need to plug in your USB then run `blkid` to get the UUID

```
$ blkid 
/dev/sda1: UUID="90f562ca-1ef2-4bfe-bbb4-99cd34a1e1ec" TYPE="ext4" PARTUUID="a58b733e-01"
/dev/sda5: UUID="f8afd7bb-d934-4e7b-9361-f659677104cd" TYPE="swap" PARTUUID="a58b733e-05"
/dev/sdc1: LABEL="pi_backups" UUID="f3ca97e2-f777-4d50-9a22-a267f7a299ea" TYPE="ext4" PARTUUID="9f00e08a-373e-4f4d-a349-e4372d8e5924"
$  
```
The disk I am mounting is 'pi_backups' with the UUID "f3ca97e2-f777-4d50-9a22-a267f7a299ea". The mount point is /media/pi_backup. This name is significant. 

## systemd.mount Definition and Service Creation

First, create the systemd.mount definition. You need to use the name of the mount mount and substitute '-' for '/'. If you don't, you get an error that the `Where=` does not match the file name.

```
$ more media-pi_backup.mount

[Unit]
Description=Mount pi_backup on USB

[Mount]
# Best pratice mount my UUID rather than device. USB device assignment can change
What=/dev/disk/by-uuid/f3ca97e2-f777-4d50-9a22-a267f7a299ea
Where=/media/pi_backup
Type=ext4
Options=rw,noauto
$
```
I initially had `After=`, `Wants=` and `[Install]` options but I was having problems stopping the service. I don't have this experience with more common `.service` definitions. I dropped them and everything seems to work. The key is that NFS shares /media/pi_backup properly. There are examples on line where some of these options are used. I didn't dig into where my problem was since it was working.

For systemd, I create shell scripts to reload definitions when I've changed them. Here's the one for this mount service

```
$ more reload.sh

#!/bin/bash
# Manage the SYSTEMD Services media-pi_backup.mount
#set -x
ctl="sudo systemctl"
nm=media-pi_backup.mount
echo Stopping Service $nm
$ctl stop $nm
echo Copy service definitions from  .
sudo cp -v ./$nm /etc/systemd/system
sudo chmod -v +644 /etc/systemd/system/$nm
echo Reload DAEMON
$ctl daemon-reload
echo Start/Enable  $nm
echo An error will occur if $nm  is busy
$ctl start $nm
$ctl enable $nm
$ctl status $nm

$
```
# APPENDIX

## Mount using SYSTEMD Service (not .mount)

When some of my projects are deployed, the server is deployed with them or on-line. Before I found systemd.mount, I'd developed the following script for the Raspberry PI Clients.

```
$ more mount_libraries.sh

#!/bin/bash
# Do not place librares mounts in FSTAB since desktopPI may be off-line.
# Mount if desktopPI.local found

echo $(date) mount_libraries.sh - Mount desktopPI.local NFS Shares

NFS=desktopPI.local
ping -c 3 $NFS > /dev/null 2>&1
        if [ $? -eq 0 ]; then
                echo "Mounting NFS from " $NFS
                sudo mount -t nfs --options proto=tcp,port=2049,lookupcache=none,rw $NFS:/mnt/pi_lib /mnt/pi_lib
                sudo mount -t nfs --options proto=tcp,port=2049,lookupcache=none,rw $NFS:/media/pi_backup /mnt/pi_backup
                showmount -e desktoppi.local
        else
                echo $NFS " Offline"
        fi

$
```
This script pings for the server; if it is on-line it mounts the NFS shares then show the mount status.

The systemd service definition is

```
$ more mount_libraries.service

[Unit]
# Need at least network services but waiting for time ensures this
After=systemd-time-wait-sync.service
Wants=systemd-time-wait-sync.service

[Service]
Type=idle
User=pi
ExecStart=/media/work/bin/mount_libraries.sh
WorkingDirectory=/media/work/bin
StandardOutput=append:/media/work/log/mount_libraries.log
StandardError=append:/media/work/log/mount_libraries.log
Restart=no

[Install]
WantedBy=multi-user.target

$
```

I find using the log files useful in debugging problems.

Here is a script which loads the systemd service

```
$ more mount_libraries_load.sh

#!/bin/bash

ctl="sudo systemctl "
fname=mount_libraries

echo Copy service $fname definitions
sudo cp -v /media/work/bin/$fname.service /etc/systemd/system
sudo chmod -v +644 /etc/systemd/system/$fname*

echo Reload DAEMON
$ctl daemon-reload

echo Start and Enable $fname
$ctl start $fname
$ctl enable $fname

$ctl status $fname

$
```
