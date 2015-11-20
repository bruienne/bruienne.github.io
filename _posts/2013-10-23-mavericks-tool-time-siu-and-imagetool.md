---
title: 'Mavericks tool time - SIU and imagetool'
author: pepijn
layout: post
permalink: /2013/10/23/mavericks-tool-time-siu-and-imagetool/
categories:
  - Development
tags:
  - cli
  - mavericks
  - netboot
  - python
---
Recently I've had to rebuild our customized NBI NetBoot images a number of times due to special OS builds (yay) and needing to test Mavericks DPs. In that process it became obvious that it's easy to make a mistake adding certain resources, deleting others and making sure that the resulting DMG is resized afterwards. I don't know about you, but if I have to repeatedly and manually run a bunch of error-prone steps my mind quickly turns towards *automating the heck out of it* to regain sanity and remove error.<!--more-->

Wanting to cut my teeth on some more Python I figured I would seize this particular itch to do that. Having been generally happy with System Image Utility (SIU) my first approach was to create an SIU workflow and then figure out a way to execute it from Python. This, I surmised, required tapping the power of the Automator framework with PyObjc through things like AMWorkflow.runWorkflowAtURL\_withInput\_error_(). Not the easiest for a first PyObjc project, admittedly. After some scary times wandering in the PyObjc desert kicking the Automator Framework around and not making a lot of headway I decided to see if perhaps System Image Utility had a CLI mode I could subvert for my needs. Thus I stumbled upon '**/System/Library/CoreServices/System Image Utility.app/Contents/MacOS/imagetool**' which I didn't remember seeing before and indeed appears to be new in Mavericks.

