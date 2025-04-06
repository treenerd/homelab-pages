---
title: "Reviving a Relic: Conquering the 2025 Cisco UCS Upgrade"
date: 2025-04-05
---

Fellow homelab adventurers, prepare for a tale of digital archaeology! I recently embarked on a mission to breathe new life into some older Cisco UCS bricks, and let me tell you, it was a journey through the labyrinthine corridors of legacy technology.

The primary hurdle?  Around 2013, Cisco decided to rely on Adobe Flash for the CIMC (IPMI) interface on UCS C220 servers.  Fast forward to 2025, and Flash is as extinct as the dodo, leaving us with a rather... interesting situation.  Trying to manage these systems without a functional CIMC is like trying to navigate a spaceship with a broken viewport.

But the Flash fiasco was just the appetizer.  The main course involved wrestling with the ancient Java version required to launch the KVM console.  Imagine trying to find a compatible Java runtime in a world dominated by modern JVMs – it was a time warp!

After a few hours of head-scratching, countless browser tweaks, and deep dives into the cryptic depths of the Cisco community forums, I finally cracked the code.  I managed to coax these venerable machines into accepting a shiny new 4.1x firmware upgrade.

It wasn't pretty, but it was victorious.  If you're facing a similar challenge, fear not!  I'm documenting the exact steps I took, the arcane rituals I performed, and the browser incantations that ultimately led to success. Stay tuned for the detailed walkthrough – you're not alone in this digital quest!

## What is Cisco CIMC?

The Cisco Integrated Management Controller (CIMC) is a dedicated management processor embedded within Cisco UCS (Unified Computing System) servers.
* **Remote Server Management:**
    * Allows administrators to remotely monitor and manage Cisco UCS C-Series rack servers.
    * Provides control even when the operating system is down.
* **Out-of-Band Management:**
    * Operates independently of the server's main OS.
    * Crucial for:
        * Powering the server on/off.
        * Monitoring server health (temperature, fan speeds, etc.).
        * Updating firmware.
        * Accessing the server's console (KVM - Keyboard, Video, Mouse).
* **Key Functions:**
    * Provides web-based interface, CLI, and XML API.
    * Supports industry-standard protocols like IPMI (Intelligent Platform Management Interface).
    * Offers low-level hardware control for data center management.

**In simpler terms:** CIMC is like a separate, always-on computer inside your server that lets you control the server's hardware remotely.

## CISCO HUU

