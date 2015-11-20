---
title: Fixing Microsoft Office 2011 SP2 deployment for Munki
author: pepijn
layout: post
permalink: /2012/04/16/fixing-microsoft-office-2011-sp2-deployment-for-munki/
categories:
  - Bugfixes
  - Deployment
tags:
  - microsoft
  - munki
  - office
---
With the release of the recent [Microsoft Office for Mac 2011 SP2 update](http://www.microsoft.com/download/en/details.aspx?id=29419)  a new and unwelcome feature was introduced to Mac admins deploying Microsoft Office 2011 updates with patch management solutions such as Munki, Casper or Absolute Manage: [zombie mode](https://groups.google.com/d/msg/munki-dev/MbNCxvf-NfQ/rvkMfbyonxsJ). For reasons not entirely clear to anyone (including the Microsoft MBU folks themselves) the PKG install of SP2 causes Munki's Managed Software Update to hang at the final stage while displaying "Finishing the Installation.." This appears to be due to a hamfisted clean up attempt by the embedded *clean_path* script which causes MSU to appear frozen in the finishing stage. One can go in and manually kill off the sleep-cycling process or wait for the 2 hour timeout that Munki uses for running processes launched by the Munki supervisor to expire. Neither is elegant nor time-effective so in an effort to remove this one misbehaving script from the equation I edited the PKG's distribution.dist script and changed the following entry (line 251):

<pre>function volumeHasUpdatableVersionTest()
{
var result = false;
try {
//system.log("volumeHasUpdatableVersionTest: running volume_updatable " + my.target.mountpoint + " " + GetTempDirectory());
result = (system.runOnce('volume_updatable', my.target.mountpoint, GetTempDirectory()) == 0);
} catch (e) {system.log("volumeHasUpdatableVersionTest: mount: "+my.target.mountpoint+" exception: "+e);}
return result;
}</pre>

To simply read:

<pre>function volumeHasUpdatableVersionTest()
{
return true;
}</pre>

Since all that the code is doing is to compare the *sys.exit()* return code from *volume_updatable* to "0" and set *result* to "true" if it is, I decided to short-circuit the function and have it return "true" at all times. We'll assume that Munki has already determined that an upgradable version of Office 2011 was found based on entries in the pkginfo, so simply passing over the test for an updatable version was acceptable for my environment.

For completeness sake, skipping over *volumeHasUpdatableVersionTest()* will bypass the following Microsoft-provided scripts:

<pre>find_office
office_updatable
clean_path</pre>

I welcome feedback on whether this is successful for others as well. I'm sure it's possible to make a more targeted edit to prevent execution of just the *clean_path* script but I will leave that up to the adventurous Mac admins to determine.

Update: On 4/25/12 Microsoft released a patched SP2 updater, [version 14.2.1](http://support.microsoft.com/kb/2705358). This appears to have fixed the issue with the Outlook database corruption but still experiences the same issue as described in this post, even though the release notes state that too was corrected. I have verified that the post-install code is still the same and will hang up at the same script. Microsoft [has suggested](https://groups.google.com/d/msg/munki-dev/MbNCxvf-NfQ/R6mjenr3f5YJ) neutering *clean_path* by going into the script and changing it there but my fix as descibed above still works.