Long story short: *imagetool* is a CLI tool that can generate NetBoot/Install/Restore NBI bundles ready for deployment as well as regular bootable installer media. Since we already have the "createinstallmedia" tool contained in the Install OS X Mavericks.app bundle ([written up so very nicely](http://bit.ly/H9L5hH) by Rusty Meyers) I will not go into the -createmedia (-c) function here as it does the same thing.

The usage info for *imagetool* is as follows:

<pre>/System/Library/CoreServices/System\ Image\ Utility.app/Contents/MacOS/imagetool

Usage: imagetool &lt;options&gt;
  where &lt;options&gt; are:
   -c | --createmedia -s &lt;path_to_install_app&gt; -d &lt;path_to_volume&gt;
  or:
   -p | --plist &lt;path&gt; path to a property list containing the build specifications
 and/or:
   [--netboot | --netinstall | --netrestore] image kind (required)
   -d | --destination &lt;path&gt; path to save the image (required)
   -s | --source &lt;mountpoint&gt; mountpoint of source volume or path to Install Mac OS X
        application (required)
   -n | --name &lt;string&gt; name of image (required)
   -i | --index &lt;integer&gt; image index (0-65535) (required)

   -h | --help this help</pre>

The help text goes on to say:

<pre>To create a property list, create a custom workflow in System Image Utility.
Add any customized steps you wish the workflow to perform.
Hold down the option key when clicking the run button in the workflow.
This will save the workflow detail to the specified location instead of executing it.</pre>

While it would be perfectly fine to feed imagetool some CLI parameters the -plist option intrigued me. I tested the "option-click Run to save" (so intuitive, Apple) in SIU and it does indeed save a plist file, different in structure from the usual .wflow document.

[<img class="alignnone size-full wp-image-213" alt="SIU_custom" src="http://enterprisemac.bruienne.com/wp-content/uploads/2013/10/SIU_custom.png" width="500" height="391" />][1]  
[<img class="alignnone size-full wp-image-214" alt="SIU_saveplist" src="http://enterprisemac.bruienne.com/wp-content/uploads/2013/10/SIU_saveplist.png" width="500" height="391" />][2]  
Upon opening the file we see a pretty straightforward set of configuration items:

<pre class="brush: xml; title: ; notranslate" title="">&lt;?xml version="1.0" encoding="UTF-8"?&gt;
&lt;!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd"&gt;
&lt;plist version="1.0"&gt;
&lt;dict&gt;
	&lt;key&gt;imageDescription&lt;/key&gt;
	&lt;string&gt;NetInstall of OS X 10.9 (13A603) Install (7.14 GB)&lt;/string&gt;
	&lt;key&gt;imageIndex&lt;/key&gt;
	&lt;integer&gt;1621&lt;/integer&gt;
	&lt;key&gt;imageName&lt;/key&gt;
	&lt;string&gt;NetInstall of Install OS X Mavericks&lt;/string&gt;
	&lt;key&gt;installType&lt;/key&gt;
	&lt;string&gt;netinstall&lt;/string&gt;
	&lt;key&gt;nbiLocation&lt;/key&gt;
	&lt;string&gt;/Path/To/NetInstall of Install OS X Mavericks&lt;/string&gt;
	&lt;key&gt;sourcesList&lt;/key&gt;
	&lt;array&gt;
		&lt;dict&gt;
			&lt;key&gt;dmgPath&lt;/key&gt;
			&lt;string&gt;/Path/To/InstallESD.dmg&lt;/string&gt;
			&lt;key&gt;isInstallMedia&lt;/key&gt;
			&lt;true/&gt;
			&lt;key&gt;kindOfSource&lt;/key&gt;
			&lt;integer&gt;1&lt;/integer&gt;
			&lt;key&gt;sourceType&lt;/key&gt;
			&lt;string&gt;ESDApplication&lt;/string&gt;
			&lt;key&gt;volumePath&lt;/key&gt;
			&lt;string&gt;/Path/To//Install OS X Mavericks.app&lt;/string&gt;
		&lt;/dict&gt;
	&lt;/array&gt;
	----SNIP----
</pre>

These are all as one would expect to see - image description, name, index as well as the location it will be written out to and source(s) of the InstallESD.dmg to use. If the "Add Packages and Post-Install Scripts" workflow item is included its settings are recorded as follows:

<pre class="brush: xml; title: ; notranslate" title="">&lt;key&gt;packageList&lt;/key&gt;
	&lt;array&gt;
		&lt;string&gt;/Users/admin/munkitools-0.9.0.1803.0.mpkg&lt;/string&gt;
		&lt;string&gt;/Users/admin/CleanImage-1.0.pkg&lt;/string&gt;
	&lt;/array&gt;
	&lt;key&gt;scriptList&lt;/key&gt;
	&lt;array/&gt;
</pre>

In addition to the above options the .plist contains a lengthy "userConfigurationOptions" dict containing a "groupID" and "userID" key which are set to that of the user running SIU. A third key named "siuPrefs" (containing a dict of other strings, integers, arrays and dicts) appears to contain a few SIU-specific keys but also many settings originating from the user creating the NetBoot image. In my testing these keys contained ColorSync profile settings, Text Replacement configs and even configurations for third party applications.

The SIU-related keys are:

<pre class="brush: xml; title: ; notranslate" title="">&lt;key&gt;addlNetBootMbytes&lt;/key&gt;
&lt;integer&gt;400&lt;/integer&gt;
&lt;key&gt;asr_blockCopyVolume&lt;/key&gt;
&lt;true/&gt;
&lt;key&gt;asr_displayCountdown&lt;/key&gt;
&lt;false/&gt;
&lt;key&gt;asr_dontReorderForMulticast&lt;/key&gt;
&lt;false/&gt;
&lt;key&gt;asr_retainOriginalVolumeName&lt;/key&gt;
&lt;true/&gt;
&lt;key&gt;consumeSuppliedImage&lt;/key&gt;
&lt;false/&gt;
&lt;key&gt;createEnabledImages&lt;/key&gt;
&lt;true/&gt;
&lt;key&gt;createSparseImages&lt;/key&gt;
&lt;false/&gt;
&lt;key&gt;enableInternalLogging&lt;/key&gt;
&lt;false/&gt;
&lt;key&gt;installToImageDiskFormat&lt;/key&gt;
&lt;string&gt;HFS+J&lt;/string&gt;
&lt;key&gt;kSIUWorkflowDirectoryKey&lt;/key&gt;
&lt;string&gt;/Users/admin/Documents&lt;/string&gt;
&lt;key&gt;lastImageLocation&lt;/key&gt;
&lt;string&gt;/Users/admin/Desktop&lt;/string&gt;
&lt;key&gt;liveUpdateInstallerSearch&lt;/key&gt;
&lt;true/&gt;
&lt;key&gt;restoreImageFormat&lt;/key&gt;
&lt;string&gt;UDZO&lt;/string&gt;
</pre>

Since I didn't want to retain most of the user settings I removed the entire "userConfigurationOptions" dict as a test and subsequent images built and booted without a problem. Taking all of the above into account it is therefore fairly straightforward to put together one or more .plist files that can be fed to imagetool with an invocation like this:

<pre class="brush: bash; title: ; notranslate" title="">sudo '/System/Library/CoreServices/System Image Utility.app/Contents/MacOS/imagetool' --plist '/Users/admin/Documents/NetRestore Template.plist' --index 3000
</pre>

This will tell imagetool to use the configuration in "NetRestore Template.plist", overriding the "imageIndex" key in the plist and substituting it with "3000". Note that while my example uses Mavericks as an installer source I have been able to verify that substituting a Mountain Lion (10.8) or Lion (10.7) installer app and InstallESD.dmg also works without a hitch and the resulting NBIs were all bootable.

As you can see, some interesting things are possible with this entirely CLI-driven process. I will be posting a tool written in Python in the next few days that leverages imagetool to automate the creation and processing of plists to generate and modify NetBoot images for use in mass deployments of OS X.

 [1]: http://enterprisemac.bruienne.com/wp-content/uploads/2013/10/SIU_custom.png
 [2]: http://enterprisemac.bruienne.com/wp-content/uploads/2013/10/SIU_saveplist.png
