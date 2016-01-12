---
title: Disable Spotlight indexing of network volumes
author: pepijn
layout: post
permalink: /2015/09/15/disable-spotlight-indexing-of-network-volumes/
categories:
  - Configuration
  - Management
  - Python
tags:
  - configuration
  - defaults
  - network
  - profile
  - spotlight
---
# Introduction

Based on a user request a question came up today regarding disabling the automatic Spotlight indexing of network volumes. While users can manually add local and network volumes to the Spotlight Privacy exception list this is not a very obvious process and it requires a tech to walk them through the process, usually more than once. When left unconfigured, Spotlight and its mds family of tools will continuously index network volumes, putting extra strain on network infrastructure and further degrading the already less-than-stellar SMB performance in OS X.<!--more-->

Not having a readily available solution available put me on the path of figuring out whether a more system-wide "killswitch" might be lurking somewhere. Most Mac Admins dealing with network volumes are likely aware of [the setting to disable](http://hints.macworld.com/article.php?story=2005070300463515) the creation of `.DS_Store` metadata on network volumes, but no similar setting to control network volume indexing was documented as far as I could tell.

## TL;DR

To have `mds` ignore all external volumes including network volumes run the following command:

<pre class="brush: bash; title: ; notranslate" title="">$ sudo defaults write /Library/Preferences/com.apple.SpotlightServer.plist ExternalVolumesIgnore -bool True</pre>

To be able to re-enable indexing for certain external volumes, run this command instead:

<pre class="brush: bash; title: ; notranslate" title="">$ sudo defaults write /Library/Preferences/com.apple.SpotlightServer.plist ExternalVolumesDefaultOff -bool True</pre>

## Diving in

Some poking around turned the attention to `mds` by way of `/System/Library/LaunchDaemons/com.apple.metadata.mds.plist` which invokes the `mds` binary at boot time. Next step was to open the executable in everyone's favorite disassembler [Hopper](http://www.hopperapp.com/) in order to see what we could see. I'm by no means a Hopper pro so I will typically perform some initial **"Labels"** or **"Strings"** searches for terms that are of interest. In this case I went with the generic *"Preferences"* query to get an idea of how and where `mds` might get and set its preferences. That initial search turned up some decent results as seen in Figure 1.

[<img src="{{ site.baseurl }}static/mds-hopper-1.png" alt="Figure 1" width="500" height="507" class="size-full wp-image-563" />][1]<em>Figure 1 - "Preferences" search</em>

Working in Hopper involves quite a bit of following rabbit holes down dead ends so I won't bore the reader with the results of following each of these results into the decompiled code but eventually I came across a rather apropos block of pseudo-code courtesy of Hopper's handy pseudo-code generator as seen in Figure 2.

<img src="{{ site.baseurl }}static/mds-hopper-21.png" alt="Figure 2" width="800" height="593" class="size-full wp-image-566" /><em>Figure 2 - Pseudo-code</em>

Highlighted are two preference keys that are called as `CFPreferencesGetAppBooleanValue` variables named `ExternalVolumesIgnore` and` ExternalVolumesDefaultOff` - exactly what we were hoping to find! Helpfully, this same block of code also contains some logging that describes exactly what the two keys do, leaving us less guesswork. The `ExternalVolumesIgnore` key logging string says:

> "ExternalVolumesIgnore" is set. All external volumes (except backup) will be ignored

Similarly the logging string for `ExternalVolumesDefaultIgnore` states:

> "ExternalVolumesDefaultOff" is set. All external volumes (except backup) will default off, override with mdutil -i on

Excellent. Those two keys would do very nicely in our quest to squelch the network-intensive file indexing by Spotlight. At this point it appears that the *"Ignore"* version of this preference concerning external volumes is the most irreversible and all-or-nothing one, while the *"Default Off"* version implies that the user will still be able to opt certain volumes back in should they wish to. For example, direct-attached storage devices like USB or Thunderbolt drives would not be indexed by default with the *"Default Off"* preference but the user would be able to enable them by running `mdutil -i on /Volumes/MyExternalDisk` from the command line.

# Progress made

With a good selection of prospective preference keys in hand the next step was to figure out *where* `mds` and/or Spotlight actually look for them. As a Hopper apprentice I usually start a search for possible preference file locations with a string search for *".plist"* as seen in Figure 3.

<img src="{{ site.baseurl }}static/mds-hopper-3.png" alt="Figure 3" width="500" height="377" class="size-full wp-image-579" /><em>Figure 3 - ".plist" search</em>

There are some good leads here, but it would be better to figure out exactly which of these plist files (or another one entirely) is where we'd want to write one of the two candidate keys to for testing. Through a little bit of crowdsourcing of opinions with the [Mac Admins Slack team](https://macadmins.slack.com/) and especially the help from [@frogor](https://twitter.com/mikeymikey) aka Mike Lynn in the #python channel I was able to determine the correct file. In an earlier search for *"Preferences"* I also found a reference to the `kMDSPreferencesName` variable which is used in direct relation to the `ExternalVolumes*` keys but I was unable to find a reference to the actual file it points to using Hopper. Calling upon Mike Lynn's notorious (and some would say legendary) [Python skills](http://michaellynn.github.io/2015/08/08/learn-you-a-better-pyobjc-bridgesupport-signature/) resulted in him whipping up [this code](https://gist.github.com/pudquick/ac8f22326f095ed2690e) which uses very clever PyObjc calls to determine the value of the variable `kMDSPreferencesName` - in this case `/Library/Preferences/com.apple.SpotlightServer.plist`. The exercise of finding this information inspired Mike to write up a more generic post on how to use the methods in this bit of code to get the kind of info I was looking for, so keep an eye on [his blog](http://michaellynn.github.io/) for that.

# Testing our findings

Back to our quest for the preference file to test our preference keys. Now that we know its name and location we can do some testing with it. By default `/Library/Preferences/com.apple.SpotlightServer.plist` does not exist, so we need to create it. The quickest way to do this is to use `defaults` on the command line. The following will simultaneously create `/Library/Preferences/com.apple.SpotlightServer.plist` and write the `ExternalVolumesIgnore` boolean "True" key:

<pre class="brush: bash; title: ; notranslate" title="">$ sudo defaults write /Library/Preferences/com.apple.SpotlightServer.plist ExternalVolumesIgnore -bool True</pre>

Or if we want to let the user opt in some external volumes at a later time if desired:

<pre class="brush: bash; title: ; notranslate" title="">$ sudo defaults write /Library/Preferences/com.apple.SpotlightServer.plist ExternalVolumesDefaultOff -bool True</pre>

Now when we mount a network volume we should see some logging in `/var/log/system.log` that indicates the volume is not being indexed:

<pre class="brush: bash; title: ; notranslate" title="">9/15/15 10:14:08.486 PM mds[18428]: (Normal) DiskStore: "ExternalVolumesIgnore" is set.  All external volumes (except backup) will be ignored</pre>

Success! Now `mds` is ignoring this and all other external volumes entirely and not indexing them, exactly what we were looking for.

# Conclusion & Download

To make testing this in other environments easier I am providing two profiles that set either the `ExternalVolumesIgnore` or `ExternalVolumesDefaultOff` key for deployment. Remember: test, test and then test again!

Sample configuration profile: [ExternalVolumesDefaultOff]({{ site.baseurl }}static/org.my.disablenexternalvolumesdefaultoff.mobileconfig)  
Sample configuration profile: [ExternalVolumesIgnore]({{ site.baseurl }}static/org.my.disablenexternalvolumesignore.mobileconfig)

 [1]: {{ site.baseurl }}static/mds-hopper-1.png
