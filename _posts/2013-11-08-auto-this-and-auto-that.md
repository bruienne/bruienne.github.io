---
ID: 232
post_title: Auto-this and Auto-that
author: pepijn
post_date: 2013-11-08 07:00:20
post_excerpt: ""
layout: post
permalink: >
  http://enterprisemac.bruienne.com/2013/11/08/auto-this-and-auto-that/
published: true
---
Inspired by recent <a href="https://github.com/autopkg/autopkg" target="_blank">Auto</a>-<a href="https://github.com/MagerValp/AutoDMG" target="_blank">events</a> I decided I was tired of having to manually roll hardware-specific NetBoot images, such as we've had to do recently for both Mountain Lion and Mavericks releases. It took me a while to spelunk into the innards of System Image Utility and its related frameworks and tools but I feel that I was able to write up something half-decent for it. With that said, please take a look at AutoNBI.py, the latest member of the Auto-family. There's no fancy GUI, but that's the point here - integration into your workflow. There's some first version caveats such as the NBI modification method being pretty basic in that it currently will replace or add only one folder since that is what I needed to be able to do for my needs. I have some additional code in the works that will let AutoNBI ingest a plist file with more complex add and remove configurations, but for now this'll have to do. Try it out, let me know what you think. Link to the Bitbucket project page is <a href="https://bitbucket.org/bruienne/autonbi" target="_blank">here</a> and the Readme follows below: 
# **AutoNBI.py** {#markdown-header-autonbipy}

**A tool to automate (or not) the building and customization of Apple NetBoot NBI bundles.** 
## **Requirements**: {#markdown-header-requirements}

*   **OS X 10.9 Mavericks** - This tool relies on parts of the *SIUFoundation* Framework which is part of System Image Utility, found in*/System/Library/CoreServices* in Mavericks.
*   **Munki tools** installed at */usr/local/munki* - needed for FoundationPlist.

## **Thanks to:** {#markdown-header-thanks-to}

*   Greg Neagle for overall inspiration and code snippets (COSXIP)
*   Per Olofsson for the awesome AutoDMG which inspired this tool
*   Tim Sutton for further encouragement and feedback on early versions This tool aids in the creation of Apple NetBoot Image (NBI) bundles. It can run either in interactive mode by passing it a folder, installer application or DMG or automatically, integrated into a larger workflow. 

## **Command line options:** {#markdown-header-command-line-options}

*   **[--source][-s]** The valid path to a source of one of the following types:
*   A folder (such as /Applications) which will be searched for one or more valid install sources
*   An OS X installer application (e.g. "Install OS X Mavericks.app")
*   An InstallESD.dmg file
*   **[--destination][-d]** The valid path to a dedicated build root folder: The build root is where the resulting NBI bundle and temporary build files are written. If the optional --folder arguments is given an identically named folder must be placed in the build root: 

*./AutoNBI <arguments> -d /Users/admin/BuildRoot --folder Packages* -> Causes AutoNBI to look for **/Users/admin/BuildRoot/Packages** 
*   **[--name][-n]** The name of the NBI bundle, without .nbi extension
*   **[--folder]** *Optional* - The name of a folder to be copied onto NetInstall.dmg. If the folder already exists, it will be overwritten. This allows for the customization of a standard NetInstall image by providing a custom rc.imaging and other required files, such as a custom Runtime executable. For reference, see the DeployStudio Runtime NBI.
*   **[--auto][-a]** *Optional* - Enable automated run. The user will not be prompted for input and the application will attempt to create a valid NBI. If the input source path results in more than one possible installer source the application will stop. If more than one possible installer source is found in interactive mode the user will be presented with a list of possible InstallerESD.dmg choices and asked to pick one.

## **Examples:** {#markdown-header-examples} To invoke AutoNBI in interactive mode: 

*sudo ./AutoNBI -s /Applications -d /Users/admin/BuildRoot -n Mavericks* To invoke AutoNBI in automatic mode: *sudo ./AutoNBI -s ~/InstallESD.dmg -d /Users/admin/BuildRoot -n Mavericks -a* To replace "Packages" on the NBI boot volume with a custom version: *sudo ./AutoNBI -s ~/InstallESD.dmg -d ~/BuildRoot -n Mavericks -f Packages -a*