---
title: Installing OS X PKGs using an MDM service
author: pepijn
layout: post
permalink: /2015/11/17/installing-os-x-pkgs-using-an-mdm-service/
categories:
  - Deployment
  - Management
  - MDM
tags:
  - DEP
  - deployment
  - installer
  - management
  - mdm
---
Just the facts? Go [here!](#tldr)

## Introduction

With each annual iteration of OS X Apple [improves](https://developer.apple.com/videos/play/wwdc2015-301/?time=201) the capabilities of its tightly integrated MDM, DEP and VPP trinity which has also made it increasingly compelling for Mac admins to take a look at what these management options could mean for them.<!--more--> While an admin may be quite content with an all-OSS setup incorporating thin imaging and Munki, the potential of a [zero-touch](http://www.jamfsoftware.com/resources/webinar/vpp-dep-leveraging-apples-it-deployment-programs/) deployment through DEP + MDM is enticing. Up until now there has been no clear way to tap into this potential unless your organization bought into one of the [MDM](https://www.jamfsoftware.com/) [products](http://www.air-watch.com/) that offer DEP integration. A recent and widely published example of this kind of zero-touch deployment is [IBM's presentation](http://www.jamfsoftware.com/blog/mac-ibm-zero-to-30000-in-6-months/) at the [JAMF Nation User Conference](http://www.jamfsoftware.com/events/jamf-nation-user-conference/2015/) (JNUC) in October. Like other JAMF customers, IBM relies on the ability of JAMF's Casper Suite's JSS (JAMF Software Server) product to act as an MDM server that is compatible with Apple's DEP service and as such can perform automatic enrollment of a "new in box" Mac, push configuration profiles to it and most importantly: install JAMF's management tools on the new Mac and present their software self-service application to the user. The ability to offer this kind of self-guided device deployment with tools other than JAMF's is something I feel Mac admins should be able to offer their users so I decided to take a stab at figuring out how to make this work.

## Further analysis

Installing applications to OS X clients enrolled with an MDM server has been possible since the release of OS X 10.9 but most available documentation has focused on installing Mac App Store-hosted applications using iTunes Store IDs. This is useful for free, non-VPP apps but it doesn't work if your organization has a management toolset that is not hosted on the MAS. In addition, MAS-hosted applications are inert upon installation - they can't include any launch items, or run pre/postinstall scripts to kick off a management agent. While some MDM vendors claim to support installing admin-supplied applications, scripts or packages this functionality is usually locked away in a paid product without offering much documentation on the MDM product's capabilities. In the case of JAMF's Casper suite the ability is also limited to the vendor's own bundled management solution and does not allow you to pick your own management agent. Not surprising from a business perspective, but also not helpful for Mac admins who are only interested in a vendor's MDM product.

## Research and results

My first dive into the MDM protocol started with Apple's own MDM Protocol Reference document which I have access to through my organization's Enterprise developer account. The MDM specification is rather exhaustive and covers MDM, DEP and VPP and the web services that comprise them. It also lists all the available MDM commands available for implementation by vendors. The command that I figured would be most relevant to what I was interested in is `InstallApplication`. Since I can't link to or post content from Apple's MDM spec I will reference the MDM protocol documentation written up by [David Schuetz](http://darthnull.org/list/papers) (darthnull.org) which can be found [here](http://darthnull.org/media/papers/MDM_CommandReference.pdf). While a bit older, the reverse-engineered MDM command reference is still accurate for `InstallApplication` with some additional caveats that will be discussed later on. In its description the author notes two forms of the `InstallApplication` command: > The first form takes an iTunesStoreID as an argument, and causes the device to prompt the user for their AppleID and Password. The ID is the same as is seen in a web-based App Store page. > The other form installs a custom-developed app. You may need to install a related provisioning profile first. The ManifestURL key points to a Manifest.plist file. We are not interested in the first form so I focused on the manifest-based install method as that appeared to hold the key to installing an installer package of my choosing. In the document there is also a template outline of what such a manifest.plist file might look like:

{% gist 089e02877490153dffe2 %}

## First attempt at a manifest

Most of the keys in this template look pretty familiar for anyone who has worked with installer packages and tools like `installer` or `pkgutil` and so I proceeded to take a stab (quite a few in fact) at creating a valid `manifest.plist` file. To test the functionality I used the barebones-yet-functional PoC from the [Project iMAS MDM server](https://github.com/project-imas/mdm-server) which I have written about in a [previous post](http://wp.me/p2q5zA-8d). The project’s code incorporates just enough functionality to send the `InstallApplication` command to an MDM-enrolled client with a `ManifestURL` key to test the manifest I created. Since Munki is what my organization uses as its management tool I picked its installer package as my test case. My initial attempts at the manifest file were met with resistance and errors from the test client:

![Munki tools install error App Store][2]{:width="600px"}


```
IFJS: Package Authoring Error: error evaluating script start_selected for
  choice launchd: TypeError: null is not an object (evaluating
  'my.target.receiptForIdentifier( "com.googlecode.munki.launchd").version')
  at x-distribution:///installer-script%5B1%5D /choice%5B4%5D/@start_selected
  ==> system.env.OS_INSTALL == 1 || system.compareVersions(
  my.target.receiptForIdentifier("com.googlecode.munki.launchd").version,
  "2.0.0.1969") != 0
```


### Script SNAFU

This first error had to do with an embedded script snippet as part of the `com.googlecode.munki.launchd` component of Munki. It contains all the launch items needed by Munki to perform its duties and the script checks whether these are already installed, and skips installation if so. An error is thrown when installed to a brand new client however, resulting in a `null` comparison which the Installer framework Javascript handler (IFJS) can't deal with and causes the install process to halt. For my testing I modified the installer package and removed the `start_selected` entry that triggers the script. Since our goal is to target systems that never before had the Munki tools installed we can be certain no previous version of the launch items will be found.


```
m-test storeassetd[15274]: SoftwareMap: No app was found with bundle ID com.googlecode.munki.core to upgrade to 2.4.0.2561
m-test storedownloadd[26695]: sending status (Munki Tools): 0.000000% (0.000000)
m-test storedownloadd[26695]: DownloadManifest: removePurgeablePath: /var/folders/8m/rzq3y3n931s84vt0qdnm222h0000gn/C/com.apple.appstore/0
m-test storeassetd[15274]: SoftwareMap: No app was found with bundle ID com.googlecode.munki.core to upgrade to 2.4.0.2561
m-test storedownloadd[26695]: AuthorizationController: Non-interactive authorization succeeded
m-test storedownloadd[26695]: AuthorizationController: authorizing PKInstallClient and SUAppStoreUpdateController
m-test storeassetd[15274]: SoftwareMap: No app was found with bundle ID com.googlecode.munki.core to upgrade to 2.4.0.2561
m-test storedownloadd[26695]: DownloadOperation: Warning, unable to check disk space recovery requirements for com.googlecode.munki.core (0) because no locally cached preflight was found
m-test storeassetd[15274]: SoftwareMap: No app was found with bundle ID com.googlecode.munki.core to upgrade to 2.4.0.2561
m-test storedownloadd[26695]: ISStoreURLOperation: Starting URL operation with url=https://munkibuilds.org/munkitools2-latest.pkg / bagKey=(null)
m-test storedownloadd[26695]: -Plain Text>Plain Text>Plain Text>Plain Text>Plain Text>[ISStoreURLOperation \_runURLOperation]: \_addStandardQueryParametersForURL: https://munkibuilds.org/munkitools2-latest.pkg
m-test storedownloadd[26695]: : Opening file /var/folders/8m/rzq3y3n931s84vt0qdnm222h0000gn/C/com.apple.appstore/0/munkitools2-latest.pkg returning file descriptor 5 (0 streamed)
m-test storedownloadd[26695]: sending status (Munki Tools): 0.000000% (0.000000)
m-test storedownloadd[26695]: AssetDownloadOperation: Subtracted 0 already-downloaded bytes from required space (now requires 3241719 bytes)
m-test storedownloadd[26695]: sending status (Munki Tools): 0.000000% (-1.000000)
m-test storedownloadd[26695]: HashedDownloadProvider: Number of bytes to hash has not been set
m-test storedownloadd[26695]: ISStoreURLOperation: Chose not to retry after error: Error Domain=ISErrorDomain Code=7 "Unknown Error." UserInfo={NSLocalizedDescription=Unknown Error.}
m-test storedownloadd[26695]: AssetDownloadOperation: Asset download cancelled/failed. Will do retry #1? 0
```


### Hash pipe - no, not like that

The error I focused on in the above error log was the `HashedDownloadProvider` mention as it indicated some kind of file hashing being attempted but failing. There were various ways to figure out what was going on here but I chose to open up `storedownloadd` in my favorite debugger tool [Hopper](http://www.hopperapp.com/) to look for references to hashing. Poking around in Hopper is a bit like the [“If you give a mouse a cookie”](https://en.wikipedia.org/wiki/If_You_Give_a_Mouse_a_Cookie) stories - one thing is going to lead to another. This trip into `storedownloadd` was no exception as searching for *“hash”* led to mentions of *“md5”* (everyone’s favorite [message-digest algorithm](https://en.wikipedia.org/wiki/MD5)) and searching for *“md5”* led to finding some apparent additional keys for the manifest file: `md5-size` and `md5s`. Googling for *“MDM InstallApplication md5-size”* in turn brought me to a page, previously unknown to me, in Apple’s [OS X Deployment documentation](http://help.apple.com/deployment/osx/) portal named somewhat confusingly [Install content wirelessly](http://help.apple.com/deployment/osx/#/ior5df10f73a). This page contains a much more detailed sample template that incorporates `md5-size` and `md5` keys and also tells us that for larger files the md5 hashes must be broken up in **10 MB** chunks so that the integrity of file(s) can be checked as they make it across the network. To me this appeared to be a holdover from the iOS side of the house where slower or less relliable connections require checking a downloaded file more frequently while in flight. Either way, we need to adhere to this requirement for OS X clients. Since my testing was with the [Munki installer](https://munkibuilds.org/munkitools2-latest.pkg) which is around 3 MB this did not complicate matters and I simply added the byte size of the `munkitools2-latest.pkg` file as the `md5-size` key and used a single `md5s` entry, calculated using the `/sbin/md5` CLI tool that is included by default with OS X.

{% gist cd7da6e53d0aac20c95d %}

I now had a template that (hopefully) included the file hashes expected by `storedownloadd` and while I was at it I also added some of the additional `metadata` tags found in the Apple OS X Deployment Reference documentation. I tried the `InstallApplication` command again, and received a new set of errors. Annoying, but I did seem to have resolved the file hash problem. The new error was as follows:


```
m-test installd[460]: PackageKit: request (at PKTrustLevelNotSigned) not compatible with right(s)
system.install.apple-software, system.install.apple-software.standard-user,
system.install.app-store-software, system.install.app-store-software.standard-user,
system.install.software.mdm-provided
```


### I saw the sign

This error seemed to be related to a violation of specific system entitlements of one of the involved tools. A big clue is the `PKTrustLevelNotSigned` mention which I figured out (thanks once again to Hopper) is found in the `PackageKit` framework at `/System/Library/PrivateFrameworks/PackageKit.framework` as part of the `PKTrust stringForTrustLevel:` function. Before encountering this error I had already wondered whether installing software using the `InstallApplication` command would require a signed package and this error seemed to confirm this. Additional confirmation came from the Apple OS X Deployment documentation which states: > The package must be signed using productbuild with the MDM solution's root certificate I actually came across this last bit of information after having already concluded the package should be signed so instead of using my test MDM server's certificate I used my existing Apple developer account cert to sign the package. Either way of signing seems to work. The command I used was as follows: [bash]productsign --sign "3rd Party Mac Developer Installer:{Dev name} ({Identifier})" munkitools-2.4.0.2561.pkg munkitools-2.4.0.2561-signed.pkg[/bash] A quick update to the manifest.plist file now pointed to a signed version of the installer `munkitools2-latest-signed.pkg` with updated `md5-size` and `md5s` keys to reflect the new file:

{% gist 004878f8a92b7c482cb1 %}


## Great success!

I fired off the `InstallApplication` once more and...

![][3]

**The Munki tools package successfully installed on the client**!

As opposed to unsuccessful runs a successful one has no visual feedback and is (thankfully) completely silent. To see what happened we have to look at `/var/log/commerce.log`:


```
m-test storedownloadd[2081]: DownloadOperation: Warning, unable to check disk space recovery requirements for com.googlecode.munki.core (0) because no locally cached preflight was found
m-test storeassetd[2078]: SoftwareMap: No app was found with bundle ID com.googlecode.munki.core to upgrade to (null)
m-test storedownloadd[2081]: ISStoreURLOperation: Starting URL operation with url=https://myhost.edu/packages/testing/munkitools-2.4.0.2561-signed.pkg / bagKey=(null)
m-test storedownloadd[2081]: -[ISStoreURLOperation \_runURLOperation]: \_addStandardQueryParametersForURL: https://myhost.edu/packages/testing/munkitools-2.4.0.2561-signed.pkg
m-test storedownloadd[2081]: <hasheddownloadprovider: 0x7fcd2047acd0="">: Opening file /var/folders/8m/rzq3y3n931s84vt0qdnm222h0000gn/C/com.apple.appstore/0/munkitools-2.4.0.2561-signed.pkg returning file descriptor 5 (0 streamed)
m-test storedownloadd[2081]: sending status (Munki Bootstrap Package): 0.000000% (0.000000)
m-test storedownloadd[2081]: AssetDownloadOperation: Subtracted 0 already-downloaded bytes from required space (now requires 3249600 bytes)
m-test storedownloadd[2081]: sending status (Munki Bootstrap Package): 0.000000% (-1.000000)
m-test storedownloadd[2081]: sending status (Munki Bootstrap Package): 0.806452% (8.000000)
m-test storeassetd[2078]: SoftwareMap: No app was found with bundle ID com.googlecode.munki.core to upgrade to 999.9.9
m-test storedownloadd[2081]: [self.download metadata].bundleVersion is nil, setting it to 999.9.9
m-test storeassetd[2078]: SoftwareMap: No app was found with bundle ID com.googlecode.munki.core to upgrade to (null)
m-test storedownloadd[2081]: setPrimaryAppPath "(null)" forProductIdentifier "com.googlecode.munki.core"
m-test storedownloadd[2081]: installClient:currentState:package:progress -1:timeRemaining -1:state 0
m-test storedownloadd[2081]: PKInstallClient started
m-test storedownloadd[2081]: sending status (Munki Bootstrap Package): 0.806452% (8.000000)
m-test storedownloadd[2081]: installClientDidBegin
m-test storedownloadd[2081]: sending status (Munki Bootstrap Package): 0.806452% (8.000000)
m-test storedownloadd[2081]: installClient:currentState:package:progress 6.718204445494434:timeRemaining -1:state 3
m-test storedownloadd[2081]: sending status (Munki Bootstrap Package): 0.816129% (8.000000)
m-test storedownloadd[2081]: installClient:currentState:package:progress 30.96103425870847:timeRemaining 18.56782197380066:state 7
m-test storedownloadd[2081]: sending status (Munki Bootstrap Package): 0.854839% (6.395009)
m-test storedownloadd[2081]: installClient:currentState:package:progress 30.96103425870847:timeRemaining -1:state 1
m-test storedownloadd[2081]: sending status (Munki Bootstrap Package): 0.854839% (7.526678)
m-test storedownloadd[2081]: installClientDidFinish
m-test storedownloadd[2081]: sending status (Munki Bootstrap Package): 0.967742% (3.000000)
m-test storedownloadd[2081]: SoftwareInstallOperation: Error calling BRAppStoreDidInstallAppAtURL (file:///Applications/Managed%20Software%20Center.app/): Error Domain=BRCloudDocsErrorDomain Code=8 "(null)"
m-test storeassetd[2078]: SoftwareMap: No app was found with bundle ID com.googlecode.munki.core to upgrade to (null)
m-test storedownloadd[2081]: sending status (Munki Bootstrap Package): 0.967742% (3.000000)
m-test storeassetd[2078]: SoftwareMap: No app was found with bundle ID com.googlecode.munki.core to upgrade to (null)
m-test storedownloadd[2081]: sending status (Munki Bootstrap Package): 1.000000% (0.000000)
```


The above was very encouraging and further sanity checks confirmed that the Munki tools package was indeed successfully installed:


```
$ pkgutil --pkgs=".\*munki.\*" com.googlecode.munki.admin com.googlecode.munki.app com.googlecode.munki.core com.googlecode.munki.launchd
$ find /usr/local/munki -maxdepth 1 /usr/local/munki /usr/local/munki/conditions /usr/local/munki/launchapp /usr/local/munki/logouthelper /usr/local/munki/makecatalogs /usr/local/munki/makepkginfo /usr/local/munki/managedsoftwareupdate /usr/local/munki/manifestutil /usr/local/munki/munkiimport /usr/local/munki/munkilib /usr/local/munki/postflight /usr/local/munki/preflight /usr/local/munki/ptyexec /usr/local/munki/supervisor
$ find "/Applications/Managed Software Center.app" -maxdepth 2 /Applications/Managed Software Center.app /Applications/Managed Software Center.app/Contents /Applications/Managed Software Center.app/Contents/Info.plist /Applications/Managed Software Center.app/Contents/MacOS /Applications/Managed Software Center.app/Contents/PkgInfo /Applications/Managed Software Center.app/Contents/PlugIns /Applications/Managed Software Center.app/Contents/Resources
```


## Confirmation

I verified that Managed Software Center launched successfully as well. Due to the package install being done with a completely vanilla configuration with no manifest server setup to a vanilla install of OS X 10.11.1 the UI notified me it was unable to contact the server. While this was expected in my testing setup, admins who intend on using this method of deploying Munki tools or any other management agent in production must make sure to install additional packages that properly configure the management tool for use in their organization. Such packages could be of the payload-free variety created with tools like [munkipkg](https://github.com/munki/munki-pkg) or [The Luggage](https://github.com/unixorn/luggage).

![][4]{:width="600px"}

<a name="tldr"></a>

## Conclusion

So what have we learned? The following list contains the essential requirements for successfully triggering a package install on an MDM-enrolled client:

- An MDM service that is capable of sending the `InstallApplication` command. This could be any one of them, but for testing see [Project iMAS](https://github.com/project-imas/mdm-server)
- A signed, flat (non-bundle style) PKG file
- An HTTPS-capable web host to load an installer package from
- An HTTPS-capable web host to load a manifest file from, sent to the client as a parameter of the `InstallApplication` command, named `ManifestURL`
- A manifest file that follows standard plist formatting with a structure following the template below. Keys with specific values are required and must not be changed:

```
<plist version="1.0">
  <dict>
    <key>items</key>
    <array>
      <dict>
        <key>assets</key>
        <array>
          <dict>
            <key>kind</key>
            <string>software-package</string>
            <key>md5-size</key>
            <string/>
            <key>md5s</key>
            <array>
              <string/>
            </array>
            <key>url</key>
            <string/>
          </dict>
        </array>
        <key>metadata</key>
        <dict>
          <key>bundle-identifier</key>
          <string/>
          <key>bundle-version</key>
          <string/>
          <key>items</key>
          <array>
            <dict>
              <key>bundle-identifier</key>
              <string/>
              <key>bundle-version</key>
              <string/>
            </dict>
          </array>
          <key>kind</key>
          <string>software</string>
          <key>sizeInBytes</key>
          <string/>
          <key>title</key>
          <string/>
        </dict>
      </dict>
    </array>
  </dict>
</plist>
```

- All the above manifest keys are required. Bundle identifiers can be found using `pkgutil` to expand the install package and by checking the `Distribution` usually found within, or the `Pkginfo` file if the package is not distribution-style.
- The md5-related keys are determined by creating md5 hashes for every 10 MB (10485760 bytes) chunk of data in the installer package. If the installer package is less than 10 MB it is safe to use its exact file size as `md5-size` key and to create an md5 hash of the entire file, creating one single entry in the `md5s` array of strings
- The `bundle-identifier` entry in the top-level `metadata` dict and the one nested in the `items` array may be the same if the installer package has only one bundle ID. Some experimentation may be required as to which bundle ID to use as the top-level one in case of multiple bundle IDs

### The end. That's it. Go Home.

I'm hoping this will be helpful for the Macadmins community out there and hopefully will trigger some new developments that I haven't thought of yet. To get you started: how about an MDM-triggered "Plan C" ( [Google already took Plan B](https://github.com/google/macops/tree/master/planb)) that reinstalls and/or reconfigures your users' management agents after they fall off the management radar somehow? If a machine was enrolled with DEP there will always be a connection with your organization's MDM service and thus the ability to deploy a package, given the information I have presented here. Exciting times.

 [1]: #blah
 [2]: https://dl.dropboxusercontent.com/u/1917411/pasted_image_at_2015_11_14_02_41_am.png
 [3]: http://i.giphy.com/TlK63EWmJDBa5MjIr6g.gif
 [4]: https://dl.dropboxusercontent.com/u/1917411/msc-vanilla-mdm.png
