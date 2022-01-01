---
title: "Device Code Auth for Azure Blob Downloads"
date: 2021-12-31T10:48:42-08:00
draft: true
---

### What is Device Code Authentication? 

According to [Microsoft's docs](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-device-code), Device Code Authentication

> allows users to sign in to input-constrained devices such as a smart TV, IoT device, or printer. 

#### Why is this interesting?

I don't manage or write software for IoT devices or Smart TVs. But Device Code is still interesting to me because I've previously written about [Glazier and WinPE](https://bkurtz.io/posts/glazier/) lamented about how [Glazier lacks authentication out-of-the-box](https://bkurtz.io/posts/glazier#webservers-that-are-open-to-the-internet-are-bad).  And WinPE is arguably an _input-contrained device_ because you can't launch a full web browser, which is _usually_ required for modern SSO. 

Enter Device Code Authentication. This could allow an imaging tool, running in WinPE, to perform authenticated downloads from Azure Blob Storage. 

#### Credit

Using Device Code Auth in WinPE was not my idea. It was [@SeguraOSD](https://twitter.com/SeguraOSD) who [pointed out its existence to me on Twitter](https://twitter.com/SeguraOSD/status/1474541279736381440?s=20). Thank you! I believe David is going to implement this in his stellar [OSDCloud](https://osdcloud.osdeploy.com/) project.

I will implement this in Go, which might be useful if I ever write [an alternative to Glazier in go](https://bkurtz.io/posts/glazier#we-could-write-some-new-glazier-actions-in-go-or-reimagine-the-entire-tool).

### Implementation in Golang

#### "From scratch"

Initially I implemented the [Azure Device Code authentication flow "from scratch"](https://github.com/discentem/azure_blob_from_scratch) in Go because I was not yet aware of the [Azure Go SDK](https://github.com/Azure/azure-sdk-for-go) 🥲.

This "from scratch" implementation took many hours to complete. It was both fun and frustrating. If you interested in reading the code for this implementation, you can find the code here: https://github.com/discentem/azure_blob_from_scratch. 

However, I would highly recommend just using the Azure SDK for Go as I do below. I just can't bare to discard this code, so here it is for academic purposes. 

#### Azure Go SDK

You can find the sdk at https://github.com/Azure/azure-sdk-for-go. There's 





