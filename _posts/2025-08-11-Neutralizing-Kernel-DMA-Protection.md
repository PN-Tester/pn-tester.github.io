---
layout: post
title: Neutralizing Kernel DMA Protection
date: '2025-08-11 10:00:00 -0600'
description: 'Research on pre-boot DMA attack methodology for disabling Kernel DMA protection on modern windows'
category:
- DMA
tags: [DMA, Pentest, Physical, Hardware]
mermaid: true
image:
  path : "assets/img/DMAReaper/DMAReaper.jpg)"
  src : "/assets/img/DMAReaper/DMAReaper.jpg)"
---

The FirstStrike approach is to leave "DMA Protection" and "Intel VT-D" enabled in BIOS, opting only to disabled "Intel VT-X" or "Virtualization Technology" when these settings can be controlled.

The attack then injects a modified pcileech kernel module during preboot while detonates automatically from within ntoskernel.exe once the OS is loaded. This approach works despite OS level Kernel DMA protection being enabled. 

### Caveat
In some UEFI implementations however, Intel VT-X cannot be disabled while VT-D is enabled (the option is greyed out). 

Furthermore, the FirstStrike shellcode only works at this time against windows 10 targets. There is no public solution against Windows 11.

We need an alternative method to disable Kernel DMA protection for these circumstances.

### Initial Setup
Before we continue, lets check our windows target and make sure that Kernel DMA Protection is indeed enabled.

The documentation from microsoft states that we can check this in two places , MSINFO and "Device Security".


