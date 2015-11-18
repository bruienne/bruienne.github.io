---
ID: 462
post_title: BSDPy Redis caching
author: pepijn
post_date: 2015-06-03 18:07:51
post_excerpt: ""
layout: post
permalink: >
  http://enterprisemac.bruienne.com/2015/06/03/bsdpy-redis-caching/
published: true
---
As we have been ramping up the BSDPy coverage in our environment it became clear that it was spending a lot of its time making API calls for clients that were checking in and not getting enough time to respond back to clients with boot acknowledgments. To offer some background to this problem the following are the steps that the BSDP client and server go through in the process of successfully Netbooting.

*   First, the client sends a broadcast `DHCP INFORM LIST` request with vendor-specific options that contain its make and model. e.g. `"AAPL/BSDPC MacBookPro8,2"`.  
    
*   Second, the server replies with a `DHCP ACK LIST` reply that once again contains vendor-specific options that contain a list of one or more images that the client is entitled to. This list also includes information designating the default image, their IDs and a name that is displayed in the client's boot selector.  
    
*   Next, the client decides either by itself which image it wants to boot from because the user held down the N-key which implies the default image, or the user decides which image to boot from by picking it in the boot picker at startup or using Startup Disk in OS X's System Preferences. To indicate its choice the client sends a `DHCP INFORM SELECT` packet with, you guessed it, vendor-specific options containing the image ID.  
    
*   Lastly the server replies back with a `DHCP ACK SELECT` packet that contains the server name to boot from, a TFTP URI to the booter (kernel) and (in another set of vendor-specific options) an HTTP or NFS URI to the NetBoot.dmg/NetInstall.dmg OS image.  
    

  
In our case as traffic to the BSDPy hosts was exponentially increasing and they were now serving just about all of our campus, thousands upon thousands of clients were sending non-BSDP DHCP requests that were putting a load on the host in general (*conntrack* anyone?) while also flooding BSDPy with BSDP requests, further taxing them. Since the `BSDP INFORM LIST` requests were all being checked against an external API this was causing latency and excessive timeouts as the replies needed to be processed and then sent back out, over and over, taking into account the fact that the Apple BSDP client tends to hammer the network with BSDP requests. Actual boot requests (`DHCP INFORM SELECT`) were going unanswered which was leading to a reduced ability to boot on the client's end. One might argue that this is a case where multi-threading might be able to help, but I wasn't comfortable with the can of worms that subject opens. Instead I decided to see if this issue could be resolved by creating a short-term cache of frequently and/or recently checked in clients to cut down on more expensive and possibly slow API calls in general. Having already considered some form of caching in the past this seemed like as good a time as any to implement it and see if this would improve performance to more acceptable levels. Spoiler alert: it did.

All of the preceding serves as a long way of saying that as of today BSDPy's <a href="https://bitbucket.org/bruienne/bsdpy/branch/api" title="" target="_blank">pre-release branch</a> includes the ability to automatically cache client requests in <a href="http://redis.io/" title="" target="_blank">Redis</a>. Redis is a popular RAM-based <a href="http://en.wikipedia.org/wiki/NoSQL#Key-value_stores" target="_blank">NoSQL key-value store</a> that is the perfect choice for storing this kind of information: simple key-value pairs with easy to configure expirations. For each client request there are a few bits of data we're interested in without the likelihood of the info being dramatically different each time: most of us don't keep more than a few NBI sets to support a legacy OS X version, a current OS X version and sometimes a forked OS X build.

So, in key-value terms the client sends us:

*   Client MAC address
*   Client Model ID
*   Client IP address

The API returns one or more groups of the following keys and their corresponding values:

*   Image ID
*   Image name
*   Image priority
*   Booter path (TFTP)
*   Image path (NFS or HTTP)

Using Redis' <a href="http://redis.io/commands#hash" target="_blank">hash commands</a> (AKA dictionary) this information can then easily be combined and stored by concatenating the client's MAC address, Model ID and IP address into one "unique-enough" key with a number of hash entries: [bash] # Create a key with our image entitlements HSET 00:11:22:33:44:55_MacBookPro8,2_12.34.45.67 image_id 5000 HSET 00:11:22:33:44:55_MacBookPro8,2_12.34.45.67 image_name 'Yosemite 14D136' HSET 00:11:22:33:44:55_MacBookPro8,2_12.34.45.67 image_priority 10 HSET 00:11:22:33:44:55_MacBookPro8,2_12.34.45.67 booter_path '/nbi/Yoyo.nbi/i386/booter' HSET 00:11:22:33:44:55_MacBookPro8,2_12.34.45.67 image_path 'http://myhost.org/nbi/Yoyo.nbi/NI.nbi' # Show all keys for this key HKEYS 00:11:22:33:44:55_MacBookPro8,2_12.34.45.67 1) "image_id" 2) "image_name" 3) "image_priority" 4) "booter_path" 5) "image_path" [/bash] 

