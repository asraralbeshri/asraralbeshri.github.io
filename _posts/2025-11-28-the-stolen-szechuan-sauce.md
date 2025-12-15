---
title: "The Stolen Szechuan Sauce - DC"
##date: 2025-28-11 12:00:00 +0300
categories: [Digital Forensics, Investigations]
tags: [Digital Forensics, Cybersecurity]
description: "The Stolen Szechuan Sauce DC Investigations"
by: Asrar AlBishri
---

## Introduction

This 

## Used Tools
I have used Arsenal FTK Imager to mount the image and use the below Kape command to find and parse the most critical artificats.

```bash
kape.exe --tsource F: --tdest D:\KAPE --msource D:\KAPE --mdest D:\KAPE\parsed --t KapeTriage --m !EZParser
```

I have also use Autopsy to extract further artifcats (Hives and Suspecuios Files)

## Details

I have analyzed the event logs and filtered for Event ID 4625 (failed logins), 
which revealed 95 failed logon attempts targeting the Administrator account. 
All attempts originated from a Kali host within the same short timeframe (03:21:25–03:21:46) 
indicating a brute-force attack.

![alt text](/assets/img/Failed Logins.png)

I have also reviewed the logs for Event ID 4624 (success logins) and confirmed that a successful logon 
occurred shortly after the failed attempts. The attacker successfully authenticated and multiple connections 
were initiated from the IP address 194.64.24[.]102. This indicates that initial access was achieved through 
an RDP brute-force attack against the domain controller. The first successful login was recorded on 2020-09-19 at 03:21:48.

![alt text](/assets/img/success login.png)

After the login, RDP logs (131,1149) were recorded indicating RDP was used to login to the DC after the logged in activity 
at 2020-09-19 03:22:07.

![alt text](/assets/img/RDP.png)

At 2020-09-19 03:23:41, a connection to the highlighted URL was observed. This URL corresponds to the public IP address 
used to access the domain controller. Accessing the link directly via its public IP is unusual and is commonly 
associated with a suspicious activity.

![alt text](/assets/img/public ip url.png)

At 2020-09-19 03:24:12, analysis of the MFT entries revealed the creation of a suspicious file named coreupdater.exe 
within the System32 directory. The timing and context indicate that this file was likely downloaded using the 
previously mentioned URL.

![alt text](/assets/img/suspicious file.png)


The review of the USN Journal (USNJrnl) entries shows that the file was initially downloaded into the Downloads 
folder and later moved into the System32 directory. This is proofed from the presence of two different parent entry 
numbers associated with the file: 84880, corresponding to the Downloads folder where the file was first created
and 2873, corresponding to the System32 folder where the file was subsequently moved.

![alt text](/assets/img/USN Journal.png)

As we can see 84880 is the entry number of Downloads directory

![alt text](/assets/img/Downloads Dir.png)

Entry number 2873 is the entry number of System32 directory

![alt text](/assets/img/System32 Dir.png)

The file content indicates that suspicious or malicious activity would be carried out upon execution.

![alt text](/assets/img/File Content.png)

I have extracted the file via Autopsy and calculated its MD5 hash using the below PSn command, confirming that it is 
a clearly malicious executable. I suggests that this file was dropped during the attacker activity.

```bash
Get-FileHash -Algorithm MD5 .\coreupdater.exe
```

![alt text](/assets/img/VirusTotal Hash.png)

I have searched for this file hash across online sandbox platforms and confirmed that it is associated with 
Meterpreter/Metasploit activity. The analysis also indicates that the command-and-control (C2) server used 
by the malware was 203.78.103[.]109.

![alt text](/assets/img/Malware Analysis.png)

Two techniques used for persistence:
Services:  as shown below a service has been created named coreupfater

![alt text](/assets/img/Service Persistence.png)

And registry key: Microsoft\Windows\CurrentVersion\Run is used to execute programs automatically at system startup.
During analysis, I have identified an unusual encoded script configured to run at boot. The presence of encoded 
content in a Run key is uncommon for legitimate software and is a well-known persistence technique used by attackers. 
Further review showed that the Run key value points to another registry location, 38283-SOFTWARE\9sEoCawv with the 
associated value 45SVAG2o.

![alt text](/assets/img/Run Key.png)

45SVAG2o value contain base64 encoded data. Note that last written date of the key was at 19/9/2025 3:30:01.

![alt text](/assets/img/Key Value.png)

Using CyberChef to decode the data, I have discovered an additional layer of encoded content embedded within the script. 
This indicates that the attacker deliberately used multiple encoding stages to obscure the payload and evade detection.

![alt text](/assets/img/Stage1.png)

The decoded data of the above encoded data

![alt text](/assets/img/Stage1Decoded.png)

We attempted to decode the embedded encoded data using the CyberChef modules shown below.

![alt text](/assets/img/Stage2.png)

The decoded data of the above encoded data indicating fileless loader / reflective shellcode injector. The loader will: 
1.	Defines helper functions that wrap native Windows API calls (via .NET reflection and P/Invoke).
2.	Contains a large Base64 blob ($gri) which is binary shellcode (or a binary payload) encoded in Base64.
3.	Allocates executable memory in the current process, writes the decoded shellcode into that memory, and then creates a thread to execute it — all done in memory (no file written to disk).
4.	The payload is therefore a memory-resident, in-memory shellcode execution (typical for malware trying to evade AV and forensic detection).

![alt text](/assets/img/Stage2Decoded.png)

I have also examined the Recent Files directory located at C:\Users\Administrator\AppData\Roaming\Microsoft\Windows\Recent 
and identified several files that had been accessed. The activity suggests that the Administrator account recently 
opened multiple files within the “Secret” file share folder.

![alt text](/assets/img/Recent.png)
![alt text](/assets/img/Secret Folder.png)

During data exfiltration, attackers commonly create a staging ZIP archive to consolidate stolen files and streamline the 
exfiltration process. To investigate this, I have reviewed the USN Journal for newly created ZIP files and identified 
one named "secret.zip"

![alt text](/assets/img/Zip.png)