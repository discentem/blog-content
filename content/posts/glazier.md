---
title: "Getting started with Windows Imaging & Glazier (1/X)"
date: 2021-05-02T15:01:26-04:00
draft: true
---
# What is Glazier? 

From [https://github.com/google/glazier#glazier](https://github.com/google/glazier#glazier) and [https://github.com/google/glazier#why-glazier](https://github.com/google/glazier#why-glazier): 

> Glazier is a tool developed at Google for automating Windows operating system deployments.

> Glazier was created with the following 3 core principles in mind: Text-Based & Code-Driven, Scalability, and Extensibility.


# Sounds awesome. How do I get started?

[The Glazier readme](https://github.com/google/glazier/blob/master/README.md) suggests: 

> See our [docs site](https://google.github.io/glazier) for how you can get started with Glazier.

[The setup overview](https://google.github.io/glazier/setup/), however, leaves a lot of the setup details (WinPE, portable Python, drivers, etc...) as an exercise for the reader. Hence why I am writing this blog post: I hope to demystify some of the setup details and share a reproducible way to create the WinPE environment needed to run Glazier.

Ironically, I may leave some setup details as an exercise for you (ex. creating a Windows 10 virtual machine). But please feel free to reach out to me or submit a pull request if you need more details!

# Setting up Glazier
### Setting up our WinPE build "server"

1. Set up a Windows 10 virtual machine. There many ways to do this but for what it's worth I'll be using VMWare Fusion on my Macbook. 
1. Install Chocolatey on this virtual machine. There are more secure ways of doing this, but for now I am going to be lazy and curl this off the internet :P
    ```powershell
    Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1')) 
    ```
    > Note: Chocolatey is not strictly required to get Windows ADK installed but [installing Windows ADK and the WinPE add-on manually](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install) is daunting. Chocolatey makes it very easy.

1. Relaunch Powershell (again as an Administrator) so that the `choco` command becomes available.
1. Install the Windows ADK chocolatey package. 
    ```powershell
    choco install windows-adk -y
    ``` 
    You should soon see output that looks like this.
    ```
    ...
    The install of windows-adk was successful.
     Software installed as 'exe', install location is likely default.
    Chocolatey installed 1/1 packages.
    See the log for details (C:\ProgramData\chocolatey\logs\chocolatey.log).
    ```
1. Install the Windows ADK WinPE addon via Chocolatey.
    ```powershell
    choco install windows-adk-winpe -y
    ```
    If successful, you'll see something like this:
    ```
    ...
    The install of windows-adk-winpe was successful.
     Software installed as 'exe', install location is likely default.
    Chocolatey installed 1/1 packages.
    See the log for details (C:\ProgramData\chocolatey\logs\chocolatey.log).
    ```
1. Launch Powershell as an Administrator and install the awesome OSD Powershell module. This will get us OSDCloud and some other tools. Mad props to [@SeguraOSD](https://twitter.com/SeguraOSD) for this wizardry. 

    ```powershell
    Install-Module OSD -Force
    ```

   A few important notes/admissions about OSDeploy:
    - [OSDCloud](https://osdcloud.osdeploy.com/) is, within itself, a very cool and powerful imaging tool that you can use **without Glazier**. 
    - **[OSDCloud](https://osdcloud.osdeploy.com/) might be a better choice than Glazier for many admins**. 
    - However, today, I'm only going to borrow it's sorcery to help us bootstrap Glazier :)
    - [OSDCloud](https://osdcloud.osdeploy.com/) probably adds lots of stuff to our WinPE image that we don't actually need for Glazier. 
      - A more experienced Windows Admin might tell us we are really bloating it unecessarily. But for simplicity sake, and because I don't know better, we'll leave the potential bloat in our WinPE image. 

1. Launch a new Powershell with a more permissible execution policy:
    ```powershell
    powershell.exe -ExecutionPolicy Bypass
    ```
1. Import the OSD Module that we installed before.
    ```powershell
    Import-Module OSD
    ```
1. Generate a ["Universal WinPE"](https://osdcloud.osdeploy.com/concepts/universal-winpe). 

    ```powershell
    New-OSDCloud.template -WinRE -Verbose
    ```
    This may take a while, so grab a coffee :) 
    
    To learn more about Universal WinPE, check out https://osdcloud.osdeploy.com/get-started/new-osdcloud.template/winre. 

    *Important Note*: This command pulls some resources from https://github.com/OSDeploy/OSDCloud/ and other places on the internet. Audit the repositories as needed.

    Once this command is finished, you should see: 
    ```
    ...
    2021-05-02-180823 New-OSDCloud.template Completed in 05 minutes 25 seconds
    OSDCloud Template created at C:\ProgramData\OSDCloud
    Get-OSDCloud.template will return C:\ProgramData\OSDCloud
    ```
### Grab drivers from your device fleet

As per Glazier's [boot media requirements](https://github.com/google/glazier/tree/master/docs/setup#requirements-1), we need to make sure our WinPE image includes
   > Any drivers required to enable the local NIC/Video/Storage on the device. (network connectivity is necessary to reach the distribution point.

In order to achieve this, you'll need to do the following steps on **each device model** that you want your Windows 10 image to support. For instance, one of the laptops in my fleet is a `Dell XPS 13 9370`, so I'll be doing these steps on that. But these steps should work on any/all device models that you may want to support.

1. Grab a laptop from your fleet. Make sure Windows 10 is up-to-date on this device and that all the latest drivers are installed.

    > For me, because I am working with a Dell machine, I used [Dell Command Update](https://www.dell.com/support/home/en-us/drivers/driversdetails?driverid=34t96) to update all of the drivers. 
1. On the laptop, launch Powershell as an administrator and set the execution policy to bypass: 
    ```powershell
    powershell.exe -ExecutionPolicy Bypass
    ```
1. Now export all of this device's drivers with this nifty powershell command: [Export-WindowsDriver](https://docs.microsoft.com/en-us/powershell/module/dism/export-windowsdriver?view=windowsserver2019-ps).
   ```powershell
   Export-Windowsdriver -Online -Destination c:\drivers
   ```
   After a few minutes you should see output that looks something like this. (This is a partial snippet showing only drivers that are obviously network related)

   ```
   ...
   Driver           : oem25.inf
   OriginalFileName : C:\Windows\System32\DriverStore\FileRepository\killernetworkextension.inf
                      _amd64_635597853a943a8a\killernetworkextension.inf
   Inbox            : False
   ClassName        : Extension
   BootCritical     : False
   ProviderName     : Rivet Networks LLC
   Date             : 4/16/2020 12:00:00 AM
   Version          : 2.2.3267.0
   ...
   ...
   Driver           : oem28.inf
   OriginalFileName : C:\Windows\System32\DriverStore\FileRepository\killernetworkcomponent.inf
                      _amd64_2caa3873bf7cf75d\killernetworkcomponent.inf
   Inbox            : False
   ClassName        : SoftwareComponent
   BootCritical     : False
   ProviderName     : Rivet Networks LLC
   Date             : 4/16/2020 12:00:00 AM
   Version          : 2.2.3267.0
   ```

1. Copy the entire contents of `C:\drivers` on the laptop to `C:\drivers` on your Windows 10 virtual machine via a flash drive or some other method.
      > Technically we only care about drivers that are related to NIC, Video, or Storage for WinPE. However, because I'm lazy, I'll just copy all of the drivers.
1. Repeat steps 1-4 on each device model that you want to support.


    