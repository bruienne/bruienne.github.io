---
ID: 91
post_title: Custom Munki Conditional Items
author: pepijn
post_date: 2012-03-08 09:12:08
post_excerpt: ""
layout: post
permalink: >
  http://enterprisemac.bruienne.com/2012/03/08/custom-munki-conditional-items/
published: true
---
Courtesy of Heig Gregorian, munkitools **0\.8.2 Build 1459** (and later) now has the ability to add custom conditional item entries using your favorite scripting language (Ruby, Python, bash). It does this by executing compatible scripts in **<tt>/usr/local/munki/conditions</tt>** to write key/value pairs to the newly added **<tt>ConditionalItems.plist</tt>** which lives in the Managed Installs directory, **<tt>/Library/Managed Installs</tt>** by default. Heig has updated the <a href="http://code.google.com/p/munki/wiki/ConditionalItems" target="_blank">ConditionalItems</a> wiki page on the Munki project page to reflect the added functionality. This very welcome addition allows for some very interesting customization of Munki's conditional_items functionality and I thank Heig for writing the code and Greg for merging it into munkitools.