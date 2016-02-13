---
title: 'MDM-azing - setting up your own MDM server'
author: pepijn
layout: post
permalink: /2015/06/06/mdm-azing-setting-up-your-own-mdm-server/
categories:
  - Deployment
  - Management
  - Python
---
Pardon the pun, but I've been meaning to use that [Shamen](https://www.youtube.com/watch?v=hxoe-xUYL8g) reference ever since MDM became A Thing.

It is not for a lack of Mobile Device Management solutions that I wanted to figure out the process of setting up my own MDM server of course. Quite the opposite as there are [many](http://www.air-watch.com/solutions/mobile-device-management) [vendors](http://www.jamfsoftware.com/products/casper-suite/) [out](http://www.bushel.com/) [there] (https://meraki.cisco.com/solutions/mobile-device-management) [offering](http://www.maas360.com/products/mobile-device-management/) [MDM](https://www.mobileiron.com/en/solutions/mobile-device-management-mdm) [solutions](http://www.symantec.com/mobile-device-management/) with varying levels of customer satisfaction. It's fair to say that often MDM solutions tend to nudge over to the far ends of the spectrum capped by "Overly complicated" on one end and "Checkbox feature" on the other. Barring a few free ones most are also pricey, further lowering one's sense of getting bang for one's buck. Another issue can be integration with existing systems, which often leads to companies deciding to buy into complete solutions from one vendor. Not a perfect situation by any stretch of the imagination for those of us who have perfectly functional management tools just looking to enhancement their toolset with the benefits of Apple's OS X MDM integration. There have been previous flurries of interest and activity in the Mac Admin community around creating a true OSS MDM solution. These attempts mostly fizzled due to uncertainty about the exact process of creating an MDM service and lack of sources of information. After some asking around it was determined that Apple keeps certain key bits of information behind the iOS Enterprise Developer paywall, such as the Mobile Device Management Protocol Reference document. Even more importantly the ability to sign the required MDM CSR for such a service is also only available to organizations subscribing to the same $300/year program.<!--more-->

One particularly interesting project has been part of the MITRE Corporation's [Project iMAS](https://github.com/project-imas/about), or iOS Mobile Application Security. Based on a 2011 presentation at the Black Hat conference by the Intrepidus Group named ["Inside Apple MDM"](https://github.com/intrepidusgroup/imdmtools/raw/master/Presentations/InsideAppleMDM_BlackHatUSA_2011.pdf) and the sample code it contained, the Project iMAS folks have quietly worked out a very useable set of [Python code](https://github.com/project-imas/mdm-server) that offers a reference MDM server ready to be built upon. This leaves the small matter of actually being able to use the code with Apple's blessing. The folks at Project iMAS helpfully provide detailed [Setup steps](https://github.com/project-imas/mdm-server#setup) for preparing the needed certificates but like [other sources](http://www.blueboxmoon.com/wordpress/?p=877) its information on how to actually get Apple's cooperation is pithy and not up to date with how Apple's iOS Developer portal works in 2015. With that said, these are the steps I followed to successfully stand up a basic MDM server that works with APNS and Apple's MDM support:

  1. Enroll in Apple's **iOS Enterprise Developer** program or have yourself added to an existing subscription
  2. Contact Apple Developer support and ask for the **"MDM CSR"** option to be enabled in the **Certificates** section
  3. The Apple representative may tell you that in order to enable the option they have to get the approval of an iOS Enterprise account **Agent** or **Admin**. It's a good idea to find out who this is (if not you) and give them a heads up about what you're asking Apple for and that you'd like them to approve the request.
  4. Once the account Agent or Admin has approved the request a new option will be available in the ["Certificates, Identifiers & Profiles"](https://developer.apple.com/account/ios/certificate/certificateCreate.action) section of the **Account** page. As far as I have been able to determine this option is only available to an **Admin** user so if you are not an Admin for the account you'll need the cooperation of someone who is in order to perform the CSR-signing step.
  5. Using **Keychain Access** (or openssl) generate a basic **Certificate Signing Request** (CSR) as outlined in the Project iMAS [Setup instructions](https://github.com/project-imas/mdm-server#setup). Use the **email address** of a developer account that is part of the same iOS Enterprise account that you had the MDM CSR option enabled on. Use a Common Name of your choice (your organization's name for example) and select to save it to disk. Name the CSR file *"mdmvendor.csr"*.
  6. Unless you are an Admin on the iOS Enterprise account you need to send the "mdmvendor.csr" file to an Admin user and have them select the **"MDM CSR"** option in Certificates, Identifiers & Profiles **(Fig. 1)**
  7. Submitting the CSR is very similar to how the Apple Push Certificates Portal works **(Fig. 2)**. The Admin selects the "mdmvendor.csr" file for upload and goes through the submission steps. At the end of the process a signed certificate file is ready for download. The Admin user should save this file to disk and name it *"mdmvendor.cer"*. The Admin needs to send the "mdmvendor.cer" file back to you either by email or through some other method. You will not need their assistance after this.
  8. Before moving on, clone the mdm-server project to the system you are going to do the rest of the certificate preparation on as it contains a number of tools that simplify the process: <pre class="brush: bash; title: ; notranslate" title="">$ git clone https://github.com/project-imas/mdm-server.git
$ git submodule init
$ git submodule update</pre>

  9. You can now continue to follow the steps starting at *"3. Export MDM private key"* in the Project iMAS README. One clarification is needed for this step: make sure to import the "mdmvendor.cer" file into the **Login** keychain in order to enable the saving of the private key as described.
 10. Once all the certificate preparations are completed the [Server Setup](https://github.com/project-imas/mdm-server#server-setup) section is next. You are free to go through these manual setup steps but in order to get up and running a bit faster I have created a [Docker image](https://registry.hub.docker.com/u/bruienne/mdm-server/) that wraps it all up into an image that is ready to go. To start a container that has access to the prerequisite certificates and keys that we created we can bind-mount our local *"mdm-server/server"* directory in which they are located, using the `-v source:destination` flag: <pre class="brush: bash; title: ; notranslate" title="">$ docker pull bruienne/mdm-server
Pulling repository bruienne/mdm-server
b7de3133ff98: Pulling dependent layers
5cc9e91966f7: Pulling fs layer
511136ea3c5a: Download complete

$ docker run -d -v /Users/username/path/to/mdm-server/server:/mdm-server/server -p 0.0.0.0:8080:8080 bruienne/mdm-server
Starting Server
https://0.0.0.0:8080/
Can't find MyApp.mobileprovision in current directory.
Need both MyApp.ipa and Manifest.plist to enable InstallCustomApp.
LOADED PICKLE
</pre>

 11. You should now be able to reach the server on the IP of the Docker host: **https://<IP\_OR\_HOSTNAME>:8080/** On an iOS device you'll see links to download the CA certificate and the enrollment profile **(Fig. 3)**

<figure id="attachment_515" class="thumbnail wp-caption alignleft" style="width: 694px"><img src="http://enterprisemac.bruienne.com/static/MDM-CSR.png" alt="Figure 1" width="684" height="222" class="size-full wp-image-515" /><figcaption class="caption wp-caption-text">Figure 1 - MDM CSR option</figcaption></figure>  
<figure id="attachment_523" class="thumbnail wp-caption alignleft" style="width: 610px"><img src="http://enterprisemac.bruienne.com/static/PushPortal.png" alt="Figure 2" width="600" height="423" class="size-full wp-image-523" /><figcaption class="caption wp-caption-text">Figure 2 - Upload CSR</figcaption></figure>  
<figure class="thumbnail wp-caption alignleft" style="width: 610px"><img src="https://raw.githubusercontent.com/project-imas/mdm-server/master/images/deviceEnroll.jpg" alt="Figure 3" width="600" class="size-full" /><figcaption class="caption wp-caption-text">Figure 3 - Open Source MDM Homepage</figcaption></figure>

This should get the basic server going. Hopefully this will be helpful in getting some more knowledge and insight into the Apple MDM process that results in simple MDM server code that can be integrated in larger management frameworks. An example of possible integration is [OpenMDM](https://github.com/OpenMDM/OpenMDM), another OSS MDM project that focuses on creating management profiles through a web interface. It's not hard to imagine combining the two to create a full solution that offers the same features Apple's Profile Manager does - without the OS X Server requirement (and hopefully also without some of its bugs). In a follow-up post I'll talk about a few simple modifications to the server code that allow for the enrollment of OS X Yosemite and Mavericks devices as well. Since OS X management is not part of the focus of the iMAS project I have been working on adding some of that but more community input and hacking will definitely be welcomed.