The key may look a little goofy but since we're really only interested in holding on to what the API says is the correct Netboot information for this particular permutation of MAC address, Model ID and IP address it works fine. The reason we're not just using the MAC address as our unique key (they *are* supposed to be unique, right?) is to account for the increasing number of Ethernet adapters techs have to use in order to NetBoot modern Macs. This means that the MAC address of one Ethernet adapter may very well be associated with dozens of different Mac models, some of which may have different NBI entitlements.

Since there's no need to hold on to the data forever we can set a fairly short expiration on the key, 5 minutes seems to be a fine limit: [bash]EXPIRE 00:11:22:33:44:55_MacBookPro8,2_12.34.45.67 300[/bash] 

We can always reset the timeout if a client checks in prior to the expiration to account for a machine that may be experiencing network issues of its own and therefor might take a few attempts to successfully boot. After a client has successfully NetBooted it is unlikely to check back in for a while so after 5 minutes its Redis key will be removed.

Enough with the gory technical details you say? Awesome. The way to have BSDPy automatically enable the Redis caching is easiest if you are already running it in a Docker container since Docker lets us use container linking to hook up Redis and have BSDPy start caching. Steps are as follows:

1.  Pull the official <a href="https://registry.hub.docker.com/_/redis/" title="" target="_blank">Redis Docker image</a> from the Docker Hub:  
    `docker pull redis`
2.  Run the Redis image with default settings (no need to expose ports):  
    `docker run -d --name redis redis`
3.  Run the BSDPy Docker image, linking it to the running Redis container:  
    `docker run -d --name bsdpy <MORE OPTIONS> --link redis:db bruienne/bsdpy:1.0`
4.  There is no step four.

The only important configuration item here is the `--link redis:db` flag as it tells Docker the name of the container to link to (`redis`) and what to name the link inside the BSDPy container (`db`). BSDPy looks for environment variables automatically created by Docker that all start with `"DB_"` to determine whether Redis was linked and thus should be used for caching. Linking Redis with any other name will cause BSDPy to ignore it and no caching will occur. When Redis caching was successfully activated the log's startup notices will look something like this: [bash]06/03/2015 05:49:30 PM - DEBUG: ------- Start Docker env vars ------- 06/03/2015 05:49:30 PM - DEBUG: BSDPY_NBI_PATH: /nbi 06/03/2015 05:49:30 PM - DEBUG: BSDPY_IFACE: eth0 06/03/2015 05:49:30 PM - DEBUG: BSDPY_API_URL: https://myapphost.org/api/v1/netboot_images 06/03/2015 05:49:30 PM - DEBUG: BSDPY_PROTO: http 06/03/2015 05:49:30 PM - DEBUG: BSDPY_IP: 192.168.1.100 06/03/2015 05:49:30 PM - DEBUG: ------- End Docker env vars ------- 06/03/2015 05:49:30 PM - DEBUG: Using Redis caching for clients with Redis host 172.17.0.43 on port 6379 06/03/2015 05:49:30 PM - DEBUG: tftprootpath is /nbi 06/03/2015 05:49:30 PM - INFO: Server priority: [251, 71] 06/03/2015 05:49:30 PM - DEBUG: Found $BSDPY_IP - using custom external IP 192.168.1.100 06/03/2015 05:49:30 PM - INFO: Server IP: 192.168.1.100 06/03/2015 05:49:30 PM - INFO: Server FQDN: 192.168.1.100 06/03/2015 05:49:30 PM - INFO: Serving on eth0 06/03/2015 05:49:30 PM - INFO: Using http to serve boot image[/bash] 

For those not using Docker the environment variables to set to a valid Redis host and port are `DB_PORT_6379_TCP_ADDR` and `DB_PORT_6379_TCP_ADDR` - either IP or FQDN for the host variable will work.

And that is all. Try it out, provide feedback, send pull requests!