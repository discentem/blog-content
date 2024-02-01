---
title: "DFU restore Apple Silicon Macs with macvdmtool"
date: 2021-11-04
draft: false
---

# What is [macvdmtool](https://github.com/AsahiLinux/macvdmtool)?

It is a tool written by [marcan](https://github.com/marcan) under the [AsahiLinux project](https://asahilinux.org/). macvdmtool lets you put any Apple Silicon or Intel Macs with the T2 chip (2018 and forward) in DFU mode with terminal commands instead of requiring awkward keyboard gymnastics.

> **Note**: macvdmtool requires that the **host machine** is **Apple Silicon** based, Intel macs do not seem to support this tool as a host.

# Setting up and using macvdmtool

1. Install [Xcode](https://developer.apple.com/xcode/) and then run this in terminal to install Xcode command-line tools:
   ```shell
   xcode-select –-install
   ```
   Check that it installed properly:
   ```shell
   xcode-select -p
   ```
1. Install [Apple Configurator 2](https://apps.apple.com/us/app/apple-configurator-2/id1037126344?mt=12) from the Mac App Store.
1. Clone [macvdmtool](https://github.com/AsahiLinux/macvdmtool.git)

    ```shell
    git clone https://github.com/AsahiLinux/macvdmtool.git
    ```
1. Change into the macvdmtool directory and compile the tool
    ```shell
    macvdmtool && make
    cc -o macvdmtool main.o -framework CoreFoundation -framework IOKit -lc++
    ```
1. Copy macvdmtool to path.
    ```shell
    sudo cp macvdmtool /usr/local/bin
    ```
    Run below command to see if its installed and added to path
    ```shell
    macvdmtool
    ```
1. Launch Apple Configurator 2 and install the "Automation Tools".

    ![Install Automation Tools](/images/macvdmtool/cfgutil/install_automation.png)

    ![Install Automation Tools dialogue](/images/macvdmtool/cfgutil/install_popup.png)

    ![Install Automation Tools password prompt](/images/macvdmtool/cfgutil/install_password.png)

1. Connect the host and target Macs together via an official Apple USB-C charging cable or a TB3/TB4 cable. The cable must be plugged into the DFU port on both ends and support data transfer.

    - On Apple Silicon machines, the DFU port is closest to the screen on the left side
    - On Intel machines, the DFU port is closest to the trackpad on the left side

1. Put the target Mac in DFU mode.

    ```shell
   sudo macvdmtool dfu
    ```
    Above command should return below:
   ```text
    Mac type: J313AP
    Looking for HPM devices...
    Found: IOService:/AppleARMPE/arm-io@10F00000/AppleT810xIO/i2c0@35010000/AppleS5L8940XI2CController/hpmBusManager@6B/AppleHPMBusController/hpm0/AppleHPMARM
    Connection: Source
    Status: APP
    Unlocking... OK
    Entering DBMa mode... Status: DBMa
    Rebooting target into DFU mode... OK
    Exiting DBMa mode... OK
    ```

## Restoring/reviving with cfgutil
> Alternatively you can use Apple Configurator 2 or [Finder](https://www.macrumors.com/2023/08/15/macos-sonoma-dfu-mode/) to restore/revive your Mac. 

> **Note**: This method with `cfgutil` only works if the target Mac is Apple Silicon.
1. Confirm the device is in DFU mode by launching Apple Configurator 2. It should look like this.
   (If it does not appear, try unplugging and plugging back in the cable between the host and target)

    ![Apple Configurator 2 is now showing target device in DFU](/images/macvdmtool/apple_configurator/dfu.png)

1. Grab the IPSW you want to restore from [Mr. Macintosh's database](https://mrmacintosh.com/apple-silicon-m1-full-macos-restore-ipsw-firmware-files-database/).

1. Use `cfgutil` to restore that IPSW.
    ```shell
    cfgutil restore -I ~/Downloads/UniversalMac_12.1_21C5021h_Restore.ipsw
    ```
    Should result in something like this:
    ```text
    objc[53015]: Class AMSupportURLConnectionDelegate is implemented in both /usr/lib/libauthinstall.dylib (0x1da64c9e8) and /Library/Apple/System/Library/PrivateFrameworks/MobileDevice.framework/Versions/A/MobileDevice (0x104d782c8). One of the two will be used. Which one is undefined.
    objc[53015]: Class AMSupportURLSession is implemented in both /usr/lib/libauthinstall.dylib (0x1da64ca38) and /Library/Apple/System/Library/PrivateFrameworks/MobileDevice.framework/Versions/A/MobileDevice (0x104d78318). One of the two will be used. Which one is undefined.
    cfgutil: restore: target OS is 12.1

    Waiting for the device [1/2] [*******************************************]  100%
    Step 2 of 2: Installing System [2/2] [**********************************>]  99%
    ```

At this point you should see an Apple logo and a loading bar on the target Mac. 

After a few minutes, the Mac will reboot and launch the standard Setup Assistant experience.

That's it! Enjoy.

# Future [automation](https://xkcd.com/1319/) enhancements

- Once Apple adds 12.X IPSWs to the [com_apple_macOSIPSW.xml](https://mesu.apple.com/assets/macos/com_apple_macOSIPSW/com_apple_macOSIPSW.xml) feed, I could write a script to download the latest IPSW and then execute macvdmtool and cfgutil. 
- Autopkg recipe to download latest IPSW and package it with Munki. This could be useful to distribute IPSWs to technicians.

# Have a question, clarification, or comment? 

File an issue or PR (pull request) against my blog at https://github.com/discentem/blog-content or reach out via MacAdmins Slack.

# Further Readings on DFU

See Mr. Macintosh's excellent [blog post on DFU restores](https://mrmacintosh.com/restore-macos-firmware-on-an-apple-silicon-mac-boot-to-dfu-mode/) for more information.






