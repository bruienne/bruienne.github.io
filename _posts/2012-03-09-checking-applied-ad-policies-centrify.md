---
ID: 100
post_title: >
  Checking applied AD policies with
  Centrify DirectControl agent
author: pepijn
post_date: 2012-03-09 16:43:55
post_excerpt: ""
layout: post
permalink: >
  http://enterprisemac.bruienne.com/2012/03/09/checking-applied-ad-policies-centrify/
published: true
---
While troubleshooting some policy behavior using Centrify DirectControl 5.0.2 on a test Mac I found myself sorely missing the Centrify-native version of **"gpresult"**. Centrify implements **"adgpupdate"** which behaves much like its Windows counterpart but in order to look at applied policies one is left tool-less. Luckily all retrieved and applied policies can be found on the local filesystem, and perused from there. To see the policies navigate to `/var/centrifydc/reg` and as root one can inspect both computer and user policies: `bash-3.2# ls -l
total 0
drwxr-xr-x  7 root  wheel  238 Dec  1 10:40 machine
drwxr-xr-x  5 root  wheel  170 Mar  6 11:16 users` The **gp.report** file contains all applied policies and their settings in `/var/centrifydc/reg/machine` and `/var/centrifydc/reg/users/SOMEUSER:` `bash-3.2# ls machine/
.lock gp.report software
applied_policies secedit` ... `bash-3.2# ls users/user2/
gp.report software` The files themselves look like basic .reg files with each policy rule occupying one line preceded by a machine or user-specific configuration stanza, which is updated by Centrify's tools when policies are updated. It is probably A Very Bad Idea to make any manual changes here. The raw .pol files as pulled from your domain's SYSVOL can be found in the **software** directory and its sub-directories for both users and machine, machine-specific security .pol files are stored in **secedit**. The **applied_policies** file lists the GUIDs of all applied policies as pulled from LDAP, they are complete DNs, one per line. My next step is going to be corralling some of this information into a single script along the lines of "adgpresult" to make the desired info a little easier to get to. But for now this is at least one way to get to the GPO policies for Macs using Centrify DirectControl.