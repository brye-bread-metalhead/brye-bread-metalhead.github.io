---
layout: default
title:  "Mikrotik SFP+ Woes"
date:   2023-12-30 10:00:00 -0600
categories: Homelab
---

Upgrading my homelab to a 10G fiber connection to the router was an interesting experience. Hopefully this information will help to troubleshoot a 10G fiber network and save you some time.

tl;dr:

- Using ifconfig -vvvv in pfsense should show that your NIC and SFP are detected
- Ensure the SFP is detected by the switch with Winbox
- ⚠️ Make sure your Mikrotik switch is on firmware 6.49.10
- Set the SFP to 10G mode and turn off auto-negotiate on the Mikrotik

In my homelab, my router is a Poweredge server with pfsense installed. It only has 2 integrated gigabit NICs. I plan to use a dual-WAN failover with my network, I need more than 2 NICs in total. My core switch, a Mikrotik CRS326, also happens to support SFP+.

What could this mean? A reason to upgrade to fiber!

In the past I had already done some research previously on which NIC to purchase. In my case it was the Intel x520-DA2. It's reasonbly power-efficient, fast enough for my network, and known to be very compatible. I recalled reading about SFP compatibility problems with this NIC, so I purchased some Intel 10G LC SFPs to tag along and avoid issues with a third party SFP.

---

The hardware I'm working with to get a 10G backbone from the core switch to my router:

- [Intel x520-DA2 Dual SFP+ NIC](https://ark.intel.com/content/www/us/en/ark/products/39776/intel-ethernet-converged-network-adapter-x520-da2.html)
- 2x Intel [E10GSFPSR](https://www.intel.com/content/dam/doc/product-brief/ethernet-sfp-optics-brief.pdf) (FLTX8571D3BCV-IT) 
  - One for the switch, one for the NIC
- [Mikrotik CRS326-24G-2S](https://mikrotik.com/product/CRS326-24G-2SplusRM) (using RouterOS)
- OM4 LC/LC MMF fiber (1M) 
  - What a mouthful! Basically a short multimode fiber cable with LC-style plugs
  - OM4 being rated for much more than 10G

I installed the NIC into my router rack and powered it up. I slotted the SFPs into both the switch and the fiber NIC. Hoping for the best, but expecting the worst, I flipped on my switch and made sure that PFSense booted up normally. Connect up the cable, then... nothing! The SFP LEDs were dead. Time to troubleshoot.

---

First thing is first, do some research to make sure someone else hasn't had the problem.

Generally speaking, I found on the Mikrotik forums that this problem (with this exact SFP) isn't uncommon. There was some discussion about SFP compatibility, both versions of firmware (6.x and 7.x) having problems, and so on. Nothing positive, other than a Mikrotik SFP is the only SFP that Mikrotik seems to test with RouterOS. A couple users mentioned that they were able to get the SFP working with a particular build of the 7.x firmware, but I am reluctant to upgrade to a new major firmware version on my core switch. 

Checking PFSense after reading forum discussion, the SFP and NIC status can be queried through the diagnostics > command prompt. There are many posts in the Reddit homelab community in regards to the X520-DA2 being perfectly compatible with PFSense, so I didn't spend a lot of time researching if there was a problem with this particular NIC and pfsense. 

---

On PFSense, SFP temperature and signal presence found through the RX and TX values show that the SFP can see a signal, and is detected by both the NIC and PFsense. The good news here is that PFSense can at least see the hardware and a signal. I didn't capture the text in the bad state, but the information displayed here is similar. If your setup is in the problem state, you would see something like 'no carrier' in teh status line. 

```
ix1: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
	description: LAN
	options=e138bb<RXCSUM,TXCSUM,VLAN_MTU,VLAN_HWTAGGING,JUMBO_MTU,VLAN_HWCSUM,WOL_UCAST,WOL_MCAST,WOL_MAGIC,VLAN_HWFILTER,RXCSUM_IPV6,TXCSUM_IPV6>
	ether xx:xx:xx:xx:xx:xx
	inet6 xxx:xxx:xxx:xxx:xxx:xxx:xxx:xxx prefixlen 64 scopeid 0x2
	inet6 xxx::x:xxx prefixlen 64 scopeid 0x2
	inet6 xxx:xxx:xxx:xxx:xxx:xxx:xxx:xxx prefixlen 64
	inet xxx.xxx.xxx.xxx netmask 0xffffff00 broadcast xxx.xxx.xxx.xxx
	media: Ethernet autoselect (10Gbase-SR <full-duplex,rxpause,txpause>)
	status: active
	nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
	plugged: SFP/SFP+/SFP28 10G Base-SR (LC)
	vendor: Intel Corp PN: FTLX8571D3BCV-IT SN: AM411QD DATE: 2012-01-29
	module temperature: 36.13 C Voltage: 3.31 Volts
	RX: 0.55 mW (-2.52 dBm) TX: 0.61 mW (-2.14 dBm)

```

Onto the Mikrotik, I looked into the SFP status with Winbox. I found similar values, where the Mikrotik could see a signal and the SFP temperature.

![Screenshot of Mikrotik SFP Winbox status](/assets/images/mikrotik-crs326-sfp-status.png)

Even though the SFP and signal are detected, there wasn't communication occurring between the two devices. I found some Mikrotik forum posts that were mentioning that a manual set of the network mode (e.g. 1G, 10G) worked around some problems. I used Winbox to configure the data rate to 1G and turned off auto-negotiate. Interestingly, on the older firmware the SFP worked! When set to the 1G mode I was able to configure the VLANs and they were pingable between the devices. This wasn't running at the desired 10G rate, but at least it was proof there wasn't a hardware failure.

It was at this point that I wasn't able to get much further. I re-read many of the Mikortik blog posts and I continued to turn up no new results or information. Browsing other information online I wasn't able to get any solid confirmations that the SFPs I had purchased were at all compatible with the Mikrotik router. Mikrotik themselves only certify their own (expensive) SFPs with their devices, and it would be disappointing to need to purchase more hardware. 

As a last ditch effort, the risks seemed lower to upgrade the switch firmware to the latest minor version in the 6.x family. Reading the [release notes](https://mikrotik.com/download/changelogs/long-term-release-tree), I didn't see any specific updates related to the SFP I was using, nor any specific changes to my switch so there weren't any proof-positive changes about my hardware configuration. It's worth upgrading anyway, for the other fixes listed. Hopefully upgrading the minor version means lower risks for things breaking.

Upgrading to the 6.4.10 was a simple matter of dropping the firmware file onto the router's filesystem root location and rebooting. A few minutes later... it's alive! 10G is working and both the router and pfsense report a 10G link. More importantly, the only outage I experienced in the lab (which also serves production my hosted services) was during the switch upgrade. No panicked late-night work this time. 

Thanks for reading, and keep on learning.

Brian

[Blog posts]: (/blog)