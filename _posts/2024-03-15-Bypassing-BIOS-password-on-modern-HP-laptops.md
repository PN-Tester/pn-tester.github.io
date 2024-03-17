---
layout: post
title: Bypassing BIOS password on modern HP laptops
date: '2024-03-15 12:00:00 -0600'
description: 'Journey to unlocking UEFI settings for physical pentest engagement'
category:
- hacking
- pentest
tags: [BIOS,UEFI,Pentest,Physical,Bypass,BIOSPASSWORD]
mermaid: true
image:
  path : "/assets/img/bios/title.jpg"
  src : "/assets/img/bios/title.jpg"
---
> TL;DR - Use my program to patch your BIOS and enter MPM -> [LINK](https://github.com/PN-Tester/pn-patcher/)

There are many reasons to want to bypass an unknown administrator password blocking you from accessing BIOS settings on a computer. On the casual side of the coin, we have people in possession of second-hand computers that want to customize some feature and find that they cannot. On the more professional side, an IT consultant may need the ability to disable secureboot and change the boot order of a laptop in order to run an alternate OS from a USB. The above are some of the more common, vanilla reasons. On the spicier side of things, we have individuals with stolen laptops attempting to wipe serial numbers from the firmware and disable MDM technologies used to by companies to track assets, like CompuTrace. Then there are those of us whose alignment is more, shall we say, chaotic neutral. Unlocking BIOS has big and often overlooked benefits from an offensive security perspective, and can be useful in expanding available attack surface in certain situations. In my particular case, I was tasked with an engagement that involved compromising a simulated "stolen laptop". This is a less frequent form of threat emulation so i was excited to be involved. I received a well-hardened machine, was given no windows credentials and the laptop had bitlocker configured to use the TPM. With a scope restricted only to the actual device itself, I didnt have whole lot of options. Full-disk encryption in this case prevented me from manipulating the filesystem to gain access to the OS via traditional means, so one of my only remaining option was to attempt Direct Memory Access (DMA) attacks. Modern DMA attacks typically leverage the fact that the PCI bus allows communication between arbitrary peripheral devices and important system components, like memory. Typically, exploiting this requires use of PCI express ports and a device (called a screamer) that can send specially crafted malicious PCI "Transmission Layer Packets" to read or modify RAM on the target computer. Typically, you'll find PCIe ports on workstations and servers, but rarely newer laptops. I had a screamer board I wanted to use for this on hand, so I opened up the laptop casing and took note of the motherboard. As expected, there were no PCIe ports, but luckily this was a modern laptop (HP Elitebook X360 G10 2-in-1) so it had a wifi module connected to the motherboard via an M.2 E port. Sweet! I know from my research that M.2 E ports typically expose at least 1 PCI express lane, so I unscrewed that sucker and plopped in a M.2 E-key to M.2 H-key adapter. I then used another adapter, M.2 H-key to PCIe 1x in order to connect my screamer! After a bit of painfull experimentation and research which lead me to understand the M.2 does not supply the needed 12v power to the board as a traditional PCIe port would, I fixed everything by using an external PSU which exposed a 15 pin SATA power cable and connected that to my M.2 H-key board via a 4-pin adapter. Bingo. Everything is powered on, and I can reach the screamer from my attack laptop. I enthusiastically attempted to read memory from the target laptop and........nothing. Repeatedly. The screamer was failing 100% of page read operations and getting kicked off the PCI bus every attempt, forcing me to reboot constantly. What gives? Well well, i'll tell you what gives. Its not 2012 anymore folks! Turns out Microsoft, Apple, Intel, AMD and co take these hardware based attacks pretty darn seriously (with good reason) and have gotten well into implementing countermeasures against DMA attacks by default on modern computers. Since my target laptop was using Intel based processor, the countermeasures that concerned me where in-fact known as "DMA Protection" and of course, "Intel VT-D" a.k.a "Virtualization Technology for Directed I/O". Damn, ok. Good for them, bad for me. How do I get around this? I would need to **disable these settings in order to enable DMA attacks**. And guess where these settings are configured? That's right, in the BIOS. 
Now when I say BIOS, I really mean UEFI as has been the standard for some time now, but we all know what i mean so lets move on because the terminology remains relatively interchangle both in documentation and in the actual settings themselves to this day. So, I need to disable these securities features in the BIOS. No problem. Should be easy enough right? I reboot the computer, smash F10 at the preboot screen and....
![BIOS PASSWORD](/assets/img/bios/1.png){: width="700" height="400" }

> Checkmate hacker !

Now this is a problem for me because I have no way to perform DMA without altering these BIOS settings. This is how my journey began. I had invested quite a bit of time into the DMA attack preparation at this point and was NOT going to take this lying down. No sir. This blog post will detail the steps I took to understanding and eventually overcoming this security feature. 

## BIOS RECON

So first of all, I needed to refresh myself on how exactly BIOS works on modern laptops. I know it's used for preboot and to initialize the OS, I know about the POST process and all that stuff from my introductory computer science classes but I have no real physical conception of how this is implemented on a modern motherboard. So I open up my trusty search engine and start learning.

BIOS tends to written to a dedicated chip which is soldered to the motherboard. This chip is a type of EEPROM, or Electronically Erasable Programmable Read-Only Memory. These chips come in different flavours, sizes, models, and even manufacturers can differ between different versions of the same computer from the same company. They typically look like this :
![EEPROM](/assets/img/bios/2.PNG){: .shadow }
Whats more, BIOS involves two of them these days. One of these chips I will refer to as the "Main BIOS" SPI chip, and the other one is known as the "Embedded Controller" or EC chip. They both contain BIOS firmware data, and both have a role to play in potential bypasses on different systems. The purpose of having two separate chips is to ensure integrity between important BIOS configurations and to have the potential to restore from the EC chip if the main BIOS is corrupted or altered. More on this later. For now, know that there are two different chips involved and they contain different yet related data. In my case, I identified those chips fairly quickly as Winbond W25Q256JVEN chips as shown below :
![Motherboard](/assets/img/bios/3.png){: .shadow }
From the chips, we can tell a couple of things. These chips have 8 pins and hold  256 megabits / 32 MB of data each, and they are soldered to the board in different ways. The EC chip is connected directly to the board, with no pads, in what is known as WSON8 configuration.
![WSON8](/assets/img/bios/4.png){: .shadow }
The Main BIOS chip on the other hand, has feet, and is thus in SOIC8 configuration. 
![SOIC8](/assets/img/bios/5.png){: .shadow }
More on this later.

Now its all well and good to locate the chips, but what the heck do we do now? The answer is roll-up-your sleeves and get ready to do some hacky shit.

## Extracting BIOS from EEPROM

In order to proceed with any kind of modification to the BIOS state to circumvent the password, we will unavoidably have to take a hardware based approach. What this means is that we need a way to read and write data to these chips. I started this journey with no idea how to do that, so bear in mind, its feasible but painful. Lets press onwards!

In order to work with these we need a device known as a Universal Programmer. It looks like this and connect via USB to your attack computer.
![Universal Programmer](/assets/img/bios/6.jpg){: .shadow }

I bought a a T48 universal programmer off amazon which arrived the same day. There are other models you can use too, I picked this one because it had good reviews and a good price. There are a number of third party softwares you can use to run this thing, I used the supported software from the manufacturer, which frankly looks like its been trojanized to hell and back but in fact works well and is quite stable, called XGPro. After updating my programmer firmware, it was recognized by the software and I was ready to start programming some chips! right? not yet no.
At this point, we encounter a fork in the road. See these programmers use "slots" to hook up the chip they want to interact with. Those slots are known as ZIF pins. There are 40 of them on the T48 and I can tell you, there is no possible way those pins can fit directly with the chips mentionned above. What is needed is an adapter, or rather, a series of adapters. First, we need something which respects the ZIF pin spacing and allows us to connect to an 8 pin chip. This is called an SOP8 adapter and you can get one very cheaply if it doesn't come with your programmer from the get-go (mine had a boatload of adapters included). And now we arrive at the fork. These adapters are designed to clamp around a _chip_. But that implies removing the chip from the motherboard! ouf. Now many of you may be familiar with soldering small components, and many of you may not be. I myself am only moderately capable at soldering, and this task was a bit outside my comfort zone. In a perfect world, one can use a heat-gun to remove the chip from the motherboard, pop it into the adapter, and go to town on it. But in my case even if I had been OK with making this potentially damaging decision in the name of science, I could not. Since this was a work engagement, and I had to respect certain scoping parameters determined beforehand, I was in fact not allowed to desolder anything from the motherboard. Luckily, there are alternative, albeit less ideal, solutions. Specifically, probes and clips! For the purpose of interacting with WSON8 chips you can acquire a 8 pin probe that fits into an adapter on your programmer. Now you can read data from the chip by holding the probe _very precisely_ to the pin sized contacts. For SOIC8, you can use clips like the one I had in my possesion shown below :
![SOIC8 clip](/assets/img/bios/7.jpg){: .shadow }
Great, but again, not so much. The clip doesnt fit whatsoever on the SOIC8 chip, and in my case the probe was 3 weeks out shipping from China. Given I had a very short timeframe for the pentest engagement, this wouldnt cut it. You'll notice a trend in my writing that hacking is very much a constant battle with sporadic obstacles that seem to pop out of nowhere. This makes it easy to get discouraged as you make a few steps progress into a problem only to discover 5 more problems as a sort of secret hacker sub menu. I however am experienced with this phenomena and was undeterred. Stuborness is key in this field. So is creative problem solving, and so, I busted out my grade school knowledge of how to make a round peg fit in a square hole, as they say. Behold :
![SOIC clip modified](/assets/img/bios/8.png){: .shadow }
Thats right, I gracelessly disassemble the clip, removing the spring and cutting it in half down the middle. Then i separated the wiring to make it maneuvrable, and finally i carefully cut off the pastic ends to expose the pins. Now thats properly hacky. With both hands, I was able to connect the pin heads to the chip pins and had a good connection, but i wasnt out of the woods yet. So, a few words on this. It turns out what i was doing initially, that is, putting the pins directly on the exposed leg pads, is in fact not a good way to get a connection.
![Bad Connection](/assets/img/bios/9.PNG){: .shadow }
Yes it works, but its extremely inconsistent. Instead, its much better to leverage the tiny holes on the motherboard located next to the legs, which are designed for this purpose. I didnt immediately figure this out since there were only 7 of those holes around this 8 leg chip, but as it turns out, one of them isnt necessary (probably a ground pin which is satisfied elsewhere). So after a lot of trial and error, I just yolo'd the pins into those holes and got a good connection to the main BIOS chip. Nice! Now its time to read.............
![Null Data](/assets/img/bios/10.PNG){: .shadow }
Of course, this would not be easy either. See, using pins or clips instead of desoldering the chip is highly inconsistent for precisely the above reason. I was getting a good connection but reading only zeroes or FF's from the chip. Thats no good. I eventually learnt that I was reading way too fast for the quality of the connection I was using. Indeed, my programmer was set to read at a clock speed of 30Mhz by default. When I dropped it down to 8Mhz, thats when the magic happened.
![Good Read](/assets/img/bios/11.PNG){: .shadow }
Bingo, I got myself a good read (after about 200 attempts, mind you). Now this was supremely hacky, but i did look pretty cool (or nerdy, depending on your interpretation) doing it! Actual picture of me using the mcgyvered probes :
![Extracting BIOS](/assets/img/bios/12.png){: .shadow }
In that pic im using both hands to hold the probes steady, and using Windows Voice Assistant to control my computer and hit Enter to trigger the read operation xD My wife , who was home that day, will attest that saying "Enter" 200 times is in fact, quite annoying, but nevertheless, I had extracted the data from the chip! Huzzah! But this isn't good enough. We need to ensure there is no corruption in the 32mb dump we extract. And there is CONSIDERABLE corruption that occurs via this extraction method. Be warned. I repeated this operation countless more times until eventually I had 3 copies of the dump I confirmed were identical by taking the MD5 checksum of the raw file.
```DOS
C:\temp\dump>for %F in (*) do certutil -hashfile "%F" MD5

C:\temp\dump>certutil -hashfile "W25Q256JV_32@WSON8.BIN" MD5
MD5 hash of W25Q256JV_32@WSON8.BIN:
929e0dc80cdb4712c13952b6954aa276
CertUtil: -hashfile command completed successfully.

C:\temp\dump>certutil -hashfile "W25Q256JV_34@WSON8.BIN" MD5
MD5 hash of W25Q256JV_34@WSON8.BIN:
929e0dc80cdb4712c13952b6954aa276
CertUtil: -hashfile command completed successfully.

C:\temp\dump>certutil -hashfile "W25Q256JV_39@WSON8.BIN" MD5
MD5 hash of W25Q256JV_39@WSON8.BIN:
929e0dc80cdb4712c13952b6954aa276
CertUtil: -hashfile command completed successfully.
```
This ensured that this was in fact, the actual dump with no corruption. Finally, we are ready to read some BIOS.

## Analyzing the BIOS dump
So first off, I needed to understand the format of this data I had pulled off the chip. I loaded it into a program called UEFITool, which is quite practical for this sort of operation. This program can interpret UEFI/BIOS dumps and provides a structured GUI layout which makes parsing it easy for a researcher. The tool cannot however make edits to the dump. For this, you will need a hex editor like HxD, which I used heavily in my research. Particularly useful is the ability to perform data comparison between two files, to single out alterations that have been made between patched and unpatched versions of the same dump. This was the basis for most of my research into this topic. Looking briefly at the data, I could identify some information about the laptop including identifiers belonging to the client who had commissioned the engagement. There were also a LOT of variables and integrity data. Doing research on BIOS structure, I came to understand that most of the data cannot be modified since it will break the checksum or the signature of the BIOS file. The only stuff that CAN be changed is referred to as NVRAM data, or non-volatile RAM data. The NVRAM data parts that can be modified are this way by design. Indeed, it wouldnt make sense that nothing could be modified, because BIOS itself allows you to set various configurable options. If you couldnt change the BIOS content, then there would be no way to change settings. And that means additionally, that something like the BIOS administrator password, which _can be changed_ will _by design_ be located, or at least referred too, in a section that can be modified! Sweet! Indeed, after much, much, much reading through forums like badcaps and reddit, I came to understand that one method to unlock BIOS on HP laptops would be to overwrite the NVRAM data section containing a reference to the aptly named "Usercreds" data. Great, all I have to do is clear this data from my dump, and reflash the chip, then there will be no more password. Easy-Peasy, or so I thought..

## Attempt 1 : Nulling out UserCred data
As has been described in the past on other sites, a simple way to remove the password is the null out the data in the UserCred variable in the chip dump. This data is contained in a section that is crucially NOT protected by BootGuard (designated as Red and Yellow), and which can be fully modified to our liking: 
![UserCred Data](/assets/img/bios/13.png){: .shadow }
I decided to go full yolostyle and try this right off the bat. The usercreds section can be reliably identified via the hex string 
```\x55\x00\x73\x00\x65\x00\x72\x00\x43\x00\x72\x00\x65\x00\x64``` so i made a quick and dirty python script to replace the appropriate amount of characters after that pattern is identified with null bytes. The before and after modification looks like this :
![UserCred comparison](/assets/img/bios/14.PNG){: .shadow }
I then loaded that pupper into XGPro and flash it to the chip, already tasting that sweet passwordless BIOS prompt. I was brought back to earth when I hit the following error :
![EC Error](/assets/img/bios/15.png){: .shadow }
The laptop then updated itself multiple times and, when all was said and done, the password remained in the BIOS settings. I was still locked out. It seemed that the change was not being accepted for some reason, and that the BIOS was being reverted with the data on the EC chip. Damn.

## Attempt 2 : Modifying Embedded Controller data
Since the data on the chip i was attacking was seemingly being overwritten by data on the EC chip, I decided that I should in fact attempt to attack the EC chip instead. Extracting this data was a LOT harder than reading from the Main BIOS SPI chip because there are no holes for our pin probe and furthermore, WSON8 is far less forgiving than SOIC8 when it comes to touching the pins to the contacts. We are talking about needle sized pin heads here folks. Saying this is a frustrating task is an understatement. Again, I toiled for hours trying to get a good read from this sucker, and eventually got 3 consistent dumps demonstrating a non-corrupted read had occured. Finally, I was ready to jump in. I was aware that there were multiple methods to unlock bios on HP computers, and that the previous method was not always successful, especially on newer computers. I was also aware of some tools that exist which perform automatic patching of binary data. Wanting to avoid being a script kiddie, I took intiative to learn how to manually perform these actions, but nevertheless I was curious how these programs performed, and if they would succeed. So, I tried out almost every existing tool I could find. I was out of luck. It seemed that all these tools expected a 16MB EC dump, and I had a chonky 32MB EC dump, indicative of brand new machine (indeed, it was a late 2023 model laptop). No matter, I could reverse engineer the methodology and make my own. I was a hacker after all, n'est-ce-pas? I eventually understood the algorithm. The tools were all variants of the same thing. In fact, they were designed to be limited to 16MB dumps intentionally since in the past, only the Main BIOS was 32MB on HP laptops, and this feature prevented incorrect use of the softare to patch the wrong chip. So I could just bypass this limitation a number of ways, and did, but i was still getting errors. It seems the algorithm that was being employed to patch the EC dump was the problem. It was _not working_ on my dump. I realized the issue soon enough by analysing the changes the tools were supposed to be making. They were seeking out a section of memory typically located at offset 37000 in the dump, and clearing the value of a variable called NvramActiveRegn. This operation would "break" the mapping of the BIOS password to the password data itself, and the computer would think that there was in fact no password set. Brilliant! Who needs tools, I know the magic sauce. Ill just do it myself! right? Turns out, this technique is either outdated in 2024, or it does not apply to my particular laptop for some reason. Manually combing through my EC dump, I realized with horror that I did not have the required NvramActiveRegn section AT ALL. Yes there was a reference to this variable near the end of the file, but not in a manner consistent with storing password references. Furthermore, I recovered several correctly patched EC dumps from the internet, and noted that these also had the same data I was seeing, in addition to the data that needed to be cleared. 
![chirpy](/assets/img/bios/16.PNG){: .shadow }
![chirpy](/assets/img/bios/17.PNG){: .shadow }
So it looked pretty bleak here, I could not clear the BIOS password using this method either, since my EC data was newer and did not seem to contain the data that needed to be modified using the existing methodology whatsoever. Ready to give up yet? Helllllllllll no! We fight for ODIN here boy. GLORY OR NOTHING!
![chirpy](/assets/img/bios/18.png){: .shadow }

## Attempt 3 : Manufacturer Programming Mode
After analyzing dozens of dumps of different BIOS configs, I had realized that in some cases, certain patched BIOS dumps which were attempting to remove features such as Absolute Persistence, or CompuTrace, etc, where in fact modifying a ? character ```\x3F``` for a < character ```\x3C```. I had a dump which i was analyzing that had this modification occur at 5 points very specifically, all next to what appeared to be the names of features. What I would originally have assumed was haphazard, a formatting issue somewhere down the line, was starting to look like an intentional pattern. Loading that particular dump into UefiTools, I realized that this changes was causing those precise elements to become invalid.
![chirpy](/assets/img/bios/19.PNG)
That was in fact removing those features! This lead me, in reverse, to understanding how the features were working in the first place. I came to understand that variables in these dumps are followed by a ? character (in hex) which designates the end of the variable's value. By changing that to <, we were in fact breaking that value assignment and rendering it null. Crucially, this lead me to notice that the variable declaration is always terminated with the hex sequence ```\xAA\x55``` preceding the ```\x3F```. Are you thinking what im thinking? If the answer is, we could potentially control these settings if we properly terminate the value with the ```\xAA\x55\x3F``` chain then the answer is yes. This is an important pattern, and brings me to the next bypass attempt. In my research, I had noticed several times various folks referring to MPM, or Manufacturer Programming Mode. I had noted that it appeared as a viable way to get around BIOS passwords, but couldnt find any documentation or tooling which would show me what needed to be modified in order to activate it. Going back to the drawing board, I decided to dive into my original main BIOS chip data and search for references to this feature. Armed with the above knowledge on how variable are set and terminated, I grepped away at possible combinations of hex values that could possibly refer to this elusive feature and I found this bad boi:
![chirpy](/assets/img/bios/20.PNG){: .shadow }
The string ```\x48\x00\x70\x00\x4D\x00\x70\x00\x6D\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xAA\x55\x3F``` was designating H.p.M.p.m, and it was terminated with ```\xAA\x55\x3F```, indicating it was probably a variable assignment. Sweet! Its current value was null, just a bunch of \x00's, but what if I changed it? And, to what? I went through a TON of BIOS dumps to find some arbitrary HP model that I new for a fact had manufacturer mode enabled, and sure enough, when I located hex pattern in the dump, there was a difference. The value had ```\x23\x01\x00\x02``` instead of ```\x00\x00\x00\x00``` , replacing 4 null values in the middle of the variable value section !!!
![chirpy](/assets/img/bios/21.PNG){: .shadow }
Could this be the secret sauce to enable MPM ? Only one way to find out. I modified my Main BIOS dump with this string, and yeeted it into the chip. When I booted the PC, it almost jumped off the desk. Fans at 200%. She seemed to grumble and gasp, booting and rebooting 2-3-4 times and then :
![chirpy](/assets/img/bios/22.png){: .shadow }
SALVATION! That sweet sweet red labelling got me tripping HARD at 3am. I smash F10 like I was entering hyperdrive, and was greeted with the following :
![chirpy](/assets/img/bios/23.png){: .shadow }
Success. Sweet, sweet success. I had entered manufacturer programming mode, the mode that the computers are in at the factory. This is where all the basic information like serial number, device type, warranty date and such are programmed into the BIOS firmware. Via this mode, you can reset factory default settings from the Main Menu. Then, after a reboot, you can reset the security settings. Finally, you can set your OWN Administrator BIOS password, sending the previously unknown password to what I can only assume is the digital equivalent of the river styx. 

![chirpy](/assets/img/bios/24.gif){: width="700" height="400" }

Finally I was able to leverage access to the Bios Settings to disable VT-D and DMA protection, thus enabling my actual objective, performing a DMA attack.
![chirpy](/assets/img/bios/25.png){: .shadow }
That particular challenge will be subject to another blog post, as it changes subjects quite a bit and is rather interesting as a topic on its own. 

## Automating the process : PN-Patcher.py
Of course, I believe in community contributions ! As such, I have created a simple python program which automates the above process of patching the binary data from the main BIOS chip and made it publicly available on my github at the following link : 

[https://github.com/PN-Tester/pn-patcher/](https://github.com/PN-Tester/pn-patcher/)

The tool takes a UNC path as an argument pointing to your main BIOS chip dump. It will automatically seek out the H.p.M.p.m hex pattern discussed above and patch it with the correct byte sequence to enter manufacturer programming mode. A demo is shown below :

![pn-patcher](/assets/img/bios/pn-patcher.gif)

The code to do this is of course available at the github page I have linked above. There is nothing fancy going on here, I am seeking the binary data in the dump which matches the hex string we discussed earlier, and literally patching it with the hex string that contains the MPM trigger. This is resumed in the below function which is part of the program :

```python
def patch_mpm(filename):
    pattern_hex_mpm = b'\x48\x00\x70\x00\x4D\x00\x70\x00\x6D\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xAA\x55'
    pattern_hex_mpm_patch = b'\x48\x00\x70\x00\x4D\x00\x70\x00\x6D\x00\x00\x00\x00\x23\x01\x00\x02\x00\x00\x00\xAA\x55'

    try:
        with open(filename, 'r+b') as file:
            print('\t[*] ATTEMPTING TO PATCH MANUFACTURER PROGRAMMING MODE')
            with mmap.mmap(file.fileno(), 0) as mem:
                loc_mpm = mem.find(pattern_hex_mpm)
                if loc_mpm == -1:
                    print('\t[-] MPM PATTERN NOT FOUND')
                    return False
                print('\t[*] MPM PATTERN FOUND AT OFFSET:', loc_mpm)
                print('\t[*] REPLACING MPM TRIGGERS')
                mem[loc_mpm:loc_mpm + len(pattern_hex_mpm_patch)] = pattern_hex_mpm_patch
                return True
    except Exception as e:
        print(f'\t[-] An error occurred while patching MPM: {e}')
        return False
```

I have not seen any other tools that do this and I especially havent seen any documentation or anything resembling open source programs that function on very new HP laptops. There is a wealth of knowledge on the subject out there, but its very inconsistently documented, and a lot of it is outdated. I do recommend anybody interested spend some time looking at the guides available on the badcaps forum as these are a great starting point for understanding this process on various computers.

Anyway, I hope this helps some people who may be seeking to remove BIOS lock from a computer for some reason or another. For professional pentesters, this is an important step to disabling IOMMU, VT-D, and other DMA countermeasures for physical attacks against a laptop or workstation. For individual computer users, it can be important information in gaining full control of your own possessions, sometimes the difference between having a useable computer or a paperweight. HP themselves states that you CANNOT bypass BIOS passwords and that you in fact need to buy from them a completely new motherboard. Usually this is nearly the same cost as the computer itself. LOL. Forget that. As always, where there is a will, there's a way, provided you are willing to roll up your sleeves, brutalize your ego, and try something hundreds of times until you succeed. This is the way of the hacker friends. 
