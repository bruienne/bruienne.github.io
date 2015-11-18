---
ID: 227
post_title: >
  Workaround for Konica Minolta (and
  other) PDEs in Mavericks
author: pepijn
post_date: 2013-10-25 19:30:10
post_excerpt: ""
layout: post
permalink: >
  http://enterprisemac.bruienne.com/2013/10/25/workaround-for-konica-minolta-and-other-pdes-in-mavericks/
published: true
---
Is your organization using those really shiny and fancy Konica Minolta multifunction printers? Did your users start upgrading to Mavericks only to find that none of the custom functionality menus (courtesy of KM <a href="http://bit.ly/HkuVCw" target="_blank">PDE</a>s) were available in the Print window? <a href="http://bit.ly/HkqHeb" target="_blank">Try this workaround</a> to make them show up again in the print dialogs of sandboxed apps (Preview, TextEdit). Note that the script can easily be modified to provide the same workaround for other vendors' incompatible PDEs as well. And make sure to use sudo, of course. **Update:** To make applying this workaround a little easier I have created a 'nopkg' Munki pkginfo that will apply it to any PDEs found in a user's /Library/Printers folder that are missing the required key in their Info.plist. It is up to the Mac admin to insert the names of the appropriate printer driver install items for which this patch should be an update. Get the pkginfo item <a href="https://gist.github.com/bruienne/7327076" target="_blank">right here.</a> **Update 2:** For those having trouble figuring out how to apply this workaround or those who don't use Munki for their patch management I have put up a <a href="https://db.tt/TgbsfLps" target="_blank">standalone version</a> of the script. Simply unzip the file and run with sudo:  
  
`sudo ./mavericks_pde_fix.py`  
  
Changes will only be made to PDEs that are missing a key in their Info.plist required for Mavericks compatibility.