---
title: Enable Google two-factor authentication for SSH connections on OS X
author: pepijn
layout: post
permalink: /2015/01/07/enable-google-two-factor-authentication-for-ssh-connections-on-os-x/
categories:
  - Configuration
  - Security
tags:
  - 2fa
  - os x
  - security
  - ssh
---
**Note:** this post was updated with additional security concerns regarding Git and the method of installing the required tools needed for compiling the PAM module. Thanks to [@marczak](https://twitter.com/marczak) and [@Magervalp](https://twitter.com/magervalp) for the feedback.

[Two-factor authentication](http://en.wikipedia.org/wiki/Two_factor_authentication) (2FA) is fairly mainstream these days, which is a good thing. It would be nifty if Mac Admins could add the increased security 2FA offers to remote (SSH) logins on OS X. There are existing commercial solutions like [Duo Security](https://www.duosecurity.com/) (a local Ann Arbor business I heartily endorse) that offer tools to accomplish this but if you are already using Google Authenticator for other services it may make sense to use that instead. As part of the [Google Authenticator open source code](https://github.com/google/google-authenticator) Google provides a [PAM](https://www.freebsd.org/doc/en/articles/pam/) module which, with some effort, can be compiled and configured for use with OS X's own PAM system. In order to compile the GA PAM module the Xcode CLI tools are required as well as automake and autoconf. The easiest way to install the latter two is either through [Homebrew](http://brew.sh/), a popular OS X package manager or using ready-made PKG installers from the [Rudix](http://rudix.org/) project.<!--more-->

## Prerequisites

In order to prepare the required tools follow these steps. First, we'll need the Xcode command line tools:

<pre class="brush: bash; title: ; notranslate" title="">$ xcode-select --install
xcode-select: note: install requested for command line developer tools</pre>

This will prompt the user to install the command line developer tools.

[<img src="http://enterprisemac.bruienne.com/wp-content/uploads/2015/01/XcodeCLItools.png" alt="XcodeCLItools" width="500" height="267" class="alignnone size-full wp-image-358" />][1]  
Before we continue a quick note regarding the Git client that ships with OS X - this is a post about security after all. A few weeks ago it [was announced](https://github.com/blog/1938-vulnerability-announced-update-your-git-clients) that all shipping Git clients had a serious security issue on case-insensitive filesystems that could allow for malicious repositories to overwrite the .git/config file and cause arbitrary command execution. Apple shipped a patch for the issue with [Xcode 6.2 beta 3](http://support.apple.com/en-us/HT204147) which I would strongly suggest downloading from [Apple's Developer site](https://developer.apple.com/devcenter/mac/index.action) and installing.

All that is left now is to install automake and autoconf which are the only required tools that do not ship with Xcode. As noted by one commenter it was necessary for him to also install libtool. I've added it to the list for reference, it may or may not be needed for everyone but won't hurt to install alongside the other two. If you are a current Homebrew user all you should have to do is:

<pre class="brush: bash; title: ; notranslate" title="">$ brew install autoconf automake libtool</pre>

Or, if you use Rudix as your package manager it should be as simple as:

<pre class="brush: bash; title: ; notranslate" title="">$ sudo rudix install automake autoconf libtool</pre>

If you would like to use either Homebrew or Rudix package managers but don't have them installed yet you must do so first. As [noted by Ed Marczak](https://twitter.com/marczak/status/552829945899388928) the recommended installation method for both Homebrew and Rudix involves directly piping and executing code from the Internet, in good faith. I agree with him that this is not necessarily a habit you want to get too comfortable with. It takes as little as one line of code inserted either accidentally or maliciously to cause data loss, install malware and so on. I'm not implying that either of these tools will, but other less scrupulous persons may take advantage of the trust you previously put into legitimate install processes. I recommend that you examine the code executed by any pipe-curl-to-interpreter install like Homebrew or Rudix beforehand. Both [Homebrew](https://github.com/Homebrew/homebrew) and [Rudix](https://github.com/rudix-mac/rudix) have Github repositories.

If you just want to install the required tools without the added weight of a packaging tool you can opt to install the self-contained PKG installers for automake, autoconf and libtool provided by the [Rudix project](http://rudix.org/packages.html). If you decide to use the Rudix PKG installers I recommend that you examine them using something like [Pacifist](https://www.charlessoft.com/) prior to installation. Pacifist is by far the best OS X package inspection tool and you should consider paying for a license. On with the show, shall we?

**Installing Homebrew, followed by an installation of automake, autconf and libtool:**

<pre class="brush: bash; title: ; notranslate" title="">$ ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
&lt;snip&gt;
$ brew install autoconf automake libtool</pre>

**Installing Rudix package manager, followed by an installation of automake, autoconf and libtool:**

<pre class="brush: bash; title: ; notranslate" title="">$ curl -s https://raw.githubusercontent.com/rudix-mac/rpm/2014.10/rudix.py | sudo python - install rudix
&lt;snip&gt;
$sudo rudix install automake autoconf libtool</pre>

**Installing automake and autoconf using Rudix PKG installers:**

[Download the automake PKG installer](http://rudix.org/packages/automake.html) (10.6-10.10)  
[Download the autoconf PKG installer](http://rudix.org/packages/autoconf.html) (10.6-10.10)  
[Download the libtool PKG installer](http://rudix.org/packages/libtool.html) (10.6-10.10)

## Installation and configuration

With the prerequisites out of the way, compiling the PAM module should now go smoothly:

<pre class="brush: bash; title: ; notranslate" title="">$ git clone https://github.com/google/google-authenticator.git
$ cd google-authenticator/libpam
$ autoreconf -ivf
&lt;snip&gt;
$ touch AUTHORS NEWS README ChangeLog
$ automake --add-missing
&lt;snip&gt;
$ ./configure
&lt;snip&gt;
$ sudo make install
&lt;snip&gt;
$ sudo cp /usr/local/lib/security/pam_google_authenticator.so /usr/lib/pam/
$ sudo vi /etc/pam.d/sshd</pre>

The last command above opens up the SSH daemon PAM configuration file in vim, where we will add the following line:

<pre class="brush: plain; title: ; notranslate" title="">auth required pam_google_authenticator.so nullok</pre>

Adding the line makes the Google Authenticator PAM module required for all authentication requests. This means that in order to perform a successful SSH login the remote user must provide both their account password and a one-time code generated by Google Authenticator or other compatible 2FA app. Note the 'nullok' option which will cause the Google Authenticator module to be skipped for users who have not yet been setup using the google-authenticator tool, which we will discuss next.

## Setting up users for two-factor authentication

As part of the 'make install' process an executable was installed to `/usr/local/bin/google-authenticator` which is used to set up a user for GA authentication. Running `google-authenticator` without any options will prompt the user to select the type of token to create (HOTP or TOTP) and a few other additional security options. Running it with the -h flag will display the full usage:

<pre class="brush: plain; title: ; notranslate" title="">$ google-authenticator -h
google-authenticator [&lt;options&gt;]
 -h, --help               Print this message
 -c, --counter-based      Set up counter-based (HOTP) verification
 -t, --time-based         Set up time-based (TOTP) verification
 -d, --disallow-reuse     Disallow reuse of previously used TOTP tokens
 -D, --allow-reuse        Allow reuse of previously used TOTP tokens
 -f, --force              Write file without first confirming with user
 -l, --label=&lt;label&gt;      Override the default label in "otpauth://" URL
 -i, --issuer=&lt;issuer&gt;    Override the default issuer in "otpauth://" URL
 -q, --quiet              Quiet mode
 -Q, --qr-mode={NONE,ANSI,UTF8}
 -r, --rate-limit=N       Limit logins to N per every M seconds
 -R, --rate-time=M        Limit logins to N per every M seconds
 -u, --no-rate-limit      Disable rate-limiting
 -s, --secret=&lt;file&gt;      Specify a non-standard file location
 -w, --window-size=W      Set window of concurrently valid codes
 -W, --minimal-window     Disable window of concurrently valid codes</pre>

We will use option flags to perform a non-interactive configuration, the output of which is shown below. The options we're using are -t (create a TOTP token, the more secure option), -d (disallow reuse), -r 3 (number of logins per time window), -R 30 (duration of time window), -w 90 (token validity window) and -f (force writing configuration to ~/.google_authenticator).

<pre class="brush: bash; title: ; notranslate" title="">$ google-authenticator  -t -d -r 1 -R 30 -w 90 -f
https://www.google.com/chart?chs=200x200&#038;chld=M|0&#038;cht=qr&#038;chl=otpauth://totp/demo@myhost%3Fsecret%3DSECRET_KEY%26issuer%3Dmyhost

Your new secret key is: SECRET_KEY
Your verification code is VERIFICATION_CODE
Your emergency scratch codes are:
51643415
73317699
44867697
70785497
91891533</pre>

The output contains a few important bits of data. The first bit is the google.com URL which is a link to a QR code used to add the token for your user and host to Google Authenticator or other compatible app. Open the URL by command-clicking it in Terminal.app which should open your default web browser and show a QR code, ready to be added to a 2FA app. Instructions on how to add a token to Google Authenticator or Authy using QR codes are here:

[Adding a new token to Google Authenticator](https://support.google.com/accounts/answer/1066447?hl=en)  
[Adding a new token to Authy](https://www.authy.com/how-use-authy-google-authenticator)

The second bit (or bits) of info are the five emergency scratch codes which can be used as one-time emergency codes in case you lose access to your 2FA application. It is a good idea to store these emergency codes someplace safe.

With the setup of the Google Authenticator PAM module and configuring of our 2FA app out of the way we can now attempt a Google Authenticator 2FA-enabled SSH login:

<pre class="brush: bash; title: ; notranslate" title="">$ ssh demo@localhost

Password:
Verification code:
Last login: Tue Jan  6 23:05:58 2015 from localhost
myhost:~ demo$</pre>

Success! As seen above the SSH login process first prompts for the regular user password and then prompts for a verification code. The six-digit code is retrieved from the 2FA app we added our token to and once entered at the prompt it is accepted and login is complete. Huzzah!

Even though this post describes how to enable Google Authenticator 2FA for SSH on OS X it should work much the same for non-OS X hosts. The [README](https://github.com/google/google-authenticator/blob/master/libpam/README) found on the Github repository contains further detailed information on configuration as well.

 [1]: http://enterprisemac.bruienne.com/wp-content/uploads/2015/01/XcodeCLItools.png
