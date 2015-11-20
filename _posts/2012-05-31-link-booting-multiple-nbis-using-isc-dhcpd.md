---
title: 'Link - Booting multiple NBIs using ISC DHCPD'
author: pepijn
layout: post
permalink: /2012/05/31/link-booting-multiple-nbis-using-isc-dhcpd/
categories:
  - Deployment
tags:
  - bootp
  - dhcp
  - netboot
---
[Brandon Penglase](http://brandon.penglase.net/index.php?title=Main_Page) wrote up a very helpful wiki article on his site outlining how to configure ISC's DHCP server to serve multiple NetBoot images as opposed to the single image, methods for which have been available for a number of years now. Noted caveats are that Startup Disk will not be able to display the available NBIs as it uses a custom port to receive the list of images back and the inability to use thin NetBoot images that require server-side storage for the client.

[Go read it now.](http://bit.ly/JPc56D)