![https://learn.microsoft.com/en-us/windows/security/hardware-security/kernel-dma-protection-for-thunderbolt](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/1.PNG)

Ok, lets confirm those elements on our target laptop :

### Device Security

![Verifying Device Security](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/2.PNG)

Looks good

### MSINFO32

![Verifying MSINFO](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/3.PNG)

Alright so Kernel DMA Protection is enabled at the OS Level. Perfect !

## Theory
Kernel DMA Protection has several *requirements* in order to function properly. We can perhaps attack some of these requirements in order to disable it.

From Microsoft Documentation on Kernel DMA Protection :

![How Kernel DMA Protection Works](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/4.PNG)

IOMMU is used. This makes sense, since we know IOMMU contains maps of physical to virtual memory each peripheral device is allowed to access.

So obvious requirements seem to be VT-x and VT-d enabled in firmware.
This is evident from Microsoft documented instruction for enabling Kernel DMA Protection seen below : 

![Kernel DMA Protection requirements](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/5.PNG)

Alright, we already have a method of attacking the OS when we can disable VT-x (FirstStrike), so lets focus on VT-d. Crucially, in some UEFI implementation, VT-x cannot be disabled without turning off VT-d first, and turning off VT-d triggers BitLocker recovery. That wont work. We need to *leave the features on* and still manage to disable DMA protection.

Let's explore how VT-d works from the Intel Virtualization Technology for Directed I/O documentation :

![Intel VT-d Documentation](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/6.PNG)

So, the **IOMMU** works because VT-d reports the remapping data to the OS through this DMA Remapping Reporting (DMAR) ACPI table structure.

What is ACPI ? Advanced Configuration and Power Interface. Essentially tables used by Firmware to report capabilities and communicate functional data to the OS (think fans, backlights, etc.)

These are just "Structures" in memory. 

Ulf Frisk made a blog post in 2016 about how nulling the DMAR ACPI can disabled Virtualization Based Security.

While this is no longer the case in 2025, it may still impact Kernel DMA Protection, which is separate from VBS. We will set the DMAR ACPI table as our target. Ulf's blog post assumes that attackers already know the address of the DMAR ACPI (by booting into Linux from a USB and using dmesg to grab logs from the kernel at boot). This is an unreasonable requirement since most target computers will not allow us to control boot order or boot from USB. We will need to find the address ourselves.

On windows, there is no easy way to read this from the OS. There are tools we can use to *parse* ACPI tables, but we **cannot dereference the virtual address** exposed to the operating system. And if we want to interact with this structure, we will need the physical address.

In UEFI, virtual and physical memory are mapped 1:1, so the best approach to identifying the location of the DMAR ACPI table is to do so from pre-boot environment. While we cannot do this during a pentest, we CAN do it for our R&D in order to create the attack primitive, and work backwards with that information to find a valid search technique. Lets get started.

#### Installing UEFIShell

We can place a custom EFI program called UEFIShell into the boot partition of the windows system as shown below :

![Deploying UEFIShell](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/7.PNG)

Once this EFI program is in place, we can reboot and boot into the shell from UEFI menu. First we open the Boot Menu (F9)

![Selecting Boot Menu](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/8.PNG)

We select boot from file

![Selecting Boot From File](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/9.PNG)

Select the right volume

![Selecting Volume](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/10.PNG)

And now we can navigate to S:/EFI/Microsoft/Boot where we placed our Shellx64.efi program earlier

![Selecting Shellx64](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/11.PNG)

Running this, we get our UEFI Shell. This is great because it runs as a program within the current UEFI environment, meaning data structures in memory are representative of the normal pre-boot environment. We can use the built-in acpiview command to read the DMAR table and get its physical address for our R&D :

![UEFIShell - acpiview](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/12.PNG)

Sweet, now we know what we are looking for, the DMAR ACPI is located at **0x57B64000**

### Finding the DMAR ACPI programmatically

So we can't exactly use acpiview from a DMA perspective, and booting UEFIShell only works in our lab environment because we intentionally disabled Secure Boot and have access to the Boot Menu in the first place. This wont work during a pentest. We need a way to derive the physical address of the DMAR ACPI table programmatically using only our pre-boot DMA capabilities.

#### Leechcore 

The PCILeech Framework includes a python library which acts as a wrapper around important basic functionality used to control the PCILeech firmware. We will leverage this to gain arbitrary READ/WRITE from python so that we can implement our own logic without the overhead of modifying the C language components of the PCILeech client or modules. 

![LeechCore](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/13.PNG)


We implement this library into a basic python program which will initialize the connection to our screamer board, and use the read(), read_scatter(), and write() functions to control the FPGA during pre-boot and obtain and modify important memory structures.

#### Starting from the EFI System Table

The EFI System table is the primary target for UEFI exploitation because it contains pointers to all the important functions and structures that an EFI program may require to run during pre-boot. This includes pointers to functions like BootServices and RuntimeServices, but also pointers to data structure as we will see.

Crucially, we can locate the EFI System Table by reading memory during pre-boot and flagging the IBI SYST hex pattern (`4942492953545359`). We will implement some basic logic in python to scan memory for this, and to eliminate false positives by checking for stuff like revision number and size.

The search for the table and resultant structure is seen here :

![Search for EFI Base](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/14.PNG)

Now that we can reliably find the EFI System Table, we can get to its content!

#### Configuration Table

The UEFI 2.1 Standard describes the Configuration Table below :

![UEFI Configuration Table documentation](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/15.PNG)

The Table will contain GUID/Pointer pairs that we are interested in.
Crucially, the offsets of this structure from the EFI System Table Base address is **always the same**

So, we know that at `0x68` from the EFI Table base address, we will have the "Table Entries" value describing how many GUID/Pointer pairs exist, and

at `0x70` from the EFI Table base address, we will have a pointer to the Configuration Table data.

We see that in our dump of the System table below, at exactly offset `0x70` as expected

![Configuration Table Pointer](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/16.PNG)

The Configuration Table Pointer in little Endian is **0x521B8000**

We can walk this table in python easily. The raw memory looks like this :

![Configuration table data](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/17.PNG)

Again, according to the UEFI 2.1 Specs, this section contains GUID/Pointer pairs. The GUIDs are 16 Bytes long and the Pointers are 8 Bytes long making each entry 24 Bytes total. We can walk these programmatically in 24 byte chunks to parse them.

Once we find the right GUID, we will use its accompanying pointer to get to that entries VendorTable data structure.

From the documentation, common GUIDs are described

![UEFI Industry Standard GUIDs](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/18.PNG)

We want to find ACPI tables, so we are interested in the **EFI_ACPI_20_TABLE_GUID** value!

The doc shows the value as 
  `{0x8868e871,0xe4f1,0x11d3,\`
    `{0xbc,0x22,0x00,0x80,0xc7,0x3c,0x88,0x81}}`

In reality, the structure uses a mix of Little and Big Endian for its different parts. If we want to pattern match, we need to normalize the different endianness of this string.

The three parts before the slash are little endian and everything after the slash is big endian, leading to:

EFI_ACPI_20_TABLE_GUID = **71 E8 68 88 F1 E4 D3 11 BC 22 00 80 C7 3C 88 81**

We locate the accompanying pointer in the Configuration table dump as shown below :

![Vendor Table Pointer](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/19.PNG)

#### ACPI 2.0 VendorTable

Now that we have the pointer to the ACPI VendorTable (**0x57BFE014**) we can read from that address. The objective is to find the **Root System Description Pointer** (RSDP) value, which points to the location of the **Root System Description Table** (RDST). 

We find the pointer precisely at the previously obtained address, reading 4 bytes backwards.

![RDSP](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/20.PNG)

#### Root System Description Table

With the RSDP known, we can find the RSDT. This table contains pointers to all the ACPI tables currently installed in the firmware! We will parse them one by one and read the data at the locations they point to, searching for an entry starting with the DMAR signature in hex. (`444D4152`)

Among the entries is the same address we obtained earlier using the acpiview program (the address is relatively static across reboots).

![RDST](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/21.PNG)

We can confirm by parsing the memory at 0x57B64000 manually. We see the content of the DMAR ACPI table :

![DMAR ACPI](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/22.PNG)

This is the physical address of our target !

#### DMAReaper.py

Now that we have the algorithm for finding the DMAR ACPI address, we can implement it in python leveraging the LeechCore library to control the FPGA.

The program will perform pre-boot DMA attack to identify the location of the DMAR ACPI table and overwrite it with 64 zeroes.

DMAReaper.py execution against Windows 11 24H2 is shown below. The program takes the following logical steps
1) Scan memory using scatter search in 100 x 4096 byte chunks (size of a page)
2) Find the EFI System Table Base address
3) Find the Configuration table using offset 0x70 from the EFI System Table Basse address
4) Parse each entry to find the ACPI 2.0 GUID
5) Find the paired ACPI 2.0 VendorTable pointer
6) Find the Root System Description Pointer in the ACPI 2.0 VendorTable
7) Find the Root System Description Table for ACPI 2.0
8) Read memory at each ACPI table pointer in RSDT
9) Identify the DMAR ACPI Table
10) Overwrite the DMAR ACPI Table with 64 zeroes

![DMAReaper.py](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/DMAReaper.jpg)

What happens if we boot now? 

#### Kernel DMA Protection Disabled

Windows 10 and Windows 11 will boot without complaining, but the DMAR ACPI table cannot be located since it was destroyed. This means that no IOMMU can be used, which means Kernel DMA Protection cannot be initialized!

We can verify this again by checking "Device Security" 

##### Windows 10 - Device Security

![Win 10 Device Security](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/23.PNG)

##### Windows 11 - Device Security

![Win 11 Device Security](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/24.PNG)

**Memory Access Protection** is now *missing* from the Core Isolation features!!

Lets check MSINFO32 as before :

##### Windows 10 - MSINFO32

![Win 10 - MSINFO](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/25.PNG)

##### Windows 11 - MSINFO32

![Win 11 - MSINFO](https://github.com/PN-Tester/pn-tester.github.io/blob/main/assets/img/DMAReaper/26.PNG)

System information is reporting that **Kernel DMA Protection is OFF**

Virtualization-based security capabilities no longer reports DMA Protection as available.

Yet in the UEFI firmware configuration :
- DMA Protection is still enabled
- VT-x is still enabled
- VT-d is still enabled
- Secure Boot is enabled
- BIOS Sure Start is enabled
- Virtualization based BIOS protection is enabled
- Enhanced Firmware Runtime Intrusion Detection and Prevention is enabled

We have successfully demonstrated that pre-boot DMA attacks can bypass OS level DMA Countermeasures that depend on communication with firmware components.
