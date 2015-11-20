---
title: Creating a signed Java Deployment Rule Set with Windows Server CA
author: pepijn
layout: post
permalink: /2014/03/04/creating-a-signed-java-deployment-rule-set-with-windows-server-ca/
categories:
  - Deployment
  - Java
tags:
  - java
  - os x
  - security
---
## **Introduction**

With the release of Oracle's Java 7 Update 51 came heightened security measures that affect [unsigned and self-signed](https://www.java.com/en/download/help/java_blocked.xml) Java applets. At its standard "High" security setting the Java web plugin and standalone JVM will refuse to run unsigned or self-signed applets unless they have been explicitly added to a [user-level whitelist](https://blogs.oracle.com/java-platform-group/entry/upcoming_exception_site_list_in) which is a newly added security feature in Java 7 Update 51.<!--more-->  
To allow large organizations to better manage security for their users Oracle previously introduced the [Deployment Rule Set feature](https://blogs.oracle.com/java-platform-group/entry/introducing_deployment_rule_sets) in Java 7 Update 40. The Deployment Rule Set consists of a single signed JAR file named "DeploymentRuleSet.jar" deployed in the Java system path "/Library/Application Support/Oracle/Java/Deployment". Given the new security measures in Java 7 Update 51 it is a good time to start using a Deployment Rule Set since it provides:

  * The ability to use wildcard exception rules, unlike the user exception site list (https://mydomain.myorg.com/*)
  * No Java security warnings when accessing a whitelisted Java applet, unlike the user exception site list
  * Easy system-wide installation and updating of the ruleset

This post deals with a common scenario for Mac admins: you're in an established Windows Server Active Directory environment that offers Certificate Authority services. Clients may already have your domain's CA in their trusted cert store so extending this to sign a Java Deployment Rule Set JAR may make sense. The process of deploying the DeploymentRuleSet.jar file is outside the scope of this article although I did include a postinstall script as an addendum to assist with the installation of the signed certificate chain that this article will help you create. With that said, let's get underway.

## **The process**

### Generate a new keystore and key

To perform the various key requests and code signing operations, a way to keep it simple is to create a fresh Java keystore file using the same password as the Java default JKS password. You're free to play around with the -keyalg and -keysize settings as needed.

<pre class="brush: bash; title: ; notranslate" title="">$ keytool -genkey -alias mykey -keyalg RSA -keysize 2048 -keystore keystore.jks -storepass changeit</pre>



### Generate a Certificate Signing Request

In order to verify and sign the code signing certificate the Windows CA is going to need a certificate signing request (CSR) to process. This command creates one based on the key we generated in the previous step.

<pre class="brush: bash; title: ; notranslate" title="">$ keytool -certreq -alias mykey -file csr.csr -keystore keystore.jks -storepass changeit</pre>



### Extract the private key from the keystore

To submit a signing request, we'll need the private key as well as the public one. The easiest way to get the private key out of an existing keystore is to import the keystore into a newly-created keystore, selecting only the key we are interested in and storing it as PKCS#12. The Windows tool we'll use later can process PKCS#12 keys so we don't need to do any further conversion.

<pre class="brush: bash; title: ; notranslate" title="">$ keytool -v -importkeystore -srckeystore keystore.jks -srcalias mykey -destkeystore myp12file.p12 -deststoretype PKCS12</pre>



### Rename private key file

Windows Server likes certain things to be a certain way and dealing with certificates is no different, so we must rename our PKCS#12 file to have a .pfx extension to allow certreq.exe to play nice.

<pre class="brush: bash; title: ; notranslate" title="">$ mv myp12file.p12 myp12file.pfx</pre>



### Sign the CSR using Windows Server

The files you will need to process the signing request are "mykey.csr", "mykey.cer" and "myp12file.pfx". Sign your generated signing request using a user account that has **Read** and **Enroll** rights to a template configured for code signing on the Windows Server CA. In the example here we're using a template named "MyCodeSigningTemplate". See here for more info on how to create a code signing Certificate Template with Windows Server: http://technet.microsoft.com/en-us/library/cc730826(v=ws.10).aspx

Allow certreq.exe to overwrite mykey.cer and mykey.csr when prompted by the "certreq" command.

<pre class="brush: bash; title: ; notranslate" title="">C:\Windows\system32&amp;gt;certreq -submit -attrib &amp;quot;CertificateTemplate:MyCodeSigningTemplate&amp;quot; mykey.csr mykey.cer myp12file.pfx</pre>



### Import signed key and CA into keystore

We need to add the signed key and signing CA (and any intermediate CA certs) back into our keystore so we can use it for code signing.

<pre class="brush: bash; title: ; notranslate" title="">$ keytool -importcert -keystore keystore.jks -file ca-certificate.pem -alias CARoot -storepass changeit
...
Certificate was added to keystore
$ keytool –importcert –keystore keystore.jks –file mykey.cer –alias mykey -storepass changeit -trustcacerts
...
Certificate reply was installed in keystore</pre>



### Create DeploymentRuleSet.jar and sign it with the newly signed key

Now we can get down to the business of creating a .jar file and signing it with our shiny new key. We stash ruleset.xml into a JAR using the "jar" command. Next, we use "jarsigner" to sign "DeploymentRuleSet.jar" with our key which is retrieved from our Java keystore using the "mykey" alias.

<pre class="brush: bash; title: ; notranslate" title="">$ jar -cvf DeploymentRuleSet.jar ruleset.xml
added manifest
adding: ruleset.xml(in = 266) (out= 225)(deflated 15%)
$ jarsigner -keystore keystore.jks -storepass changeit DeploymentRuleSet.jar mykey
</pre>



### Combine signing key and associated root CA certificates

In order to easily distribute our public signing cert as well as those of our CA and any intermediate CAs they should be concatenated into one single file. The order to concatenate them in is CA -> Intermediate -> (Optional intermediates) -> mykey.pem.

<pre class="brush: bash; title: ; notranslate" title="">$ cat rootCA.pem (intermediateCert1.pem, intermediateCert2.pem) mykey.pem &amp;gt; mychain.pem</pre>



### Import mychain.pem into Java keystore on client(s)

In order for the Java browser plugin to accept our Deployment Rule Set without complaining, we need to add its code signing key public key certificate to the Java keystore.  
The Java home path for the browser plugin is different from system Java so we need to import our certificate chain into the browser plugin-specific keystore located at "/Library/Internet Plug-Ins/Contents/Home/lib/security/cacerts". To make the certificate chain available to standalone Java applications as well it must be imported into the system Java keystore at "/Library/Java/Home/lib/security/cacerts".

Both keystores use the same default password: "changeit". For enhanced security, it may be a good idea to [change the password](http://docs.oracle.com/cd/E19957-01/817-3331/6miuccqo3/index.html) for the individual keystores to a new one after importing the certificate chain. This is optional, but a security note worth mentioning.

<pre class="brush: bash; title: ; notranslate" title="">$ keytool -importcert -keystore /Library/Internet\ Plug-Ins/Contents/Home/lib/security/cacerts -storepass changeit -alias mykey -file mychain.pem -noprompt
$ keytool -importcert -keystore /Library/Java/Home/lib/security/cacerts -storepass changeit -alias mykey -file mychain.pem -noprompt</pre>



### Testing the Deployment Rule Set

Excelsior! We should now be able to place our DeploymentRuleSet.jar file into its designated path and load a web page we whitelisted in the ruleset.xml file. If all went well the Java application on the web page will load without any warnings from the JVM about the DRS using an untrusted self-signed certificate or the application being blocked because of its unsigned or self-signed status. You can verify the presence of an active Deployment Rule Set by navigating to the "Java" preference pane in System Preferences and clicking the "Security" tab. If active, the tab will contain a line of blue text that says "View the active Deployment Rule Set". The blue text can be clicked to view the current rule set in a new window. The Deployment Rule Set will also allows inspection of the signing certificate and its associated root certificates. These should match the code signing key's certificate and any root and intermediary CA certificates used in our previous steps.

<figure id="attachment_308" class="thumbnail wp-caption alignnone" style="width: 288px">[<img src="http://enterprisemac.bruienne.com/wp-content/uploads/2014/03/java7u51_security_high-278x300.png" alt="java7u51_security_high" width="278" height="300" class="size-medium wp-image-308" />][1]<figcaption class="caption wp-caption-text">Java 7u51 Security tab</figcaption></figure>  
<figure id="attachment_310" class="thumbnail wp-caption alignnone" style="width: 310px">[<img src="http://enterprisemac.bruienne.com/wp-content/uploads/2014/03/java7u51_security_DRS-300x262.png" alt="java7u51_security_DRS" width="300" height="262" class="size-medium wp-image-310" />][2]<figcaption class="caption wp-caption-text">Java 7u51 Deployment Rule Set</figcaption></figure>  
<figure id="attachment_320" class="thumbnail wp-caption alignnone" style="width: 310px">[<img src="http://enterprisemac.bruienne.com/wp-content/uploads/2014/03/java7u51_security_DRS_certs-300x205.png" alt="Java 7u51 DRS Certificates" width="300" height="205" class="size-medium wp-image-320" />][3]<figcaption class="caption wp-caption-text">Java 7u51 DRS Certificates</figcaption></figure>

## **Conclusion**

Hopefully this will help a few Mac admins with deploying a self-signed DeploymentRuleSet.jar file using their organization's local CA. If you have questions or comments leave them at the end of this post, find me on [Twitter](https://twitter.com/bruienne) or on [Freenode IRC](https://webchat.freenode.net/) in ##osx-server.



### Addendum: Package postflight

To make distribution of the signing certificate chain a little easier I've included a postinstall script that can be added to an installer package. The postinstall script will check the browser plugin and system keystores for the presence of the "my\_chain" alias and if it is not found in either one of the keystores it adds the certificate chain. The script expects the installer to drop "my\_chain.pem" into /tmp and it securely removes the file after completion of the script. You are free to change file names and aliases as needed.

<pre class="brush: bash; title: ; notranslate" title="">#!/bin/bash

# Check Java Plugin and System keystores for existence of the signing cert.
#   If found, we skip installation and report success. If not found, tag the
#   keystore as needing installation and proceed with installation. Check result
#   of installation afterwards and log the result for reporting later.

# Executable and keystore file statics
KEYTOOL_LIST=&amp;quot;/usr/bin/keytool -list -storepass changeit -keystore &amp;quot;
KEYTOOL_IMPORT=&amp;quot;/usr/bin/keytool -importcert -storepass changeit -trustcacerts -file /tmp/my_chain.pem -alias my_chain -noprompt -keystore &amp;quot;
JAVA_PLUGIN=&amp;quot;/Library/Internet Plug-Ins/JavaAppletPlugin.plugin/Contents/Home/lib/security/cacerts&amp;quot;
JAVA_SYSTEM=&amp;quot;/Library/Java/Home/lib/security/cacerts&amp;quot;
LOGGER=&amp;quot;/usr/bin/logger -t JAVADRSINSTALL&amp;quot;
REMOVE_PEM=&amp;quot;/usr/bin/srm /tmp/my_chain.pem 2&amp;gt;&amp;amp;1 &amp;gt;/dev/null&amp;quot;

# Initialize reporting variables. I like string comparisons, deal with it.
keystore_plugin='0'
keystore_system='0'

# Check whether the signing cert is installed in the Java plugin keystore
if ! `${KEYTOOL_LIST} &amp;quot;${JAVA_PLUGIN}&amp;quot; | grep my_chain 2&amp;gt;&amp;amp;1 &amp;gt;/dev/null`; then
    ${LOGGER} &amp;quot;Cert chain for Java DRS must be installed in Java Plugin.&amp;quot;
    keystore_plugin='1'
fi
# Check whether the signing cert is installed in the System Java keystore
if ! `${KEYTOOL_LIST} &amp;quot;${JAVA_SYSTEM}&amp;quot; | grep my_chain 2&amp;gt;&amp;amp;1 &amp;gt;/dev/null`; then
    ${LOGGER} &amp;quot;Cert chain for Java DRS must be installed in System Java Home.&amp;quot;
    keystore_system='1'
fi

# If we didn't find the signing key in the keystores we need to install them.

# Install into Java plugin keystore
if [[ $keystore_plugin == '1' ]]; then
    echo $keystore_status
    ${LOGGER} &amp;quot;Installing cert chain for Java DRS into JavaAppletPlugin&amp;quot;
    ${KEYTOOL_IMPORT} &amp;quot;${JAVA_PLUGIN}&amp;quot; 2&amp;gt;&amp;amp;1 &amp;gt;/dev/null

    # Check whether our signing key is now in the keystore
    if `${KEYTOOL_LIST} &amp;quot;${JAVA_PLUGIN}&amp;quot; | grep my_chain 2&amp;gt;&amp;amp;1 &amp;gt;/dev/null`; then
        keystore_plugin='2'
    fi
fi

# Install into System Java keystore
if [[ $keystore_system == '1' ]]; then
    ${LOGGER} &amp;quot;Installing cert chain for Java DRS into System Java Home&amp;quot;
    ${KEYTOOL_IMPORT} &amp;quot;${JAVA_SYSTEM}&amp;quot; 2&amp;gt;&amp;amp;1 &amp;gt;/dev/null

    # Check whether our signing key is now in the keystore
    if `${KEYTOOL_LIST} &amp;quot;${JAVA_SYSTEM}&amp;quot;  | grep my_chain 2&amp;gt;&amp;amp;1 &amp;gt;/dev/null`; then
        keystore_system='2'
    fi
fi

# Report on status of installs, log any failures and securely remove our key

# No installation needed for either keystore, report it
if [[ ($keystore_plugin == '0') &amp;amp;&amp;amp; ($keystore_system == '0') ]]; then
    ${LOGGER} &amp;quot;Java DRS cert chain install not needed.&amp;quot;
    ${REMOVE_PEM}

# Both checks came back as failed, report it
elif [[ ($keystore_plugin == '1') &amp;amp;&amp;amp; ($keystore_system == '1') ]]; then
    ${LOGGER} &amp;quot;Java DRS cert chain install into all keystores failed.&amp;quot;
    ${REMOVE_PEM}

# Both checks came back correctly, report success for all keystores
elif [[ ($keystore_plugin == '2') &amp;amp;&amp;amp; ($keystore_system == '2') ]]; then
    ${LOGGER} &amp;quot;Java DRS cert chain install into all keystores complete.&amp;quot;
    ${REMOVE_PEM}

# Java Plugin installation not needed, System Java keystore succeeded.
elif [[ ($keystore_plugin == '0') &amp;amp;&amp;amp; ($keystore_system == '2') ]]; then
    ${LOGGER} &amp;quot;Java DRS cert chain install into Java Plugin keystore not needed.&amp;quot;
    ${LOGGER} &amp;quot;Java DRS cert chain install into System Java keystore successful.&amp;quot;
    ${REMOVE_PEM}

# Java Plugin installation succeeded, System Java keystore not needed.
elif [[ ($keystore_plugin == '2') &amp;amp;&amp;amp; ($keystore_system == '0') ]]; then
    ${LOGGER} &amp;quot;Java DRS cert chain install into Java Plugin keystore successful.&amp;quot;
    ${LOGGER} &amp;quot;Java DRS cert chain install into System Java keystore not needed.&amp;quot;
    ${REMOVE_PEM}

# Java Plugin installation not needed, System Java keystore failed.
elif [[ ($keystore_plugin == '0') &amp;amp;&amp;amp; ($keystore_system == '1') ]]; then
    ${LOGGER} &amp;quot;Java DRS cert chain install into Java Plugin keystore not needed.&amp;quot;
    ${LOGGER} &amp;quot;Java DRS cert chain install into System Java keystore failed.&amp;quot;
    ${REMOVE_PEM}

# Java Plugin installation failed, System Java keystore not needed.
elif [[ ($keystore_plugin == '1') &amp;amp;&amp;amp; ($keystore_system == '0') ]]; then
    ${LOGGER} &amp;quot;Java DRS cert chain install into Java Plugin keystore failed.&amp;quot;
    ${LOGGER} &amp;quot;Java DRS cert chain install into System Java keystore not needed.&amp;quot;
    ${REMOVE_PEM}

# Java Plugin installation failed, System Java keystore succeeded.
elif [[ ($keystore_plugin == '1') &amp;amp;&amp;amp; ($keystore_system == '2') ]]; then
    ${LOGGER} &amp;quot;Java DRS cert chain install into Java Plugin keystore failed.&amp;quot;
    ${LOGGER} &amp;quot;Java DRS cert chain install into System Java keystore successful.&amp;quot;
    ${REMOVE_PEM}

# Java Plugin installation succeeded, System Java keystore failed.
elif [[ ($keystore_plugin == '2') &amp;amp;&amp;amp; ($keystore_system == '1') ]]; then
    ${LOGGER} &amp;quot;Java DRS cert chain install into Java Plugin keystore successful.&amp;quot;
    ${LOGGER} &amp;quot;Java DRS cert chain install into System Java keystore failed.&amp;quot;
    ${REMOVE_PEM}
fi

exit 0
</pre>

 [1]: http://enterprisemac.bruienne.com/wp-content/uploads/2014/03/java7u51_security_high.png
 [2]: http://enterprisemac.bruienne.com/wp-content/uploads/2014/03/java7u51_security_DRS.png
 [3]: http://enterprisemac.bruienne.com/wp-content/uploads/2014/03/java7u51_security_DRS_certs.png
