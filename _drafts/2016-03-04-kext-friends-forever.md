---
title: Kext Friends Forever
author: pepijn
layout: post
permalink: "/2016/03/04/kext-friends-forever/"
categories: 
  - Wifi
  - Networking
  - Opinion
  - Apple
  - Security
tags: 
  - management
  - wifi
  - networking
published: true
---



## Introduction
After this past weekend's kext-ageddon it once again became clear that Apple seems to have a problem with QA when it comes to OS X, as discussed on [this week's](http://atp.fm/episodes/159) Accidental Tech Podcast episode. While there are many areas of OS X where poor QA will likely go unnoticed by users until a later update fixes whatever QA missed, Apple has been unfortunate as of late with the specific areas of its OS X that were hit by poor QA.

There were several [App Store certificate expiration](http://www.macrumors.com/2015/11/12/mac-app-store-apps-damaged-expired-receipts-issue/) occurences that caught developers and users off-guard, the most recent one happening in mid-February as summarized [here](http://mjtsai.com/blog/2016/02/16/more-mac-app-store-certificate-problems/) by Michael Tsai. <!--more-->This latest certificate expiration has also warranted the [need to download new copies](http://tidbits.com/article/16302) of any OS X installers downloaded prior to the expiration. Users who depend on these expired OS X installer applications to deploy OS X from USB media or who are otherwise relying on the installer application will be greeted with an error that the application cannot be verified.

## Let's talk about kext, baby
All of this continued black eye-inducing lack of QA brings us to the most recent [widely publicized](http://www.macrumors.com/2016/02/27/ethernet-not-working-imac-macbook-pro/) [trouble](http://www.businessinsider.com/apple-confirms-that-it-accidentally-broke-ethernet-ports-on-some-mac-computers-with-a-software-update-2016-2) surrounding Apple's own Ethernet drivers. The affected  update was named __Incompatible Kernel Extension Configuration Data 3.28.1__, released on Friday February 26th.

### How? Also, what?
While the how, the what and the fixes have been [written up elsewhere](https://derflounder.wordpress.com/2016/02/28/apple-security-update-blocks-apple-ethernet-drivers-on-el-capitan/) I will quickly run through the actual mechanism that made it possible for Apple to carry out a rather efficient DDoS attack on its own OS X platform by straight up disabling network interfaces instead of going at it the hard way by flooding them with traffic.

As part of the OS X Software Update mechanism Apple distributes certain updates that it deems to be of a higher priority, such as Gatekeeper and XProtect updates both of which deal with malware and system security. These items are given their own identifying type in the Software Update realm, named `config-data`. These items are represented in the App Store preference pane by a separate checkbox labeled _"Install system data files and security updates"_.

![AppStorePrefs.jpg]({{site.baseurl}}/static/AppStorePrefs.jpg)

The possible impact of disabling system data files and security updates by Mac Admins who manage their own Apple updates through other means has previously been covered by Tim Sutton in [this post](https://macops.ca/os-x-admins-your-clients-are-not-getting-background-security-updates). Most home users will not make changes to the schedule and thus will automatically receive these `config-data` updates. Clearly this is the desired state in order to guarantee users have baseline protection against emerging malware.

Sadly it is also the existence of this silent update method that enabled this latest Apple SNAFU to be so destructive.

### Kext? What?
Kernel extensions (henceforth kexts) are what Apple calls self-contained bits of code that extend the OS to allow non-critical or third party hardware and software to do what they do. Because they extend the kernel, the lowest level at which any program can run, they have potential far-reaching capabilities. Apple ships a fairly long list of its own kexts with the OS located in `/System/Library/Extensions` while third parties are supposed to place their kexts in `/Library/Extensions`. With the advent of SIP `/Library/Extensions` is the _ONLY_ valid location for third party kexts. In addition Apple now enforces the signing of all kexts with a valid Apple developer account in order to be authorized by OS X. Apple does not enable kext-signing abilities for every developer, they must request this separately. The relevance of this caveat will become clear a bit later.

[As we know](https://derflounder.wordpress.com/2015/10/01/system-integrity-protection-adding-another-layer-to-apples-security-model/) there is still an exception that can be made for this signing requirement by running `csrutil enable --without kext` from the Recovery partition but I would expect Apple to put the kibosh on that before long entirely. As it stands there are [plenty of issues](https://objective-see.com/blog.html#blogEntry9) surrounding kexts to worry about, even when they are signed.

While Apple can revoke any kext signing certificates in order to block identifiable "bad" kexts they decided to add a second layer of protection against terrible, no good, very bad kexts by adding the __AppleKextExcludeList__ kext to the OS, which lives at `/System/Library/Extensions/AppleKextExcludeList.kext`. This self-proclaimed kext doesn't actually contain any executables but instead contains two files that contain enumerations of "bad" kext identifiers and their versions, and known kernel panic-inducing ones: `Info.plist` and `KnownPanics.plist`. During system boot `kextd` loads the `AppleKextExcludeList` kext and parses the two files within. If a match is found either by kext ID or by a more complicated regex query the kext in question is excluded and is not loaded.

At this point it should be somewhat obvious what happened. Perhaps in preparation for an [upcoming OS X 10.11.4 update](http://www.macrumors.com/2016/03/01/apple-seeds-os-x-10-11-4-beta-5/) Apple engineers added the identifiers for the majority of Apple Ethernet devices to the exclusion list. To be exact, the `AppleBCM5701Ethernet` and `AppleYukon2` Ethernet kexts were added, which contain a wide selection of Ethernet chipset families including the one found in Apple's Thunderbolt Display. No wireless chipsets were affected. Unfortunately this change, whatever its purpose was, did not get removed before release and went out as part of the __Incompatible Kernel Extension Configuration Data 3.28.1__ update. And as we learned earlier, this update was tagged as `config-data` or "silent and automatic" which means that it was picked up by Macs across the globe and installed without user interaction.

### Again, what?
Now you might say, _"Who cares? Who uses Ethernet to connect to a network anyway? That's so early 2000's!"_ That may be the case for those who are lucky enough to be part of an all-wireless work environment where Wifi and coffee are the only requirements for productivity. Yet, for large environments wired desktops are still very much a reality. Examples include the Enterprise where users are on wired desktop Macs due to high speed network requirements for creative production. In Education, large shared computing sites and class labs will tend to use wired networking in order to reduce unnecessary load on available Wifi networks. And besides actual need, entirely disabling major components of a customer's computer is probably not the kind of publicity Apple needs right now.

### Apple (begrudgingly) to the rescue
To their credit, Apple quickly identified the problem and posted a detailed KB article on the issue and how to recover from it, [found here](https://support.apple.com/HT6672). The easiest to recover were those Macs with working Wifi connections as they could download the fixed __Incompatible Kernel Extension Configuration Data 3.28.2__ update which removed the Apple Ethernet kexts from the blacklist, after which all was well. At worst users needed to boot into Recovery mode to reinstall their current OS. This does not incur user data loss but is certainly a hassle and can take up to an hour before being able to work again.

## What for, ye Gods?
As the full breadth of the failure was unfolding I and a few others on the [Macadmins Slack](http://macadmins.org/) were wondering to what end all of this was being done. Most of us were unaware of this kext blacklist or how it could be updated out of sight of the user, if left unmanaged.

### Ch-ch-changes
So if the Apple Ethernet kexts were not the intended changes, what was? In [diffing](http://bit.ly/1Tfk8M3) the contents of the two files it became clear that only one file really changed: `Info.plist`. Besides expected standard contents for an `Info.plist` file it contains two lists specific to the `AppleKextExcludeList` kext's purpose. The first list is named `OSKextExcludeList` which contains kext identifiers and their identifier, which is where the Apple Ethernet kexts also ended up. The other list is named `OSKextSigExceptionHashList` and serves the opposite purposes as a whitelist of explicitly allowed kexts. As part of comparing the differences I noticed two other major changes:

A kext named `com.spyresoft.dockmod.driver` of version `1` was added to `OSKextExcludeList`:

```
<key>com.spyresoft.dockmod.driver</key>
<string>1</string>
```

A sizeable selection of Hackintosh-related kexts had been removed from the `OSKextSigExceptionHashList`. Since this update applied to OS X 10.11 only, this may simply be due to SIP and its enforcement of kext signing, which kexts needed for Hackintosh needs will never be able to obtain.

```
<string>org.netkas.driver.FakeSMC	1307</string>
<key>0040423890f8ea380c181a813efa6ae0bf102f45</key>
```

### Mod my what?
Since only one kext was actually added, let's try to find out what this thing does, shall we? A quick search for Spyresoft Dockmod leads us to a shiny product page: [https://www.spyresoft.com/dockmod/](https://www.spyresoft.com/dockmod/) 

On it, we find out that Dockmod offers users a nice GUI to modify their Dock layout. Nomen est omen then, it does what it says. Next we take a look at why this fine-looking app might be banned so harshly by Apple. It certainly looks benign. Searching the product page for "kernel extension" gets us a result from the FAQ section titled _"Is it safe? How does it actually modify the dock?"_

> Dockmod 4 for El Capitan does not modify any system files. Apple introduced a new security policy on OS X El Capitan (System Integrity Protection, aka Rootless) that prevents modification of system files, even by privileged processes. This policy also prevents code injection into system processes. No previous injection method used could be modified to work with this new policy. To get around these measures and still achieve code injection, Dockmod 4 utilizes a signed kernel extension (KEXT) to handle the injection. Overall, this is the safest, most stable Dockmod yet, as no files are modified and everything is done dynamically at runtime.

Interesting. Code injection. A quick read about how older versions work reveals more talk of code injection, so this is clearly a path the developer(s) have chosen. Something that stands out in the description of how the app works in El Capitan is the fact that a kext is now _required_ in order to work around SIP limitations. This seems to be at odds with how Apple might see this issue.

Since I knew Apple has a page for registered developers to request the ability to sign kexts to be added to their account I figured I would go see what the official word on this kind of thing is.

Apple maintains a page (Developer login required) at [https://developer.apple.com/contact/kext/](https://developer.apple.com/contact/kext/) where developers can request kext signing abilities for their account. Apple presents three intended use cases: Personal Use, Testing and Consumer or Enterprise Distribution. Of those three options only one states that the developer actually needs to sign their kext. For Personal Use Apple recommends disabling the kext signing requirement with aforementioned `csrutil enable --without kext` while for Testing they recommend Kext Developer Mode. For Consumer or Enterprise Distribution the developer can submit an actual request with the product's Organization, Company or Product URL and a Reason for request description. At the bottom of this form the following disclaimer caught my eye:

> A kext signing certificate will not be issued for products that bypass OS X security features such as System Integrity Protection. For more information please see the System Integrity Protection Guide.