Where to start the upgrade journey?  
Well let's check the related Release notes of the UCS Rack Server Software.  
In case of UCS C220 M4 the lates release at time of writing was [4.1(2)](https://www.cisco.com/c/en/us/td/docs/unified_computing/ucs/release/notes/b_release-notes-for-cisco-ucs-rack-server-software-release-4_1_2.html#reference_ikd_w5t_zjb) .
In case of UCS C220 M3 [3.0(4)](https://www.cisco.com/c/en/us/td/docs/unified_computing/ucs/release/notes/b_UCS_C-Series_Release_Notes_3_0_4.html)
In case of UCS C22 M3 [3.0(4r)](https://www.cisco.com/c/en/us/td/docs/unified_computing/ucs/release/notes/b_UCS_C-Series_Release_Notes_3_0_4.html)

So in case of C220 M4 with CIMC version 2.0.2, the suggested upgrade path was 3.0(3a) to 4.1(2a) .  
The upgrades worked, but to install them was not as easy as expected.

## The challenge

How is it possible to access flash based web interfaces in 2025 to upgrade the CIMC.
Short answer, I tried, failed and discovered it is not needed at all for the upgrade.

### Just use a USB flash drive

My first thoughts where. Just use an USB stick, copy the ISO, plug it in and let the server boot into huu (the cisco upgrade tool).
Well, I recognized that it was not a common thing to use usb flash drives for this task back in 2013.
It seems Admins had to burn CD's and use CDROM drives to let the server boot into huu iso images, when it was not possible to access the out of band management.

Well ... Back in time I had CDROM drives at the past, but now we have 2025..

### Do I really need to access Flash?

I do not have to access the Adobe Flash interface to access vKVM. There is a workaround, which I found in the Cisco Community Forum.
Just the JNLP file is needed to start the java web application. Our goal is to update CIMC, not to change settings.
There is a workaround:
```bash
curl -o kvm.jnlp -k https://<cimc_ip>/kvm.jnlp?cimcAddr=<cimc_ip>\&tkn1=admin\&tkn2=<password>

```
So far so good.
But now we need some compatible java version.
Well, I had CDROM drives at the past. But in 2025? Nope not working anymore.

### Do I really need to access Flash?

Nope I do not have to access the Adobe Flash interface. There is a workaround, which I found in the Cisco Community Forum.
Just the JLNP file is needed to start the java web application.

```bash
https://<cimc_ip>/kvm.jnlp?cimcAddr=<cimc_ip>&tkn1=admin&tkn2=<password>

```
So far so good.
But now we need the java. Openjdk-8 and icedtea didn't work in my case.

#### Which java to use?

Can you remember, back in time when java has been available at [java.com](java.com)?
It is still available there! [Java downloads](https://www.java.com/en/download/)

```
cd ~/Downloads
sudo mkdir -p /opt/java/64
sudo tar xzvf ./jre-8u441-linux-x64.tar.gz /opt/java/64
sudo ln -s /opt/java/64/jre1.8.0_441 /opt/java/64/jre
sudo update-alternatives --install "/usr/bin/java" "java" "/opt/java/64/jre/bin/java" 1
sudo update-alternatives --install "/usr/bin/javaws" "javaws" "/opt/java/64/jre/bin/javaws" 1
sudo update-alternatives --set java /opt/java/64/jre/bin/java
sudo update-alternatives --set javaws /opt/java/64/jre/bin/javaws

java -version
java version "1.8.0_441"
Java(TM) SE Runtime Environment (build 1.8.0_441-b07)
Java HotSpot(TM) 64-Bit Server VM (build 25.441-b07, mixed mode)
```

### CIMC KVM Java Web Start Application

Because of the outdated Certificates in CIMC, we also have to think about a solution to overcome this obstacle.

When I tried to open the kvm.jlnp I got an exception:
```
sun.security.validator.ValidatorException: PKIX path validation failed: java.security.cert.CertPathValidatorException: denyAfter constraint check failed: SHA1 used with Constraint date: 2019-01-01; params date: 2025-04-06T11:35:24.198Z used with certificate: CN=DigiCert Assured ID Code Signing CA-1, OU=www.digicert.com, O=DigiCert Inc, C=US

```
This makes sense, because SHA1 is not secure anymore and it is dangerous to use it.
But in this case it was not possible for me to find a workaround other than change the denyAfter date in the java.security settings.

`sudo vim /opt/java/64/jre/lib/security/java.security` 

```
 642 jdk.certpath.disabledAlgorithms=MD2, MD5, SHA1 jdkCA & usage TLSServer, \
 643     RSA keySize < 1024, DSA keySize < 1024, EC keySize < 224, \
 644     include jdk.disabled.namedCurves, \
 645     SHA1 usage SignedJAR & denyAfter 2019-01-01
```

Change the **denyAfter** date to today + 1 day.
**!! this is dangerous and you should be aware what you are doing with this change. Do not do this on your workstations java settings! !!**

When solved this issue, the next thing came up, because outdated certificates.
By adding the CIMC url to the exceptions.sites file on my users java configuration it could get solved.

```
echo "https://<CIMCIP>/" > ~/.java/deployment/security/exception.sites
```

If the login still fails with timeout or wrong user/password warnings, double check the CIMC password!!
After this handstand, it was possible to get into CIMC vKVM and mount the huu ISO on CIMC versions 2.x at C22 M3, C220/240 M4.

## C22 M3
I tend to disable secureboot in a homelab environment, but feel free to let it enabled it is on your opinion.
But be aware, there are threads about this topic, because difficult to disable it can be, after a certain CIMC version.
So check for secureboot in boot settings before you start with the upgrades, and decide what you would like to have.

Lets start with the upgrades:
In my case I started with CIMC version 2.0(13e)

### version 3.0(3a)
Release notes state to upgrade to 3.0(3a) from 2.0
Download ucs-c2x-huu-3.0.3a.iso and mapped it via KVM to the server.
Reboot
Press F6 and choose Cico vKVM-Mapped vDVD1.22 to boot from to start Cisco UCS Host Upgrade Utility
During the upgrade wizard it will ask you again if you want to enable Secure Boot.
Be aware of that, because it is not possible to change this afterwards.
Read the messages and then do ClickOPS.

###  version 3.0(4r)

Latest release ucs-c2x-huu-3.0.4r.iso
Before you can connect vKVM again, clear the javaws cache
```
javaws -clearcache
javaws -uninstall

```

## C220 M4


