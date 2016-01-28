---
title: Skipping Wifi setup in Setup Assistant
author: pepijn
layout: post
permalink: /2016/01/28/skip-wifi-in-setup-assistant/
categories:
  - Python
  - Management
  - MDM
  - DEP
  - Wifi
tags:
  - deployment
  - management
  - mdm
  - wifi
  - DEP
---

## Introduction

A question came up in the [Mac Admins Slack](https://macadmins.org) _#general_ channel regarding skipping the Wifi setup pane in Setup Assistant. The user _"tuxudo"_ wondered if any methods exist to skip the Wifi setup pane during a Setup Assistant run. The main use for suppressing the Wifi setup pane is to avoid confusion when deploying Macs that lack active wired networking and receive their Wifi configuration through a profile. Unfortunately the Wifi setup pane will still appear in this scenario as Setup Assistant has no knowledge of installed Wifi profiles. This can lead to support calls and changes being made that diverge from the desired Wifi configuration.<!--more-->

Looking for just the facts? Go [here](#tldr) now.

## To the Hopper-mobile!

Since I still had Hopper open with Setup Assistant to confirm something for [Rich Trouton](https://twitter.com/rtrouton) related to [Setup Assistant skipping Diagnostics and iCloud Setup skipping](https://derflounder.wordpress.com/2016/01/28/suppressing-the-icloud-and-diagnostics-pop-up-windows-on-el-capitan-using-profiles/) it was a quick trip back to Hopper to look for traces related to Wifi-skipping behavior. My general approach with Hopper as mentioned in previous posts is to do a simple string-based search for possible hits on terms related to the question at hand. In this case I started with the logical "Wifi" query and it didn't take long to find something rather promising-sounding.

<img src="{{ site.baseurl }}static/setupassistant-hopper-1.png" alt="Figure 1"/><em>Figure 1 - String search</em>

The path `/var/db/.MBSkipWiFiSetupIfPossible` makes sense as Apple uses hidden trigger files in `/var/db` regularly for various early-boot purposes like `.AppleSetupDone` as well as other lesser-known ones that manage DEP and MDM setup processes. Looking at the (awesome) pseudo-code that Hopper will generate for blocks of assembly code offers a pretty easy to follow logic flow in this case:

{% gist a9a85a0f875830de01fa %}

To break it down, the `r12` placeholder variable that Hopper uses when generating the pseudo-code holds the result of a function called `fileExistsAtPath` which takes a string containing a path to check. The use of the verb "exists" means it likely returns a boolean `True` or `False` or the equivalent integers `1` or `0`. Later in the code and after a few other Wifi-related operations an `if` statement checks for two conditions. We'll ignore the left-hand side of the `||` (or) logical operator side and pay attention to the right-hand side condition: `r12 != 0x0`:

```
if ((var_29 != 0x0) && (((var_38 == 0x0) || (r12 != 0x0))))
```

 This comparison will return `True` if the value of `r12` is not `0`. As we recall, at this point in the pseudo-code `r12` holds the result of checking whether or not the file `.MBSkipWiFiSetupIfPossible` at the path `/var/db` exists. As comparison with the value of `r12` uses `0x0` we can now assume that the `fileExistsAtPath` functions returns `1` if `/var/db/.MBSkipWiFiSetupIfPossible` exists. Since `r12` will be `1` if the path exists and `1` is not equal to `0` the result of the `if` statement is `True` and the corresponding code is executed:

```
rbx = @"CloudConfigurationWelcome";
```

This operation pushes the value `CloudConfigurationWelcome` onto the `rbx` register which is one of the [standard x86 registers](http://www.eecg.toronto.edu/~amza/www.mindsec.com/files/x86regs.html) called the Base register. In effect this causes Setup Assistant to go the "Welcome" screen instead of the Wifi setup screen. Thrilling stuff, agreed.

<a name="tldr"></a>

TL;DR: If the trigger file `/var/db/.MBSkipWiFiSetupIfPossible` is found on disk this code tells Setup Assistant to go directly to the "Welcome" screen and to skip the `SelectWiFiNetwork` pane if Airport hardware was found or to skip the `HowDoYouConnect` pane if no networking devices were found at all.

## Conclusion

These findings didn't remain theoretical for very long as _"tuxudo"_, the user who asked the question reported back the next day and confirmed that the Setup Assistant behavior is indeed as expected with the trigger file in place. His post on the topic can be found [here](https://www.afp548.com/2016/01/28/skipping-network-setup-in-setupassistant/). Great success!

To deploy the trigger file in a workflow and skip the Wifi setup window one approach could be a payload-free package as created with [munkipkg](https://github.com/munki/munki-pkg). This package might contain a `postinstall` script that runs `touch /var/db/.MBSkipWiFiSetupIfPossible` (with some possible error checking to be safe) as part of pre-boot configuration with tools like [AutoDMG](https://github.com/MagerValp/AutoDMG), [Imagr](https://imagr.io) or [DeployStudio](http://www.deploystudio.com). The only caveat is that a working Wifi profile must have been installed prior to Setup Assistant running (h/t **tuxudo**). This can be done by placing such a profile in `/var/db/ConfigurationProfies/Setup` and deleting `/var/db/ConfigurationProfies/Setup/.profileSetupDone` which will trigger an automatic installation at boot time. The aforementioned tools can be used to install the profile and remove the trigger file to prime the profile's installation.
