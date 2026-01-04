---
title: "RDP Bitmap Cache"
date: 2025-12-27 12:00:00 +0300
categories: [Digital Forensics, Investigations]
tags: [Digital Forensics, Cybersecurity]
description: "RDP Bitmap Cache Forensics"
auther: Asrar AlBishri
---

## 🧭 Introduction

Remote Desktop Protocol (RDP) was designed to enable users to connect to other Windows systems through a graphical user interface (GUI). 
To improve the overall RDP user experience and minimize network bandwidth consumption, the RDP Bitmap Cache mechanism was introduced. 
On Windows systems, RDP bitmap cache artifacts are stored within the user profile, typically under the Terminal Server Client c
ache directory. These cache files (such as .bmc and cache.bin) are created when a user establishes an RDP session using the 
built-in mstsc.exe client. The existence of these files can indicate interactive remote access activity and may serve as a valuable
artifact when investigating lateral movement or potentially unauthorized RDP usage.

It is important to note that RDP bitmap cache files are generated only on the client system initiating the connection, not on the 
destination host. As a result, if an attacker accesses a compromised system via RDP from an external or personal device, these 
artifacts are unlikely to be available for collection. However, in scenarios where an attacker moves laterally within the network, 
investigators should focus on identifying the source systems used for those connections and attempt to collect the associated RDP 
bitmap cache files from those endpoints.

By default, the cache is enabled. This setting can be found within the RDP client experience settings as shown below.

![alt text](/assets/img/RDP Options.png)

There are two primary limitations associated with RDP Bitmap Cache artifacts:

**Incomplete Visibility:** The bitmap cache contains small image fragments rather than full screenshots. 
Its purpose is performance optimization, not activity recording. As a result, while the cache may provide useful indicators or partial 
context, it cannot reconstruct an entire RDP session or fully represent user actions.

**Client-Side Storage Only:** RDP bitmap cache files are stored on the system that initiates the RDP connection. In scenarios where a 
threat actor accesses the network through a VPN and launches RDP from their own device, investigators will not have access to that 
endpoint and, consequently, will not be able to retrieve the cache artifacts.

Although RDP bitmap cache artifacts can add valuable context (particularly in cases where traditional logging is limited) they should be 
treated as a supporting artifact rather than a definitive source of truth. They represent one tool within the broader forensic toolkit, 
not a complete investigative solution. In this example, I will leverage the domain controller (DC) image examined in the previous blog, 
as it was confirmed that the DC was used as a pivot point for lateral movement to another device within the network.

 [Blog link](https://dfirmadness.com/case001/DC01-E01.zip){: .btn .btn-primary }

## 🛠️ Tools

There are several methods available to collect RDP bitmap cache files. These artifacts can be acquired using the **KAPE** command shown below
, extracted through **Autopsy** or collected by mounting the disk image with **Arsenal FTK Imager**.

```bash
.\kape.exe --tsource C: --tdest C:\Temp\Collection --target RDPCache --zip RDPCache
```
In this example I have extract them by mounting the image using **FTK Imager** tool.

Additionally, specific tools are required to parse and visualize RDP bitmap cache files. Fortunately, several well-maintained 
open-source tools are available to assist with this process. Some commonly used options include:

 [BMC-Tools](https://github.com/ANSSI-FR/bmc-tools){: .btn .btn-primary }

 [RDP Cache Stitcher](https://github.com/BSI-Bund/RdpCacheStitcher){: .btn .btn-primary }
## 🔍 Details

Firstly, i have accessed the file path and extract bin file.

![alt text](/assets/img/RDP Bin File.png)

Upon extrating .bin file we need to parse it using BMC-Tool to extract and parse the BMC and BIN files containing the RDP cache. 
This will likely be the first step you do before running any additional tools, as this will parse the compressed data and make it 
more readable for other tools.

BMC-Tools will process the cache files and extract the bitmap images, saving them in the output directory. 

 ```bash
python3 bmc-tools.py -s /mnt/c/Users/User/Desktop/Cache/Cache0000.bin -d /mnt/c/Users/User/Desktop/RDPCache/
 ```
![alt text](/assets/img/BMC-Tool Command.png)

After that, we will use RDP Cache Stitcher to start piecing all the images together to hopefully get an indication of what occurred 
during the RDP session.

![alt text](/assets/img/RDP Switcher1.png)

![alt text](/assets/img/RDP Switcher2.png)

As shown, the attacker established persistence on the compromised system by executing a command that creates a scheduled task. 
The referenced URL was used to retrieve and download the malicious payload onto the host.

![alt text](/assets/img/RDP Switcher3.png)

While the RDP Bitmap Cache may not provide a complete or clear reconstruction of a threat actor’s RDP session, 
it can still contain critical fragments that help connect the dots. Even a small snippet may offer the insight needed 
to confirm activity or support the overall investigative narrative.