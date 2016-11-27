---
title: Signing installer packages without Apple tools
author: pepijn
layout: post
permalink: /2016/09/21/pkgsign-of-the-times/
categories:
  - Deployment
  - Automation
  - DevOps
tags:
  - deployment
  - installer
  - management
  - automation
  - docker
published: false
---

## Introduction
While reading through [updated Apple Dev documentation about the current state of codesigning](https://developer.apple.com/library/content/technotes/tn2206/_index.html) in preparation for the Sierra release I returned to an aspect of codesigning I had been meaning to figure out further in order to possibly find a more generic and portable way to handle this. Signing macOS (and iOS) applications is rather involved and includes modifying binaries and creating code resource files best done with Apple's native tools and is nothing easily duplicated with generic tools. However the signing of packages is less complicated and has an actual practical application for me and others.
## Packages redux
As a refresher, since bundle-style packages and PackageMaker are [no longer supported](https://developer.apple.com/library/content/technotes/tn2206/_index.html#//apple_ref/doc/uid/DTS40007919-CH1-TNTAG405) all package installers must be distributed as flat-file archives. Apple adopted the [XAR](https://en.wikipedia.org/wiki/Xar_(archiver) format for this new flat-file package format, which was originally part of the OpenDarwin project. Version `1.8dev` of `xar` has shipped with the last few major OS versions, including macOS 10.12 Sierra. A complete description of the format can be found [here](https://github.com/mackyle/xar/wiki/xarformat). Even though Apple installer packages bear the `.pkg` extension they are regular XAR archives as seen when running the `file` command against a PKG file. The `file` POSIX command uses file magic numbers in a file's header to determine its file type. For PKG files the format is as follows:

```
$ file SomeApp.pkg
/Users/admin/Documents/SomeApp.pkg: xar archive - version 1
```
While Apple ships the dedicated `pkgutil` tool for working with PKG files most of that tool's functionality relates to receipt database queries and modifications with just two commands available to directly operate on the PKG archive itself: `--expand` and `--flatten`. As one can guess the `--expand` command will take a source PKG file and expand its contents into a target directory:
```
$ pkgutil --expand SomeApp.pkg SomeApp
$ ls SomeApp
Bom PackageInfo Payload Scripts
```
We can achieve the exact same results by running the `xar` command as well:

```
$ xar -d -f SomeApp.pkg
$ ls SomeApp
Bom PackageInfo Payload Scripts
```

Regardless of the command used to decompress the package we can see that the contents of the outer XAR wrapper are the "classic" contents of a bundle-style format:

`Bom` - The Bill Of Materials of the payload created by `mkbom`, a binary format

`PackageInfo` - An XML file that stores file names and version information of the Payload

`Payload` - The gzip-compressed payload as installed by the PKG

`Scripts` - A standard directory containing scripts that run either before or after payload installation

If we were to inspect the codesigning status of a package using Apple's included `pkgutil` tool we would see that it is signed:

```
$ /usr/sbin/pkgutil --check-signature SomeApp.pkg
Package "SomeApp.pkg":
   Status: signed by a developer certificate issued by Apple
   Certificate Chain:
    1. 3rd Party Mac Developer Installer: John Doe (WJ1C234G56)
       SHA1 fingerprint: 11 22 33 44 55 66 77 88 99 AA BB CC DD EE FF 12 34 56 78 90
       -----------------------------------------------------------------------------
    2. Apple Worldwide Developer Relations Certification Authority
       SHA1 fingerprint: FF 67 97 79 3A 3C D7 98 DC 5B 2A BE F5 6F 73 ED C9 F8 3A 64
       -----------------------------------------------------------------------------
    3. Apple Root CA
       SHA1 fingerprint: 61 1E 5B 66 2C 59 3A 08 FF 58 D1 4A E2 24 52 D1 98 DF 6C 60
```

By listing the entire certificate chain we can see that starting at the developer's leaf certificate the signing can be verified via the intermediary Apple Worldwide Developer Relations CA through to Apple's Root CA - the package is properly signed. Even more detailed information regarding the signing status can be obtained using helpful tools like [Pacifist](https://www.charlessoft.com) or the equally excellent [Suspicious Package](http://www.mothersruin.com/software/SuspiciousPackage/).

Now if we were to recompress the decompressed payload back into a XAR/PKG archive:

```
$ pkgutil --flatten SomeApp SomeApp.pkg
# OR
$ xar -c -f SomeApp.pkg SomeApp
```

And we check the signing status of the recompressed PKG:

```
$ /usr/sbin/pkgutil --check-signature SomeApp.pkg
Status: no signature
```
The package that was signed mere minutes ago is no longer signed after simply decompressing and compressing it again.

That brings up a fundamental question about signed packages: where is the signing information stored? Looking at the contents of the PKG payload files shows no mention of certificates, checksums or anything else that would indicate that the PKG was ever signed. More confusingly, after simply uncompressing and recompressing it the package is no longer signed.

The answer is that the signing information for a PKG is not stored inside the XAR archive's payload.  Since we'd probably not want to open or process the contents of the PKG if the signing information doesn't check out it's a good thing that this is not the case. Instead the signing information is in the XAR table of contents (or TOC) as described in detail [here](https://github.com/mackyle/xar/wiki/xarformat#The_Table_of_Contents). The XAR TOC is a UTF-8 encoded XML document that contains metadata about the payload. Apple expanded the available TOC keys for codesigning purposes and added the top-level `<signature>` key together with sub-keys that allow storing required things such as base64-encoded certificates inside the `X509Data` key. When we look at the TOC of a signed package we see the following:
```
$ xar --dump-toc=toc.xml -f SomeApp.pkg
$ head -15 toc.xml
<?xml version="1.0" encoding="UTF-8"?>
<xar>
 <toc>
  <checksum style="sha1">
   <size>20</size>
   <offset>0</offset>
  </checksum>
  <creation-time>2016-09-10T12:27:19</creation-time>
  <signature style="RSA">
   <offset>20</offset>
   <size>256</size>
   <KeyInfo xmlns="http://www.w3.org/2000/09/xmldsig#">
    <X509Data>
     <X509Certificate>MIIFgzCCBGugAwIBAgIIN/KUb03/XHkwDQYJKoZIhvcNAQELBQAweTEtMCsGA1UEAwwkRGV2
ZWxvcGVyIElEIENlcnRpZmljYXRpb24gQXV0aG9yaXR5MSYwJAYDVQQLDB1BcHBsZSBDZXJ0
```
