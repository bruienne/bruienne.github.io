---
ID: 436
post_title: >
  Adding Python or Ruby to custom
  NetInstall images with AutoNBI
author: admin
post_date: 2015-04-21 15:41:34
post_excerpt: ""
layout: post
permalink: >
  http://enterprisemac.bruienne.com/2015/04/21/creating-custom-netboot-images-with-autonbi/
published: true
---
A recent update to <a href="https://bitbucket.org/bruienne/autonbi" title="" target="_blank">AutoNBI</a>, a tool I wrote to automate the creation of custom Apple NetInstall images (NBIs), expands its customization abilities. So far an admin has been able to essentially forklift a custom folder into the NBI, as explained quite nicely by Graham Gilbert in recent blog posts <a href="http://bit.ly/1bfT2k0" title="" target="_blank">here</a> and <a href="http://grahamgilbert.com/blog/2015/04/13/more-fun-with-autonbi/" title="" target="_blank">here</a>. The immediate use for this is to replace the "Packages" folder on a standard NetInstall volume with one that has been prepped with a custom rc.imaging file and additional custom tools meant to be run at boot time such as a lightweight disk imaging or no-imaging tool. This feature works for applications that are fully self-contained like a compiled Cocoa app, but is not as useful if the application has a dependency on frameworks like Python or Ruby which are not part of the default NetInstall environment. The updated version of AutoNBI now offers the option to include either one or both of the Python or Ruby frameworks into the NetInstall BaseSystem.dmg allowing custom scripts written in either language to be run. The first tool to leverage this ability is Graham Gilbert's very promising <a href="https://github.com/grahamgilbert/imagr" title="" target="_blank">Imagr</a> tool which is written in PyObjc and thus relies on the availability of the Python framework in /System/Library/Frameworks.

I'm looking at including other potentially useful add-ons such as VNC or SSH while sticking to the overall goal of keeping the boot environment lightweight in order to provide short boot times and minimal network load.

A special word of thanks goes to every Mac Admin's favorite <a href="https://gist.github.com/pudquick" title="" target="_blank">Python whisperer</a> <a href="https://twitter.com/mikeymikey" title="" target="_blank">Michael Lynn</a> for figuring out how to parse Apple's custom wrapper around OS X Yosemite installer sources without which it would have been nearly impossible to add these new features.

The current main branch contains the changes, so go <a href="https://bitbucket.org/bruienne/autonbi/src" title="" target="_blank">check it out</a>! Updated instructions can be found in the Readme on the Bitbucket repository.