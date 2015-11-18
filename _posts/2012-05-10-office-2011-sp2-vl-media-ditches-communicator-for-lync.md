---
ID: 159
post_title: >
  Office 2011 SP2 VL media ditches
  Communicator for Lync
author: pepijn
post_date: 2012-05-10 11:46:04
post_excerpt: ""
layout: post
permalink: >
  http://enterprisemac.bruienne.com/2012/05/10/office-2011-sp2-vl-media-ditches-communicator-for-lync/
published: true
---
After an earlier <a href="https://twitter.com/#!/golby/status/200612050697850880" target="_blank">Twitter exchange with @golby</a> today I realized that as of the <a href="http://support.microsoft.com/kb/2685940" target="_blank">SP2</a> VL version of Office 2011 Microsoft has removed Communicator 13.x (also known as <a href="http://www.microsoft.com/mac/enterprise/communicator" target="_blank">Communicator 2011 for Enterprise</a>) in favor of their current <a href="http://www.microsoft.com/mac/enterprise/lync" target="_blank">Lync for Mac 2011</a> application, version 14.0.2. This is an important change for those on both sides of the Lync-coin: if you didn't upgrade to Lync from OCS yet, you will now need to remove the Lync client during pre- or post-install and install the separately available Communicator 2011 PKG from the VL site. Since Communicator 2011 is not available outside of the VL site you will need to have yourself granted access to the site, or ask someone with access to download it. Those who are using Lync Server (<a href="http://lync.microsoft.com/en-us/get-lync/Pages/lync-licensing.aspx" target="_blank">Standard or Enterprise</a>) and Lync for Mac 2011 will need to adjust their installation recipes since up to this point one would have had to install Lync for Mac 2011 separately, and remove the Communicator 2011 application either in pre- or post-install. In my case I am installing Office 2011 and Lync for Mac 2011 using Munki and since all users get the Lync client I have it requiring the Office 2011 installer. That will now have to be changed to just install Office 2011 SP2 while separately maintaining updates for Lync beyond the included 14.0.2 version.