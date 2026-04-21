# Apex Logistics Recruitment Lure  
## Part 2 – Dynamic Analysis & Payload Behaviour

---

## Executive Summary

This part of the investigation focuses on what actually happens when 'Position Details and Compensation Policy For Emp.EXE' is executed in a controlled live enviroment.

In the first write-up, I looked at how the file was delivered and what it looked like statically. Going into this, I thought I was dealing with something fairly simple, probably centred around one of the DLLs.

After running it in a lab, it turned out to be a lot more layered than that.

Instead of one obvious payload, this is a **multi-stage setup** using:

- disguised files  
- a batch script to control execution  
- a password-protected archive  
- and a bundled Python environment running under a fake system process name  

I didn’t see clear command-and-control traffic during testing, but based on how it’s put together, this doesn’t look like something that just runs once and stops. It looks more like it’s setting the system up so something else can happen afterwards.

---

## Introduction

In my initial investigation into the Apex Logistics recruitment lure, I focused on the delivery chain and file structure without executing the payload.

At that point, my thinking was that the main activity would probably come from one of the DLLs in the package, maybe through sideloading.

That wasn’t completely wrong, but it didn’t explain everything.

To get a clearer picture, I moved into a lab and looked at what actually happens when the file is run using:

- Process Explorer  
- Process Monitor  
- Burp Suite  
- A Windows 10 VM ("CannonFodder")  

---

## Initial Execution – First Impressions

When the file is executed, a document opens straight away:


![Decoy document opened on execution](Images/14_notepad_after_exe.png)


The aim of this document is to convince the user they have opened a legitimate document and distract them from what is going on in the background...

While the process was running I noticed `zhen.mkv`, a file I had seen earlier but, at that point, I had assumed was just a decoy video file based on the extension.  
However, it turned out to be a RAR archive, and once executed it began triggering the loading of multiple DLLs in the background.


![Files Executing in Background](Images/zhen_mkv.png)


The process ended with a file named MpEng.exe, which at first glance looked like Microsoft Defender, but the company name of Python Software Foundation made it clear that wasn't the case. 


![Initial Process Explorer view after execution](Images/16_procexp_after_exe_2026-04-20_00-20.png)

It became clear this process wasn’t actually Defender at all, but a Python-based runtime executing scripts in the background.

---

## Going Back to the Files

I went back to the extracted files and noticed something I’d missed earlier. The files I thought were just decoys in my earlier investigation were hidden in a `_` folder:


![Archive structure showing hidden underscore folder](Images/29_payload_subfolder_structure.png)

![Files inside hidden underscore folder](Images/30_payload_hidden_components_in_underscore_folder.png)

I’d already seen that `zhen.mkv` wasn’t what it appeared to be, so finding these files grouped together made it clear they weren’t random decoys, they were part of the actual execution chain.

---

## File Types

Checking the real file types changed everything:

![File type identification showing disguised components](Images/31_file_type_identification_disguised_components.png)

- `.mkv` → executable  
- `.pdf` → archive  
- `Deju` → batch script  

This is where it became clearer what was happening.

---

## zhen.mkv 

![PE structure of zhen.mkv](Images/32_zhen_mkv_pe_structure.png)

This is actually a renamed **WinRAR command line tool**.

So not the payload itself, but something used to unpack it.

---

## TAIWAN.pdf 

![TAIWAN archive prompting for password](Images/34_taiwan_contents.png)

Despite the name, this isn’t a PDF. It’s a password-protected archive.

---

## Deju 

![Deju batch script content](Images/33_sed_Deju.png)

This file is the key component in delivering the payload.

It:

- executes the WordPad document
- executes zhen.mkv
- extracts `TAIWAN.pdf`  
- includes the password  
- writes files into a user directory  
- sets up persistence  
- runs the next stage

This ties everything together, Deju isn’t just another file in the archive, it’s the component orchestrating the entire execution flow.

---

I used the password to unpack Taiwan.pdf
It contains a large number of files rather than one obvious payload.

![1876 files](Images/1876_files.png)

Those files were:

- a full Python environment  
- standard libraries  
- compiled modules  
- and, **MpEng.exe**

![TAIWAN contents](Images/Taiwan_contents.png) 

---

At this point, `MpEng.exe` appears to be the actual payload, masquerading as Windows Defender while running a Python-based environment.

![Fake Defender Python runtime in Process Explorer](Images/38_fake_defender_python_runtime.png)

---

## Procmon Findings

![Procmon showing WindowsApps activity](Images/17_proc_mon_after_exe.png)

Activity is happening in:

`C:\Users\<user>\AppData\Local\Microsoft\WindowsApps`

Lots of file access attempts and `NAME NOT FOUND`.

This behaviour suggests the payload is attempting to locate or prepare its execution environment before fully committing to the next stage.

---

## Persistence 

After seeing that Deju had created a "WindowsUpdate" scheduled task, I checked Task Scheduler Library and confirmed that the created task is designed to run 'WindowsUpdate.bat' every 10 minutes indefinitely.

**The system is now persistently compromised at user level**

![Persistence](Images/persistence.png)

---

## Network Activity

No clearly malicious outbound traffic was observed during execution process.

![Burp showing no clearly malicious outbound traffic](Images/37_no_significant_traffic.png)

I monitored traffic during the scheduled 'WindowsUpdate.bat' tasks and again, none was observed.
This suggests that the payload is only operating locally at this point or is using network mechanisms not captured by the proxy.

---

## Execution Flow

```mermaid
flowchart TD
    subgraph S1[Initial execution]
        A[User executes lure EXE]
        B[Decoy document opens]
        C[Deju batch script runs]
        A --> B
        A --> C
    end

    subgraph S2[Staging]
        D[zhen.mkv used as WinRAR CLI]
        E[TAIWAN.pdf unlocked using embedded password]
        F[Payload files written to WindowsApps path]
        C --> D --> E --> F
    end

    subgraph S3[Persistence]
        G[Scheduled task created for persistence]
        F --> G
    end

    subgraph S4[Execution]
        H[conhost.exe launches next stage]
        I[MpEng.exe process starts]
        J[MpEng.exe is a disguised Python runtime]
        K[Python modules loaded]
        L[Further payload logic available]
        G --> H --> I --> J --> K --> L
    end

    M[System left with persistent execution capability]
    L --> M
```

---

## What This Likely Is

This appears to be a **multi-stage loader/backdoor setup**, rather than a standalone payload.

The structure suggests the goal is to establish persistence and prepare the system for further activity, rather than carry out an immediate, visible action.

---

## Level of Impact

If this ran on a real system:

- attacker likely keeps access  
- can run more code later  
- possible data access or further compromise  

It’s quiet, not destructive, but that’s the point.

---

## Final Thoughts

This turned out to be quite sophisticated.

I initially thought it was something relatively straightforward, but it turned out to be much more layered once I actually ran it.

Based on the behaviour observed, this payload appears to function as a staged loader rather than a standalone attack.

The use of disguised files, controlled extraction, and scheduled task persistence suggests it is designed to maintain access and enable further execution over time.

While no clear command-and-control traffic was observed during the analysis window, this may be due to delayed execution, environmental awareness, or network behaviour not captured by the proxy.

Even without immediate visible impact, this represents a meaningful compromise of the system.


---

## Original Investigation:

<https://github.com/Rayza-Slyce/Apex_Logistics_Recruitment_Lure_Investigation>
