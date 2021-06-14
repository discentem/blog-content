---
title: "Getting started with Windows Imaging & Glazier (Part 1)"
date: 2021-06-12
draft: true
---
# What is Glazier? 

From [https://github.com/google/glazier#glazier](https://github.com/google/glazier#glazier) and [https://github.com/google/glazier#why-glazier](https://github.com/google/glazier#why-glazier): 

> Glazier is a tool developed at Google for automating Windows operating system deployments.

> Glazier was created with the following 3 core principles in mind: Text-Based & Code-Driven, Scalability, and Extensibility.


### Sounds awesome. How do I get started?

[The Glazier readme](https://github.com/google/glazier/blob/master/README.md) suggests: 

> See our [docs site](https://google.github.io/glazier) for how you can get started with Glazier.

[The setup overview](https://google.github.io/glazier/setup/), however, leaves a lot to be desired. Hence why I am writing this blog post: 

I hope to demystify some of the setup details and share a reproducible way to create the WinPE environment needed to run Glazier. 

# Setting up Glazier
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


### Create a WinPE "build server"

1. Set up a Windows 10 virtual machine. 
1. On the virtual machine: launch Powershell via Run As Administrator and set a permissive execution policy. 
   ```powershell
   powershell.exe -ExecutionPolicy Bypass
   ```
1. Clone my `glazier-config` repository. This repo contains various tools, including a script that will automatically create a WinPE iso containing Glazier and all of it's dependencies. We'll poke around later.
   ```powershell
   git clone https://github.com/discentem/glazier-config.git
   ```

### Setting up Glazier server

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
