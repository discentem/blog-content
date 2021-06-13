---
title: "Getting started with Windows Imaging & Glazier (Part 1)"
date: 2021-06-06
draft: true
---
# What is Glazier? 

From [https://github.com/google/glazier#glazier](https://github.com/google/glazier#glazier) and [https://github.com/google/glazier#why-glazier](https://github.com/google/glazier#why-glazier): 

> Glazier is a tool developed at Google for automating Windows operating system deployments.

> Glazier was created with the following 3 core principles in mind: Text-Based & Code-Driven, Scalability, and Extensibility.


# Sounds awesome. How do I get started?

[The Glazier readme](https://github.com/google/glazier/blob/master/README.md) suggests: 

> See our [docs site](https://google.github.io/glazier) for how you can get started with Glazier.

[The setup overview](https://google.github.io/glazier/setup/), however, leaves a lot to be desired. Hence why I am writing this blog post: 

I hope to demystify some of the setup details and share a reproducible way to create the WinPE environment needed to run Glazier. 

# Setting up Glazier
### Setting up our WinPE "build server"

> **Note**: All steps in the section, except for step 1, should be performed on a VM or some other machine that can dedicated to building WinPE. I'll often refer to this dedicated machine as your Windows 10 VM or virtual machine, even though it could work on a physical one.

1. Set up a Windows 10 virtual machine. There many ways to do this but for whatever it's worth I'll be using [VMWare Fusion](https://www.vmware.com/products/fusion.html) on my Macbook. 
1. On the virtual machine: launch Powershell via Run As Administrator and install Chocolatey. There are more secure ways of doing this, but I'm going to be lazy and curl this off the internet 😝
    ```powershell
    Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1')) 
    ```
   Chocolatey is not strictly required to get Windows ADK installed but [installing Windows ADK and the WinPE add-on manually](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install) seems daunting. Chocolatey makes it very easy. 
1. Install Windows ADK. 
    ```powershell
    choco install windows-adk -y
    ``` 
    After a few minutes, you should see output like this:
    ```
    ...
    The install of windows-adk was successful.
     Software installed as 'exe', install location is likely default.
    Chocolatey installed 1/1 packages.
    See the log for details (C:\ProgramData\chocolatey\logs\chocolatey.log).
    ```
1. Install the Windows ADK WinPE addon.
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
1. Install the OSD Powershell module. This will get us OSDCloud and a suite of other Powershell functions. Shout out to [@SeguraOSD](https://twitter.com/SeguraOSD) for [OSD](https://osd.osdeploy.com/)!

    ```powershell
    (Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force) -and (Install-Module OSD -Force)
    ```

   A few important admissions about my usage of OSDCloud:
   - [OSDCloud](https://osdcloud.osdeploy.com/) is meant to be a standalone and powerful imaging tool that you can absolutely use **without Glazier**.
   - **OSDCloud might be a better choice than Glazier for many admins**. 
   - I'm using only a handful of OSDCloud's features to help us bootstrap Glazier and also get Wifi support in WinPE.
   - OSDCloud probably adds lots of stuff to our WinPE image that we don't actually need for Glazier. 
      - A Windows Admin with more experience than me might say that we are bloating our WinPE unecessarily. They would be correct. But for simplicity sake, we'll leave the bloat in our WinPE image. 
1. Import the OSD Module that we just installed.
    ```powershell
    Import-Module OSD
    ```
1. Generate a ["Universal WinPE"](https://osdcloud.osdeploy.com/concepts/universal-winpe). 

    ```powershell
    New-OSDCloud.template -WinRE -Verbose
    ```
    This command took my VM about ~6 minutes to run, so grab a coffee :) 

   > **Note**: This command pulls some resources from https://github.com/OSDeploy/OSDCloud/ and other places on the internet. Audit the repositories as needed.

    Once this command is finished, you should see: 
    ```
    ...
    2021-06-06-110050 New-OSDCloud.template Completed in 05 minutes 55 seconds
    OSDCloud Template created at C:\ProgramData\OSDCloud
    ```
1. Create an [OSDCloud Workspace](https://osdcloud.osdeploy.com/get-started/new-osdcloud.workspace).

    ```powershell
    New-OSDCloud.workspace -WorkspacePath C:\OSDCloud
    ```

   ```
   ...
   2021-06-06-111710 New-OSDCloud.workspace Completed in 00 minutes 02 seconds
   OSDCloud Workspace created at C:\OSDCloud
   ```

   OSDCloud Workspaces allow you to juggle multiple, diverging WinPEs simulatenously without having to overwrite your work. We won't need to take advantage of this OSDCloud feature for Glazier, but it's a pretty neat idea!

### Install Python and Glazier

Back on your Windows 10 VM:

1. Download Python
   ```shell
   curl https://www.python.org/ftp/python/3.9.5/python-3.9.5-amd64.exe -UseBasicParsing -OutFile ~/Downloads/python3.9.5-amd64.exe
   ```
    - The `-UseBasicParsing` flag avoids the need to interactively lauch Internet Explorer. See [https://github.com/MicrosoftDocs/PowerShell-Docs/blob/live/reference/5.1/Microsoft.PowerShell.Utility/Invoke-WebRequest.md#-usebasicparsing](https://github.com/MicrosoftDocs/PowerShell-Docs/blob/live/reference/5.1/Microsoft.PowerShell.Utility/Invoke-WebRequest.md#-usebasicparsing) for more info.

1. Install Python to `C:\OSDCloud\Autopilot`

   ```powershell
   C:\Users\brandon_kurtz\Downloads\python3.9.5-amd64.exe TargetDir=C:\OSDCloud\Autopilot\Python39\ Include_launcher=0 /passive
   ```

    - If you are wondering why we are installing to `C:\OSDCloud\Autopilot`: 
      - ODSCloud automatically copies all the files in this folder to WinPE
      - I'm too lazy to figure out "the correct" way to copy arbitrary files to OSDCloud generated WinPE.

   > The Python Installer GUI will pop up to show installation progress but no user interaction is required. If you'd rather the GUI did not pop up at all, you can pass `/quiet` instead of `/passive` per Python's [Installing without UI](https://docs.python.org/3/using/windows.html#installing-without-ui) docs.
1. Install Git

   ```powershell
   choco install git -y
   ```
1. Relaunch powershell so that the `git` command becomes available and then clone Glazier from Github
   ```powershell
   git clone https://github.com/google/glazier.git C:\OSDCloud\Autopilot\glazier
   ```

   ```
   Cloning into 'C:\OSDCloud\Autopilot\glazier'...
   remote: Enumerating objects: 2515, done.
   remote: Counting objects: 100% (287/287), done.
   remote: Compressing objects: 100% (164/164), done.
   remote: Total 2515 (delta 140), reused 225 (delta 114), pack-reused 2228 eceiving objects:  98% (2465/2515)
   Receiving objects: 100% (2515/2515), 746.62 KiB | 2.05 MiB/s, done.
   Resolving deltas: 100% (1806/1806), done.
   ```

1. Install Glazier's Python requirements
   ```powershell
   C:\OSDCloud\Autopilot\Python39\python.exe -m pip install -r C:\OSDCloud\Autopilot\glazier\requirements.txt
   ```

   ```
   Collecting gwinpy
   Cloning git://github.com/google/winops (to revision master) to c:\users\brandon_kurtz\appdata\local\temp\pip-install-x1on25fz\gwinpy_e87728180e5f4e7b86151759bc7df024
   Running command git clone -q git://github.com/google/winops 'C:\Users\brandon_kurtz\AppData\Local\Temp\pip-install-x1on25fz\gwinpy_e87728180e5f4e7b86151759bc7df024'
   Collecting absl-py
   ...
   Installing collected packages: urllib3, six, idna, chardet, certifi, requests, PyYAML, pyfakefs, ntplib, mock, gwinpy, absl-py
   WARNING: The script chardetect.exe is installed in 'C:\OSDCloud\Autopilot\Python39\Scripts' which is not on PATH.
   Consider adding this directory to PATH or, if you prefer to suppress this warning, use --no-warn-script-location.
    Running setup.py install for gwinpy ... done
   Successfully installed PyYAML-5.4.1 absl-py-0.12.0 certifi-2020.12.5 chardet-4.0.0 gwinpy-0.3.0 idna-2.10 mock-4.0.3 ntplib-0.3.4 pyfakefs-4.4.0 requests-2.25.1 six-1.16.0 urllib3-1.26.4

1. We also need to install pywin32
   ```powershell
   C:\OSDCloud\Autopilot\Python39\python.exe -m pip install pywin32
   ```
   ```
   Collecting pywin32
   Downloading pywin32-300-cp39-cp39-win_amd64.whl (9.2 MB)
      |████████████████████████████████| 9.2 MB 3.3 MB/s
   Installing collected packages: pywin32
   Successfully installed pywin32-300
   ```

### Grab drivers from your device fleet

As per Glazier's [boot media requirements](https://github.com/google/glazier/tree/master/docs/setup#requirements-1), we need to make sure our WinPE image includes
   > Any drivers required to enable the local NIC/Video/Storage on the device. Network connectivity during WinPE is necessary to reach the distribution point.

In order to achieve this, you'll need to do the following steps on **each device model** that you want your Windows 10 image to support. One of the laptop models in my fleet is the `Dell XPS 13 9370`, so I'll be doing these steps on that. 

1. Grab a laptop from your fleet. Make sure Windows 10 is up-to-date on this device and that all the latest drivers are installed. I am working with a Dell machine, so I used [Dell Command Update](https://www.dell.com/support/home/en-us/drivers/driversdetails?driverid=34t96) to update all of the drivers. 
1. On the laptop, export all of this device's drivers with [Export-WindowsDriver](https://docs.microsoft.com/en-us/powershell/module/dism/export-windowsdriver?view=windowsserver2019-ps).
   ```powershell
   Export-Windowsdriver -Online -Destination c:\drivers
   ```
   After a few minutes you should see output that looks something like this. This is only a partial snippet of the output.

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

1. Copy the entire contents of `C:\drivers` on the laptop to `C:\drivers` on your Windows 10 virtual machine. 
      > Recall that we technically only need drivers related to NIC, Video, or Storage for WinPE. However, because I'm lazy, I'll just copy all of the drivers.
1. Repeat the above steps 1-3 above on each device model that you want to support.

### Create a Glazier WinPE flash drive
1. Create your WinPE WIM, with all the drivers you've gathered.

   ```powershell
   EditOSDCloud.winpe -DriverPath c:\drivers
   ```
   After a few minutes, you should see something like this: 
   ```
   2021-05-04-195925 Edit-OSDCloud.winpe Completed in 05 minutes 52 seconds
   ```

1. Create the iso
1. Create the usb
1. Edit winpeinit

### Setting up Glazier server (finally!)

#### Read the docs

1. Take a few minutes to read over the following resources. These resources will help you become familiar the layout and syntax of Glazier configuration files. 

   - https://google.github.io/glazier/setup/config_layout. This page goes into great detail about the layout of a Glazier distribution point and Glazier config files. 
   - https://github.com/google/glazier/blob/master/docs/actions.md#actions. This page describes all of the existing Glazier actions (aka functions). You can also [create your own actions](https://github.com/google/glazier/blob/master/docs/setup/new_actions.md).
   - https://github.com/google/glazier/tree/master/examples has a few good examples of `build.yaml` files which show off some of the actions Glazier is capable of.

   Bookmark these for later when you want to customize your glazier setup. But for now, in order to get you started quickly:

#### "Writing" your first Glazier Configs

1. Clone my `glazier-config` repository. This repository contains an extremely basic but _completely functional_ set of Glazier config files. These configs won't do very much without some modifications but they will enough for a Glazier proof-of-concept!
   ```shell
   git clone https://github.com/discentem/glazier-config.git
   ```
   Feel free to make modifications as you wish. Remember to check out the above Glazier documentation. Also [https://github.com/google/glazier/tree/master/examples/yaml](https://github.com/google/glazier/tree/master/examples/yaml) may be helpful. 

#### Set up a Glazier Distribution Point (aka the web server)

Per [https://github.com/google/glazier/tree/master/docs/setup#distribution-point](https://github.com/google/glazier/tree/master/docs/setup#distribution-point), we need set up a web server to host our Glazier config files. I'm going to use [Amazon S3](https://aws.amazon.com/s3/) but you can use absolutely anything that can act as an https web server. A few important notes:
   - **By default, Glazier does not provide authentication for downloading resources**. So your web server needs to be open to the internet. 
   - If you have [Fresnel](https://github.com/google/fresnel) infrastructure set up and pass the right flags to `autobuild.py` you can do authenticated downloads. I may cover Fresnel in a future blog post but it is out-of-scope for this post.
   

**If you aren't using s3**, set up your own web service and skip past the "Assuming S3" section.

##### Assuming S3

1. Create an S3 bucket. You can do this via the AWS Console, via [Terraform's AWS provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket), or another tool of your choice. For more information see [https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-bucket.html](https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-bucket.html).

1. Turn off the `Block Public Access` settings. 
    > The AWS console will show you persistent warnings that you should not do this because it is generally _not_ recommended. However, we need to do this for Glazier because we do not have [Fresnel](https://github.com/google/fresnel) set up.

    ![s3 public access](/images/s3/public_access.png)

1. Grant `Everyone (public)` read and list permissions on the s3 bucket.

    ![s3 everyone list and read](/images/s3/everyone_list_read.png)

##### Upload Glazier configs

1. Upload your Glazier Configs to your web server.
   <details>
      
      <summary>If you are using Amazon S3 and want a fancy go binary for syncing content to your Glazier server</summary>
      <pre>    </pre>
      Use my <a href=https://github.com/discentem/glazier-config/blob/master/sync.go>sync.go</a> script:
      <ul>
      <li>Set up an IAM user for S3 uploads. See <a href=https://docs.easydigitaldownloads.com/article/1455-amazon-s3-creating-an-iam-user>https://docs.easydigitaldownloads.com/article/1455-amazon-s3-creating-an-iam-user)</a>.</li>
      <li>Set up the <a href=https://github.com/discentem/glazier-config/blob/master/cmd/root.go#L59-L61>required environment variables</a></li>
      <li>Install <a href=https://golang.org/dl/>go</a></li>
      <li>Run <text style="font-family:monospace,monospace;font-size: 1em">go run sync.go</text> from the root of the <text style="font-family:monospace,monospace;font-size: 1em">glazier-config</text> repo.</li>
      <li> Profit! </li>
      </ul>
   </details>
