---
published: false
---

## Introduction
After this past weekend's kext-ageddon it once again became clear that Apple seems to have a problem with QA when it comes to OS X, as discussed on [this week's](http://atp.fm/episodes/159) Accidental Tech Podcast episode. While there are many areas of OS X where poor QA will likely go unnoticed by users until a later update fixes whatever QA missed, Apple has been unfortunate as of late with the specific areas of its OS X that were hit by poor QA.

There were several [App Store certificate expiration](http://www.macrumors.com/2015/11/12/mac-app-store-apps-damaged-expired-receipts-issue/) occurences that caught developers and users off-guard, the most recent one happening in mid-February as summarized [here](http://mjtsai.com/blog/2016/02/16/more-mac-app-store-certificate-problems/) by Michael Tsai. <!--more-->This latest certificate expiration has also warranted the [need to download new copies](http://tidbits.com/article/16302) of any OS X installers downloaded prior to the expiration. Users who depend on these expired OS X installer applications to deploy OS X from USB media or who are otherwise relying on the installer application will be greeted with an error that the application cannot be verified.

## Let's talk about kext, baby
All of this continuous black eye-inducing lack of QA brings us to the most recent [widely publicized](http://www.macrumors.com/2016/02/27/ethernet-not-working-imac-macbook-pro/) [trouble](http://www.businessinsider.com/apple-confirms-that-it-accidentally-broke-ethernet-ports-on-some-mac-computers-with-a-software-update-2016-2) surrounding Apple's own Ethernet drivers.

### How? Also, what?
While the how, the what and the fixes have been [written up elsewhere](https://derflounder.wordpress.com/2016/02/28/apple-security-update-blocks-apple-ethernet-drivers-on-el-capitan/) I will quickly run through the actual mechanism that made it possible for Apple to carry out a rather efficient DDoS attack on its own OS X platform by straight up disabling the network interface instead of just flooding it with traffic.

As part of the OS X Software Update mechanism Apple distributes certain updates that it deems to be more important than others such as Gatekeeper and XProtect updates, both of which deal with malware and system security. These items are given their own identifying type in the Software Update realm, named `config-data`. These items are represented in the App Store preference pane by a separate checkbox named "Install system data files and security updates". The possible impact of disabling all updates including aforementioned item by Mac Admins who manage their own Apple updates through other means has previously been covered by Tim Sutton in [this post](https://macops.ca/os-x-admins-your-clients-are-not-getting-background-security-updates). Most home users will not have made changes to the schedule however and they will be set to automatically and silently receive these `config-data` updates which generally is a positive thing in order to guarantee baseline protection against emerging malware.

It is through the existence of this silent and automatic update method that the destructive nature of Apple's SNAFU with its Ethernet drivers was made possible.

### Kext? What?
Kernel extensions (henceforth kexts) are what Apple calls self-contained bits of code that extend the kernel to allow non-critical or third party hardware and software to do what they do. Apple ships a fairly long list of its own kexts with the OS located in `/System/Library/Extensions` and third parties are supposed to place their kexts in `/Library/Extensions`. With the advent of SIP `/Library/Extensions` is the _ONLY_ valid location for third party kexts in addition to Apple's signing requirement: all kexts must be signed with a valid Apple developer account to be loaded by OS X.

[As we know](https://derflounder.wordpress.com/2015/10/01/system-integrity-protection-adding-another-layer-to-apples-security-model/) there is still an exception that can be made for this signing requirement by running `csrutil enable --without kext` from the Recovery partition but I would expect Apple to put the kibosh on that before long entirely.

While Apple can revoke any kext signing certificates in order to block identifiable "bad" kexts they decided to add a second layer of protection against terrible, no good, very bad kexts by adding the Inception-inducing AppleKextExcludeList kext which lives at `/System/Library/Extensions/AppleKextExcludeList.kext`. This self-identified kext doesn't actually contain any executable but instead contains two files that contain enumerations of different types of "bad" kext identifiers and their versions, and known kernel panic-inducing ones: `Info.plist` and `KnownPanics.plist`. During system boot kextd loads the `AppleKextExcludeList` kext and parses the two files within and if a match is found either by kext ID or by a more complicated regex query the kext in question is excluded and does not load.

At this point it should be obvious what happened here. Perhaps in preparation for an [upcoming OS X 10.11.4 update](http://www.macrumors.com/2016/03/01/apple-seeds-os-x-10-11-4-beta-5/) Apple engineers added the `AppleBCM5701Ethernet` and `AppleYukon2` Ethernet kexts to the `OSKextExcludeList` listing that is part of `AppleKextExcludeList`. Unfortunately this change, whatever its purpose was, did not get removed before release and went out as part of the __ Incompatible Kernel Extension Configuration Data 3.28.1__ update. And as we learned earlier, this update was tagged as `config-data` or "silent and automatic" which means it was picked up by Macs across the globe and installed without further hesitation by many of them.

### Again, what?
Now you might say, "Who cares? Who uses Ethernet to connect to a network anyway? That's so early 2000's!" That may be the case for those who are lucky enough to be part of a mobile/Agile/Starbucks-colocated work environment where Wifi and medium half-skim soy double shot machiatos are the only requirements for productivity. But for those larger environments wired desktops are still very much a reality. Examples include the Enterprise where users are on wired desktop Macs due to high speed network requirements for creative production or in Education where large shared computing sites and class labs will tend to use wired networking in order to reduce unnecessary load on available Wifi networks. And besides actual need, entirely disabling major components of a customer's computer is probably not the kind of promotion Apple needs right now.
