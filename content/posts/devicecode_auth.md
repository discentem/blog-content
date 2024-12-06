---
title: "Device Code Auth for secure downloads in WinPE from Azure Blob Storage"
date: 2021-12-31T10:48:42-08:00
draft: true
---

### What is Device Code Authentication? 

Per [IETF (Internet Engineering Task Force) rfc8628](https://datatracker.ietf.org/doc/html/rfc8628), Device Code Authentication (aka `Device Authorization Grant`)

> is designed for Internet-connected devices that either lack a browser to perform a user-agent-
based authorization or are input constrained to the extent that requiring the user to input text in order to authenticate during the authorization flow is impractical. It enables OAuth clients on such
devices (like smart TVs, media consoles, digital picture frames, and printers) to obtain user authorization to access protected resources by using a user agent on a separate device. 

#### Why is this interesting?

I don't manage or write software for IoT devices or Smart TVs. But one input-constrained environment that I have found myself experimenting with is WinPE. In my previous adventure with [Glazier and WinPE](https://bkurtz.io/posts/glazier/) I lamented about how [Glazier lacks authentication out-of-the-box](https://bkurtz.io/posts/glazier#webservers-that-are-open-to-the-internet-are-bad). Device Code can solve this problem without needing additional infrastructure by enabling authenticated downloads from various cloud storage providers.

#### Credit

Huge shout out to [@SeguraOSD](https://twitter.com/SeguraOSD) [for sharing his idea to utilize Device Code auth](https://twitter.com/SeguraOSD/status/1474541279736381440?s=20). I will implement this in Go...which might be useful if I ever get around to writing [an alternative to Glazier in go](https://bkurtz.io/posts/glazier#we-could-write-some-new-glazier-actions-in-go-or-reimagine-the-entire-tool). 

#### Which Device Code implmentation and cloud storage should we use? GCS, S3, or Azure Blob?

- Google does support Device Authorization Grant but [_only for limited API scopes_](https://developers.google.com/identity/protocols/oauth2/limited-input-device#allowedscopes). Authenicating to GCS is not supported using this flow. Thus GCS is not an option for this project. 

- AWS [does appear to support Device Authorization Grant](https://aws.amazon.com/blogs/security/implement-oauth-2-0-device-grant-flow-by-using-amazon-cognito-and-aws-lambda/) and _does not appear_ to limit the available scopes. So S3 seems feasible but I find Amazon's docs on the implementation process to be very confusing. 

In contrast, [Microsoft Azure's docs on device code](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-device-code) seemed more straightforward. Thus I choose Azure Blob storage. 

### Implementation in Golang

I've written Go for many recent projects at work and my team helps mantain a few open-source tools in Go, such as [MDMDirector](https://github.com/mdmdirector/mdmdirector) and [macadmins/osquery-extension](https://github.com/macadmins/osquery-extension), thus Go seemed like an obvious choice.

#### Device Code "from scratch"

Initially I implemented the [Azure Device Code authentication flow "from scratch"](https://github.com/discentem/azure_blob_from_scratch) because I was not yet aware of the [Azure Go SDK](https://github.com/Azure/azure-sdk-for-go) 🥲. Azure SDK for Go provides everything I implemented and much more; it is also much more elegant.

My ["from scratch"](https://github.com/discentem/azure_blob_from_scratch) implementation took many hours to complete. It was a fun adventure but ultimately not needed. 

#### Device with Azure SDK for Go

You can find the official Azure SDK for Go at https://github.com/Azure/azure-sdk-for-go. After much digging around I found the first bread crumb I needed: https://github.com/Azure/azure-sdk-for-go/tree/main/sdk/azidentity#credential-types. Thankfully the devs behind Azure SDK for Go implemented the credential I am after: [DeviceCodeCredential](https://pkg.go.dev/github.com/Azure/azure-sdk-for-go/sdk/azidentity#DeviceCodeCredential).









