---
ID: 165
post_title: 'Link &#8211; Booting multiple NBIs using ISC DHCPD'
author: pepijn
post_date: 2012-05-31 09:28:20
post_excerpt: ""
layout: post
permalink: >
  http://enterprisemac.bruienne.com/2012/05/31/link-booting-multiple-nbis-using-isc-dhcpd/
published: true
---
<a href="http://brandon.penglase.net/index.php?title=Main_Page" target="_blank">Brandon Penglase</a> wrote up a very helpful wiki article on his site outlining how to configure ISC's DHCP server to serve multiple NetBoot images as opposed to the single image, methods for which have been available for a number of years now. Noted caveats are that Startup Disk will not be able to display the available NBIs as it uses a custom port to receive the list of images back and the inability to use thin NetBoot images that require server-side storage for the client. <a href="http://bit.ly/JPc56D" target="_blank">Go read it now.</a>