---
title: Box cutting, or how I stumbled onto a serious security flaw in Box Sync for Mac
author: pepijn
layout: post
permalink: /2015/02/07/box-cutting-how-i-stumbled-onto-a-serious-security-flaw-in-box-sync-for-mac/
categories:
  - Deployment
  - Management
  - Python
  - Security
tags:
  - box
  - configuration
  - vulnerability
---
**TL;DR - Update to [Box Sync for Mac 4.0.6035][1] immediately. The app exposes several sensitive bits of data like API keys, internal user IDs, URLs and passwords. Read on for details.**

### The trouble with Box Sync

Recently I revisited the convoluted mess that is the Box Sync application for Mac. If you are a Mac Admin in charge of even a small deployment environment you probably know how tedious it is to deploy the Box Sync application and manage its settings. Its only deployment method is an application bundle, which would be fine if it behaved like a normal drag and drop application: to deploy it your mass-deployment tool simply copies the application to `/Applications`, uses a profile or MCX to configure settings published by the vendor and all is well. Not so with Box Sync. <!--more-->Box offers [instructions](https://support.box.com/hc/en-us/articles/200520648-Box-Desktop-Applications-And-Installers-IT-Instructions-for-Large-Scale-Silent-MSI-DMG-Installations#sync" title=") on how to deploy Box Sync for large scale clients which require the Mac Admin does the following:

  * Copy the Box Sync application to `/Applications`
  * Copy `/Applications/Box Sync.app/Contents/Resources/com.box.sync.bootstrapper` to `/Library/PrivilegedHelperTools`
  * Run `sudo /Applications/Box Sync.app/Contents/Resources/com.box.sync.bootstrapper` which performs first run setup

To many Mac Admins it will occur that these steps would be better handled by using a standard Apple PKG, and they would be correct. Performing manual copy operations followed by running commands to finalize the installation by hand doesn't scale very well. [Apple installer packages][2] are rather good at placing files in specific filesystem locations with specific ownerships and permissions (step one and two in the Box Sync install process) and afterwards running post-install commands to further configure the application for use by the end user (step three). Further complications arise once the Box Sync application is installed because of its default behavior to automatically update in the background. This is fine for home users but in a large deployment where an admin maintains a workflow where all software updates flow through "testing", "QA" and "production" stages this is problematic. Very little information is available from Box regarding changing the application's default preferences, nor does the application's Settings tab offer much:  
<img src="http://enterprisemac.bruienne.com/wp-content/uploads/2015/02/box-prefs.jpg" alt="box-prefs" width="500" height="403" class="alignnone size-full wp-image-388" />  
In our environment we follow the aforementioned testing/QA/production workflow so having the Box Sync client update itself without allowing us to ensure its compatibility within our environment was a problem. Lacking any documentation from Box and on a hunch I took it upon myself to check out if there was any configuration information buried inside the Box Sync bundle.

### Time to go spelunking

At first glance the app bundle contents seem pretty unassuming:  
<img src="http://enterprisemac.bruienne.com/wp-content/uploads/2015/02/box-1.jpg" alt="box-1" width="400" height="282" class="alignnone size-full wp-image-386" />  
Nothing too interesting in Frameworks either, just Python and Growl. The bundled Python framework makes sense (though, what's wrong with System Python?) since we know that the Box service, [like Dropbox's](http://en.wikipedia.org/wiki/Dropbox_%28service%29#Technology), is partly written in Python:  
<img src="http://enterprisemac.bruienne.com/wp-content/uploads/2015/02/box-2.jpg" alt="box-2" width="400" height="332" class="alignnone size-full wp-image-387" />  
The Resources folder has a lot of PNG images used as icons for document types and the UI, as well as a number of [.nib files][3] so we'll ignore those. We see some more signs of Box being written in Python:  
[<img src="http://enterprisemac.bruienne.com/wp-content/uploads/2015/02/box-3.jpg" alt="box-3" width="400" height="522" class="alignnone size-full wp-image-389" />][4]  
Let's check out the `include` and `lib` folders, familiar-looking names for anyone who has had to install a Python application that has Python modules as dependencies:  
<img src="http://enterprisemac.bruienne.com/wp-content/uploads/2015/02/box-4.jpg" alt="box-4" width="400" height="479" class="alignnone size-full wp-image-390" />  
It appears that the `include` folder contains just the `pyconfig.h` header file, so we'll look at the `lib` folder which seems to have more contents:  
<img src="http://enterprisemac.bruienne.com/wp-content/uploads/2015/02/box-5.jpg" alt="box-5" width="400" height="200" class="alignnone size-full wp-image-391" />  
The most interesting item here is the `site-packages.zip` file which is another familiar name to those who have deployed Python applications before. OS X itself has a folder named `site-packages` located inside `/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7`. Inside are third party Python modules that can be installed through `easy_install` or `pip` and are meant to be available to all Python applications. Let's see what's inside, shall we? The unzipped `site-packages` folder is over 20 MB in size, containing 51 subfolders and nearly 140 `.pyo` files. The `.pyo` files are [Python optimized bytecode](https://docs.python.org/release/1.5.1p1/tut/node43.html) files, essentially just like the `.pyc` files Python automatically creates when running a Python `.py` program file. More about these in a bit as we dive into the contents of the folders next.

### Further down the rabbit hole we go

Most of the folders contain commonly used Python modules one would expect to see in an app of Box Sync's magnitude: a number of [PyObjc modules](https://bitbucket.org/ronaldoussoren/pyobjc/src) to call Apple frameworks, XML, SSL, JSON, SQLite, NTLM modules and various other modules that don't seem too interesting. What does look promising are the `box` and `boxsdk` folders:  
<img src="http://enterprisemac.bruienne.com/wp-content/uploads/2015/02/box-6.jpg" alt="box-6" width="375" height="405" class="alignnone size-full wp-image-395" />  
Looking at the contents of the `box` folder our eye is drawn to the `conf` subfolder since we're still hunting for clues about somehow configuring the application a little better. At this point we'll look at the Python optimized bytecode filetype again that we've determined all the `.pyo` files have. A quick Google search of *"Python pyo files"* tells us that they are trivial to revert back to regular Python code using the [uncompyle2](https://pypi.python.org/pypi/uncompyle2/1.1) tool which is available through either `easy_install` or `pip`. Once installed we run `uncompyle2` against the contents of the entire `site-packages` folder using the `-r` switch to recursively process it. If there's other places where interesting configuration options may be lurking, we're bound to find them.

<pre class="brush: bash; title: ; notranslate" title="">$ easy_install uncompyle2
$ uncompyle2 -o /tmp/site-packages-decomp -r -p 8 /Applications/Box Sync.app/Contents/Resources/lib/site-packages
# 2015.02.07 01:36:54 EST
decompiled 1 files: 1 okay, 0 failed, 0 verify failed
decompiled 1 files: 1 okay, 0 failed, 0 verify failed
decompiled 1 files: 1 okay, 0 failed, 0 verify failed
decompiled 1 files: 1 okay, 0 failed, 0 verify failed
decompiled 1 files: 1 okay, 0 failed, 0 verify failed
decompiled 1 files: 1 okay, 0 failed, 0 verify failed
&lt;snip&gt;
</pre>

After the processing is done we check out the contents of `/tmp/site-packages-decomp` and see that `uncompyle2` produced files with a `.pyo_dis` extension, which are just plain `.py` files at this point. We could rename them all, but any text editor will read them now so we won't bother.  


### Deep dive for plists

Opening up the `/tmp/site-packages-decomp` folder in an app like Textmate or BBEdit is going to be the easiest way to search the entire codebase for anything related to preferences, like a plist file. To start off we'll use Textmate 2 to search for any files containing the text "plist":  
<img src="http://enterprisemac.bruienne.com/wp-content/uploads/2015/02/box-search-plist1.jpg" alt="box-search-plist" width="600" height="449" class="alignnone size-full wp-image-404" />  
That wasn't too hard, was it? It looks like the file at `site-packages/box/conf/base.pyo_dis` has references to a file named `/Library/Preferences/com.box.sync.plist`:

<pre class="brush: bash; title: ; notranslate" title="">conf.set(u'preferences.mac_plist_file.path', u'/Library/Preferences/com.box.sync.plist')</pre>

Eureka! This file is normally not to be found on systems with Box Sync installed and configured for the user, so this is a great start. Some more searching reveals the `_overridable_settings` list in `configuration.py` which contains a key named `auto_update.enabled`! Exactly the kind of setting we were looking for. Its inclusion in a list of settings named "overridable settings" further increases our confidence. To test that this setting actually works we start by writing a new `/Library/Preferences/com.box.sync.plist` file using `defaults` like so:

<pre class="brush: bash; title: ; notranslate" title="">$ sudo defaults write /Library/Preferences/com.box.sync.plist auto_update.enabled -bool False</pre>

And indeed, when tested by installing an older version of Box Sync with the `/Library/Preferences/com.box.sync.plist` preference file in place with the `auto_update.enabled` key set to `False` the application indeed no longer attempts to auto-update. Mission accomplished, go home? Well&#8230; almost.  


### Things get kinda real at this point.

Once I started to scroll around a bit more in the `base.py` and `auto_update_release.pyo` files and saw **what else** was in it, my reaction can be summarized as follows:

<img src="http://enterprisemac.bruienne.com/wp-content/uploads/2015/02/wherearewe.gif" alt="wherearewe" width="245" height="141" class="alignnone size-full wp-image-406" />

As it turns out, the development team at Box embedded a *lot* of rather sensitive information in the files belonging to the `conf` module. A quick scan reveals sensitive-looking key/value pairs such as:

<pre class="brush: bash; title: ; notranslate" title="">app.remote_control_port
app.remote_control_auth_key
api_key #(Mac/Windows specific)
client_secret #(Mac/Windows specific)
upload_only_auth_token_for_log_folder
auto_update.internalqa_user_ids
mac_api_key
win_api_key
mac_client_secret
win_client_secret
au_password
v1_internal_admin_login_url
s3_access_key
s3_secret_key
jenkins_url
</pre>

This is probably not a good thing. Since bots exist that [scan Github][5] and other public version control services for unintentionally checked in API keys and secrets, Box probably didn't mean to expose all this information in the Box Sync application. *To be clear*, I did not try to use any of the information I found to gain access to any Box systems. I am also not publishing the full source code or complete key/value pairs, this is left up to the inquisitive reader to pursue. In early January, after realizing what I had found, I reported the issue to the Box Security team and after receiving acknowledgment of the severity of the issue I was asked to delay disclosure to give the Box development team time to develop and ship a fix. On February 6th I was notified that an updated version **4.0.6035** had been released which is supposed to resolve the issue. Since the update is now available I am publishing my findings in order to give a heads up to fellow Mac Admins and anyone else who uses or deploys Box Sync to ensure that the 4.0.6035 update is applied ASAP. There is no way of knowing who else has been aware of the exposed information before me and whether or not it may have been used to access Box customer data. This is especially important in environments that use a managed software update workflow which may be holding back automatic updates until specific action is taken by an admin.  


### Conclusion

I hope this information will be useful to Mac Admins and individual Mac users alike and again stress that every Box Sync user make sure that their installed version is at version **4.0.6035** or above.

 [1]: https://app.box.com/sync4mac
 [2]: http://s.sudre.free.fr/Software/Packages/resources.html
 [3]: https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/LoadingResources/CocoaNibs/CocoaNibs.html
 [4]: http://enterprisemac.bruienne.com/wp-content/uploads/2015/02/box-3.jpg
 [5]: http://it.slashdot.org/story/15/01/02/2342228/bots-scanning-github-to-steal-amazon-ec2-keys
