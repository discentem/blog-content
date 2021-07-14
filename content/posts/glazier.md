---
title: "Getting started with Windows Imaging & Glazier (Part 1)"
date: 2021-07-10
draft: false
---
# What is Glazier? 

From [https://github.com/google/glazier#glazier](https://github.com/google/glazier#glazier) and [https://github.com/google/glazier#why-glazier](https://github.com/google/glazier#why-glazier): 

> Glazier is a tool developed at Google for automating Windows operating system deployments.

> Glazier was created with the following 3 core principles in mind: Text-Based & Code-Driven, Scalability, and Extensibility.


### Sounds awesome. How do I get started?

[The Glazier readme](https://github.com/google/glazier/blob/master/README.md) suggests: 

> See our [Glazier's] [docs site](https://google.github.io/glazier) for how you can get started with Glazier.

However, [The setup overview](https://google.github.io/glazier/setup/) comes "without the batteries included". The authors expect that you already know how to create [WIMs](https://en.wikipedia.org/wiki/Windows_Imaging_Format), use the [dism](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/dism---deployment-image-servicing-and-management-technical-reference-for-windows) CLI tool, and some other common WinAdmin tasks. These expectations are completely fair. 

But: 
- I didn't know how to use these tools. And
- I was worried that even if I set up Glazier by hand successfully once, I'd never remember the exact steps required to do it again. 

Thus, I worked hard to streamline and automate the setup details and share a reproducible way to create a [WinPE](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-intro) ISO that supports Glazier. I hope you find the results useful :) 

# Set up Glazier

### Create a WinPE "factory"

1. Set up a Windows 10 virtual machine. It can be on any platform: AWS, GCP, VMWare (ESXI or Fusion), etc. We'll use this VM later to create a WinPE ISO.
1. Clone my `glazier-starter-kit` repository. 
   ```cmd
   git clone https://github.com/discentem/glazier-starter-kit.git
   ```
   This repository contains 3 main things:
   - [a powershell script](https://github.com/discentem/glazier-starter-kit/blob/master/tools/bootstrap_winpe.ps1) that automatically generates a WinPE ISO with Glazier and all of its dependencies 🎉
   - [a complete example set](https://github.com/discentem/glazier-starter-kit/tree/master/glazier-repo) of Glazier config files
   - [a golang script](https://github.com/discentem/glazier-starter-kit/blob/master/tools/main.go) that can sync Glazier config files to s3

   I will discuss these things in more detail later on. For now, let's move on to drivers.
### Gather drivers from your device fleet

As per Glazier's [boot media requirements](https://github.com/google/glazier/tree/master/docs/setup#requirements-1), we need to make sure our WinPE image includes any drivers required to enable the local NIC/Video/Storage on the device. Network connectivity during WinPE is necessary to reach the distribution point.

In order to achieve this, you'll need to do the following steps on **each device model** that you want your Windows 10 image to support. One of the laptop models in my fleet is the `Dell XPS 13 9370`, so I'll be doing these steps on that. 

1. Grab a laptop from your fleet. Make sure Windows 10 is up-to-date on this device and that all the latest drivers are installed. I am working with a Dell machine, so I used [Dell Command Update](https://www.dell.com/support/home/en-us/drivers/driversdetails?driverid=34t96) to update all of the drivers. 
1. On the laptop, export all of this device's drivers with [Export-WindowsDriver](https://docs.microsoft.com/en-us/powershell/module/dism/export-windowsdriver?view=windowsserver2019-ps). You will need to launch Powershell as an Administrator.
   ```powershell
   Export-Windowsdriver -Online -Destination c:\drivers
   ```
   During the execution of this command, you'll see output like this. This is only a partial snippet of the output.

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
      
      Recall that we technically only need drivers related to NIC, Video, or Storage for WinPE. However, because I'm lazy, I'll just copy all of the drivers.
      
      After copying `C:\drivers` back to the virtual machine, the directory should look something like this, with each subfolder containing the files for an individual driver: 
      (partial snippet)
      ```cmd
      PS C:\Users\brandon> ls C:\drivers\
      
      Directory: C:\drivers

      Mode                 LastWriteTime         Length Name
      ----                 -------------         ------ ----
      d-----         6/19/2021   5:01 PM                atheros_bth.inf_amd64_059e038b6e018050
      d-----         6/19/2021   5:01 PM                cui_dch.inf_amd64_b8e01d9e8716d2a7
      d-----         6/19/2021   5:01 PM                detectionverificationdrv.inf_amd64_dcc202b5af4a34b5
      ...
      ```
1. Repeat the above steps 1-3 above on each device model that you want to support.


### Setting up Glazier distribution point

#### Set up a web server

Per [https://github.com/google/glazier/tree/master/docs/setup#distribution-point](https://github.com/google/glazier/tree/master/docs/setup#distribution-point), we need to set up a web server to host our Glazier config files. I'm going to use [Amazon S3](https://aws.amazon.com/s3/) but you can use any web server that supports https. A few important notes:
   - By default, Glazier does not provide web authentication for downloading resources. So your web server needs to be open to the internet, behind a network ACL, or have some other form of device-based authentication.
   - [Fresnel](https://github.com/google/fresnel), another open-source project from Google's WinOps team, can provide authenticated downloads for Glazier. However, Fresnel is out-of-scope for this post. 
   

**If you aren't using s3**: set up your own web service and skip to [Get familiar with Glazier](#get-familiar-with-glazier).

##### Set up S3

1. Create an S3 bucket. You can do this via the AWS Console, via [Terraform's AWS provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket), or another tool of your choice. For more information see [https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-bucket.html](https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-bucket.html).

1. Turn off the `Block Public Access` settings. 
    > The AWS console (screenshot below) will show you persistent warnings that you should not set "Block Public Acccess" to "Off" because it is generally _not_ recommended for any service hosted on s3. However, we need to do this for Glazier because we do not have [Fresnel](https://github.com/google/fresnel) set up. 
    
    **Disclaimer: If you set up Glazier for production, you should probably turn `Block Public Access` on again and set up Fresnel or another authentication mechanism.**

    ![s3 public access](/images/s3/public_access.png)

1. Grant `Everyone (public)` read and list permissions on the s3 bucket.

   **Disclaimer: If you set up Glazier for production, you should probably not do this. Instead, set up Fresnel or consult with a Security Engineer to design some authentication mechanism.**

    ![s3 everyone list and read](/images/s3/everyone_list_read.png)

#### Get familiar with Glazier 

1. Take a few minutes to read over the following resources. These docs will help you become familiar the layout and syntax of Glazier configuration files so that you can customize Glazier for your environment.

   - https://google.github.io/glazier/setup/config_layout. This page goes into great detail about the layout of a [Glazier distribution point](https://github.com/google/glazier/tree/master/docs/setup#distribution-point) and Glazier config files. 
   - https://github.com/google/glazier/tree/master/examples has a few good examples of `build.yaml` files which show off some of the actions Glazier is capable of.
   - https://github.com/google/glazier/blob/master/docs/actions.md#actions. This page describes all of the existing Glazier actions (aka functions). You can also [create your own actions](https://github.com/google/glazier/blob/master/docs/setup/new_actions.md).
   
1. If desired, fork `glazier-starter-kit/glazier-repo` and/or customize as necessary.

   You don't necessarily need to customize anything at first. `glazier-starter-kit/glazier-repo` contains a _complete_ set of config files. [These basic configs](https://github.com/discentem/glazier-starter-kit/blob/master/glazier-repo/stable/config/build.yaml#L3) won't accomplish very much but they will enable you to create a proof-of-concept setup of Glazier.

#### Upload Glazier configs

1. Upload the contents of `glazier-starter-kit/glazier-repo`, with any additions/modifications you've made, to your web server. The root of your web server should look something like this: 
   ```shell
   % ls -1
   dev
   stable
   version-info.yaml
   ```
   <details>
      
      <summary>[<--- Expand here] If you are using Amazon S3 as a distribution point, you can use a <a href=https://golang.org/>Go</a> binary I wrote to upload Glazier configs to your Glazier distribution point.</summary>
      <body>
      <p>&emsp;Use my <a href=https://github.com/discentem/glazier-starter-kit/blob/master/tools/main.go>main.go</a> script:</p>
      <ul>
      <li>Set up an IAM user for S3 uploads. See <a href=https://docs.easydigitaldownloads.com/article/1455-amazon-s3-creating-an-iam-user>https://docs.easydigitaldownloads.com/article/1455-amazon-s3-creating-an-iam-user</a>.</li>
      <li>Set up the <a href="https://github.com/discentem/glazier-starter-kit/blob/master/tools/cmd/root.go#L60-L72">required environment variables</a>.</li>
      <li>Install <a href=https://golang.org/dl/>go</a>.</li>
      <li>Run <text style="font-family:monospace,monospace;font-size: 1em">go run main.go sync</text> from within <text style="font-family:monospace,monospace;font-size: 1em">glazier-starter-kit/tools/</text>.</li>
      </ul>
      </body>
   </details>
1. Make note of web server's url. If you are using s3, it will be in the form of https://yourbucketname.s3.amazonaws.com/.

#### Create the WinPE ISO and USB
Back on your virtual machine

1. Launch Powershell as an Administrator again and start a new session with an execution policy of bypass. 
   ```cmd
   powershell.exe -ExecutionPolicy Bypass
   ```
1. Run the `bootstrap_winpe.ps1` script. Note, the script may be in a different location on your machine, depending on where you cloned `glazier-starter-kit`.
   ```cmd
   C:\glazier-starter-kit\tools\bootstrap_winpe.ps1 --config_server https://YOUR_WEBSERVER_OR_BUCKETNAME_URL_GOES_HERE
   ```
   After a few minutes you should see the output end like this.
   ```cmd
   Done.
   NoPromptIso: C:\OSDCloud\OSDCloud_NoPrompt.ISO
   =========================================================================
   2021-06-27-142237 New-OSDCloud.ISO Completed in 00 minutes 10 seconds
   OSDCloud ISO created at C:\OSDCloud\OSDCloud.ISO
   =========================================================================
   ``` 
   
   For a full breakdown of what this script does, check out the code comments at https://github.com/discentem/glazier-starter-kit/blob/master/tools/bootstrap_winpe.ps1. In summary [bootstrap_winpe.ps1](https://github.com/discentem/glazier-starter-kit/blob/master/tools/bootstrap_winpe.ps1):
   - Installs the Chocolatey package manager
   - Installs Windows ADK and the WinPE components with Chocolatey
   - Installs and imports the [OSDeploy](https://github.com/OSDeploy/OSD) powershell module
   - Creates a [OSDCloud template](https://osdcloud.osdeploy.com/get-started/new-osdcloud.template) and a [OSDCloud workspace](https://osdcloud.osdeploy.com/functions/osdcloud.workspace)
   - Downloads Python
   - Installs your drivers into the wim
   - Mounts a `boot.wim`
   - Installs Python (in the wim)
   - Installs git (with chocolatey)
   - Clones [Glazier](https://github.com/google/glazier) into the mounted wim and runs git pull
   - Installs Glazier's python requirements
   - Writes a custom `startnet.cmd` which will prompt for Wifi and auto start Glazier upon booting the wim
   - Generates an ISO with all of the above

   It is safe to run the script multiple times. Some steps are idempotent. Others are repeated unnecessarily but will result in the correct ISO.


1. Use your favorite tool to create a bootable usb of `C:\OSDCloud\OSDCloud_NoPrompt.ISO`. I've been using [New-OSDCloud.usb](https://osdcloud.osdeploy.com/get-started/new-osdcloud.usb) but theoretically UNetBootin, Rufus, or another similiar tool will work just fine.

#### Start "Imaging"
1. Boot one of the machines in your fleet from your newly created usb.
1. Upon booting the usb, if you aren't connected to ethernet, you'll be prompted to select a Wifi network.

   <img src="/images/winpe/1.png" alt="Picture of a CLI Wifi selection menu" width="1500"/>

   <img src="/images/winpe/2.png" alt="Picture of a Windows credential password prompt" width="1500"/>

1. After getting connected to the internet, `autobuild.ps1` will run. This is heavily borrowed from [@tseknet's](https://github.com/TsekNet) example [autobuild.ps1](https://github.com/google/glazier/commit/6e9b94bfeaa0a0b5c6a908c337c0a488cd6fe600#diff-4130388a9ea73c798df0683813708bfd3cdfb1e25ff8d4e4ddf2af10746c7691). Thanks again for sharing that!

   ```cmd
   X:\windows\system32>powershell -NoProfile -NoLogo -WindowStyle Maximized -NoExit
   -File "X:\Windows\System32\autobuild.ps1" -config_server https://YOUR_WEBSERVER_OR_BUCKETNAME_URL_HERE
   ```

1. If you are using Glazier with my [original config files](https://github.com/discentem/glazier-starter-kit/tree/master/glazier-repo), next you'll see this screen. 

   <img src="/images/winpe/8.png" alt="Picture of the UI notifying user that build will start in 1 minute" width="800"/>

   You see this because I configured it in [this build.yaml file](https://github.com/discentem/glazier-starter-kit/blob/master/glazier-repo/stable/config/build.yaml#L9-L19). See https://google.github.io/glazier/yaml/chooser_ui for more info.

    ```yaml
   - pin: ''
      choice:
         name: test_thing
         type: toggle
         prompt: "test toggle"
         options: [
            {label: 'False', value: False, tip: ''},
            {label: 'True', value: True, tip: ''},
         ]
   - pin: ''
      ShowChooser: []
   ```

   The logo is copied from https://github.com/discentem/glazier-starter-kit/blob/master/glazier-resources/logo.gif to the [WIM](https://en.wikipedia.org/wiki/Windows_Imaging_Format): https://github.com/discentem/glazier-starter-kit/blob/master/tools/bootstrap_winpe.ps1#L132. 
   
   Glazier expects the logo to be in the [resources directory](https://github.com/discentem/glazier-starter-kit/blob/master/tools/autobuild.ps1#L18): https://github.com/google/glazier/blob/master/glazier/chooser/chooser.py#L98. 

1. After clicking "Image now" or waiting 60 seconds, Glazier will run the powershell commands I [configured it to run](https://github.com/discentem/glazier-starter-kit/blob/master/glazier-repo/stable/config/build.yaml#L3). 

   <img src="/images/winpe/9.png" alt="Picture of the output of Get-Time running inside of Glazier" width="800"/>

**Yay! We did it! Wait. What did we do exactly?**

We set up Glazier in a repeatable fashion and configured it to do some basic stuff. It was a proof-of-concept, but we can do much more! See https://google.github.io/glazier/actions for all the things Glazier can do.

### Potential next steps for you or for a follow-up blog post
#### We should _actually_ partition the computer we were "imaging".
- **Problem**: You might have noticed we didn't really "image" anything. We just told Glazier to [give us the date](https://github.com/discentem/glazier-starter-kit/blob/master/glazier-repo/stable/config/build.yaml#L3) and exit 🥲. 
- **Solution**: Write a powershell script that actually partitions the drive and expands a vanilla Windows 10 wim onto the drive. Something [like this](https://github.com/OSDeploy/OSD/blob/master/Public/OSDCloud/Invoke-OSDCloud.ps1#L428) from OSDCloud.

#### Webservers that are open to the internet are bad.

- **Problem**: Glazier does not provide an authentication method for clients out-of-the-box. It is not secure by default. 
- **Solution**: Spin up [Google's Fresnel project](https://github.com/google/fresnel)
- **Stretch solution**: Write equivalent of Fresnel that can be hosted somewhere other than AppEngine.

#### We should automate some "post-imaging" tasks once booted into the host OS.
- **Problem**: Even if we solve partitioning the drive and install vanilla Windows 10, we still haven't done anything novel or special. We should also install some packages such as a configuration management tool ([Puppet](https://puppet.com/try-puppet/open-source-puppet/download/) or [Saltstack](https://saltproject.io/)) and a package manager ([Chocolatey](https://chocolatey.org/), [Googet](https://github.com/google/googet), or [Gorilla](https://github.com/1dustindavis/gorilla)).
- **Solution**: Add some Glazier config to automatically log in after the host OS is installed and do stuff. See https://github.com/google/glazier/blob/master/docs/setup/README.md#images--sysprep for some ideas.

#### We could write some new Glazier actions in go or re-imagine the entire tool.
- **Problem**: Setting up Python in WinPE is a giant pain. If Glazier or Glazier configs revolved around single go binary setup would be easier. It seems like the folks at Google [have the same idea](https://github.com/google/glazier/tree/master/go). But none of the Glazier actions currently use this code (yet). 

- **Possible Solutions**:
   - Write some Glazier actions that call the above Go code
   - A somewhat out-there idea: Write a completely new imaging tool that uses starlark instead of yaml with https://github.com/google/starlark-go. 

### Have a questions, clarification, or comment? 

File an issue or PR (pull request) against my blog at https://github.com/discentem/blog-content. If you want to make a PR against this specific post, you'll find the source for this post [here](https://github.com/discentem/blog-content/blob/master/content/posts/glazier.md).

### Credits

- Thank you to Google WinOps for writing Glazier. This tool has really captured my imagination.
- Thank you [@contains_eng](https://twitter.com/contains_eng) for inspiring me to write something to help people set up Glazier. Sorry it's a few years late :P
- Thank you [@tseknet](https://github.com/TsekNet) for always answering my questions and always jumping at the chance to improve Glazier's docs. You rock! I miss working with you buddy. 
- Thank you [@seguraosd](https://twitter.com/seguraosd) for writing the awesome OSDCloud tool.
- Thank you [@editingemily](https://twitter.com/editingemily) for talking openly at your [MacDevOpsYVR keynote](https://www.youtube.com/watch?v=zn_t5JvlANI) about your struggle/journey to write a book. The scale of this blog post is nothing compared to a book, but you still inspired me to keep pushing through when I was struggling.
- Thank you [@groob](https://twitter.com/wikiwalk) for introducing to me to [starlark-go](https://github.com/google/starlark-go). I haven't used it for any production tools yet but I keep trying to find a use case :) 