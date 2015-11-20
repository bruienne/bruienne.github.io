---
title: Another tool to fix PKGs with expired certs
author: pepijn
layout: post
permalink: /2012/04/13/another-tool-to-fix-pkgs-with-expired-certs/
categories:
  - Deployment
  - Development
tags:
  - certs
  - installer
  - pkg
---
Originally posted as reply to [Greg Neagle's post](http://managingosx.wordpress.com/2012/03/24/fixing-packages-with-expired-signatures/) regarding his very helpful tool to fix PKG installers with expired certs, this deserves some attention as it has the potential to be quite a bit faster because it doesn't do a full unflatten/flatten run on targeted PKGs:

[https://github.com/etrepum/strip\_pkg\_signature][1]

Go check it out.

 [1]: https://github.com/etrepum/strip_pkg_signature
