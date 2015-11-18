---
ID: 29
post_title: Boot Mac OS X 10.6 from a DMG
author: pepijn
post_date: 2012-02-28 17:37:52
post_excerpt: ""
layout: post
permalink: >
  http://enterprisemac.bruienne.com/2012/02/28/boot-mac-os-x-10-6-from-a-dmg/
published: true
---
Being able to boot Mac OS X from a DMG has been possible [since 10.5][1] but Apple has only recently started to use it with the Lion installation process where BaseSystem.dmg is used as the system boot volume. There are situations where this could come in handy, for example to create a Rescue partition-sans-partition for 10.5 or 10.6 systems, or to upgrade 10.5 users to Snow Leopard without needing NetBoot access. In fact, the sparseimage that is inside the NBI bundle file created by following [Rich Trouton's tutorial][2] on performing Snow Leopard upgrades with NetInstall and DeployStudio can be used without modification following the steps below. So far I have verified that both read-only compressed DMG files and read/write sparseimage files can be used. Prepping a 10.6 bootable DMG, this assumes the source DMG contains Mac OS X 10.6: 
1.  Create a folder to store the required files in **/private/BootDMG** to secure its contents. All files and folders including the root folder must have root ownership.
2.  Inside the BootDMG root folder create** Contents/Resources/Files** and **Contents/Resources/Image**, using mkdir: 
    <pre>sudo mkdir -p /private/BootDMG/Contents/Resources/Files
sudo mkdir -p /private/BootDMG/Contents/Resources/Image</pre>

3.  Copy the DMG of a Snow Leopard-based bootable DMG such as the Snow Leopard Install DVD or a DeployStudio Runtime boot volume to **/private/BootDMG/Contents/Resources/Image/YOURDMG.dmg**: 
    <pre>sudo cp /Users/admin/Some/Folder/YOURDMG.dmg /private/BootDMG/Contents/Resources/Image/</pre>

4.  Mount **/private/BootDMG/Contents/Resources/Image/YOURDMG.dmg**, using Disk Utility or hdiutil: 
    <pre>sudo hdiutil attach -nobrowse -noverify /private/BootDMG/Contents/Resources/Image/YOURDMG.dmg</pre>

5.  From the mounted DMG copy the following files to **/private/BootDMG/Contents/Resources/Files **and unmount the DMG: 
    <pre>sudo cp /Volumes/YOURDMG/mach_kernel /private/BootDMG/Contents/Resources/Files/
sudo cp /Volumes/YOURDMG/usr/standalone/i386/boot.efi /private/BootDMG/Contents/Resources/Files/
<strong></strong>sudo cp /Volumes/YOURDMG/System/Library/Caches/com.apple.kext.caches/Startup/Extensions.mkext /private/BootDMG/Contents/Resources/Files/
sudo umount /Volumes/YOURDMG</pre>

6.  *Note: if Extensions.mkext does not exist on the source DMG, you can simply create an empty file by the same name using **touch** or by saving an empty text file with tools such as **vi**, **TextMate** or **TextEdit**.*
7.  Use **defaults** to write required settings to a new **com.apple.Boot.plist **file: 
    <pre>sudo defaults write /private/BootDMG/Contents/Resources/Files/com.apple.Boot Kernel /private/BootDMG/Contents/Resources/Files/mach_kernel
sudo defaults write /private/BootDMG/Contents/Resources/Files/com.apple.Boot 'Kernel Flags' "rp=file://private/BootDMG/Contents/Resources/Image/YOURDMG.dmg"
sudo defaults write /private/BootDMG/Contents/Resources/Files/com.apple.Boot 'MKext Cache' /private/BootDMG/Contents/Resources/Files/Extensions.mkext</pre>

8.  Finally, use the **bless **command to set the Mac to boot from the DMG: 
    <pre>sudo bless --folder "/private/BootDMG/Contents/Resources/Files" --file "/private/BootDMG/Contents/Resources/Files/boot.efi" --setBoot --options "config=\private\BootDMG\Contents\Resources\Files\com.apple.Boot"</pre>

9.  You can now reboot the Mac, which will boot using the DMG you prepped. Given the steps above it shouldn't be too hard to create a deployment-ready package that drops a prepped bootable DMG and accompanying files somewhere on a user's machine and configures it for booting either immediately or later through a simple user-facing GUI tool to select either regular HD boot or DMG boot.

 [1]: http://www.opensource.apple.com/source/xnu/xnu-1228.15.4/bsd/kern/imageboot.c
 [2]: http://derflounder.wordpress.com/2011/05/21/upgrading-an-intel-mac-from-10-5-x-to-10-6-x-using-netinstall-and-deploystudio/