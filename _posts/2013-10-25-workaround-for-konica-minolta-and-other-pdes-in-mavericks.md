---
title: Workaround for Konica Minolta (and other) PDEs in Mavericks
author: pepijn
layout: post
permalink: /2013/10/25/workaround-for-konica-minolta-and-other-pdes-in-mavericks/
categories:
  - Bugfixes
  - Development
tags:
  - konica
  - mavericks
  - minolta
  - pde
---
Is your organization using those really shiny and fancy Konica Minolta multifunction printers? Did your users start upgrading to Mavericks only to find that none of the custom functionality menus (courtesy of KM [PDE](http://bit.ly/HkuVCw)s) were available in the Print window? [Try this workaround](http://bit.ly/HkqHeb) to make them show up again in the print dialogs of sandboxed apps (Preview, TextEdit). Note that the script can easily be modified to provide the same workaround for other vendors' incompatible PDEs as well. And make sure to use sudo, of course.

**Update:** To make applying this workaround a little easier I have created a 'nopkg' Munki pkginfo that will apply it to any PDEs found in a user's /Library/Printers folder that are missing the required key in their Info.plist. It is up to the Mac admin to insert the names of the appropriate printer driver install items for which this patch should be an update. Get the pkginfo item [right here.](https://gist.github.com/bruienne/7327076)

**Update 2:** For those having trouble figuring out how to apply this workaround or those who don't use Munki for their patch management I have put up a [standalone version](https://db.tt/TgbsfLps) of the script. Simply unzip the file and run with sudo:

`sudo ./mavericks_pde_fix.py`

Changes will only be made to PDEs that are missing a key in their Info.plist required for Mavericks compatibility.
