---
published: false
---
## Quickpost

To help others playing around with APFS, wishing to boot from an APFS-formatted volume here's a quick post documenting the current process as of 10.12.4 beta 4 (16E175b) This is currently the quickest way to take an existing HFS boot volume, convert it to APFS and configure it for booting. Going forward you should be working with a non-production test system that you can afford to lose in any horrible APFS conversion accidents. I used a VMware Fusion VM without trouble.<--!more-->

### Prepare for APFS conversion

Although the volume you're working with (I'd recommend using a VM) is already HFS+ formatted, you do need to make sure it is also part of a valid CoreStorage (CS) LVM group. A quick way to determine that is by using `diskutil list` from the Terminal (all of the conversion process will be done in the Terminal):

**No CS configured:**

```
$ diskutil list
/dev/disk1 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                         1.0 TB     disk0
   1:                        EFI EFI                     314.6 MB   disk0s1
   2:                  Apple_HFS Macintosh HD            999.6 GB   disk0s2
   3:                 Apple_Boot Recovery HD             650.0 MB   disk0s3

```

**CS configured:**

```
$ diskutil list
/dev/disk0 (internal):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                         1.0 TB     disk0
   1:                        EFI EFI                     314.6 MB   disk0s1
   2:          Apple_CoreStorage Macintosh HD            999.6 GB   disk0s2
   3:                 Apple_Boot Recovery HD             650.0 MB   disk0s3

/dev/disk1 (internal, virtual):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:                            Macintosh HD           +999.2 GB   disk1
                                 Logical Volume on disk0s2
                                 8AD205C1-E377-4434-BB38-DBC7338374E0
```

In order to convert a non-CS volume to CS run the following (non-destructive) conversion command:

```
$ sudo diskutil cs convert /Volumes/Macintosh\ HD
Started CoreStorage operation on disk0s2 Macintosh HD
Resizing disk to fit Core Storage headers
Creating Core Storage Logical Volume Group
Reviewing boot support loaders
Attempting to unmount disk0s2
Switching disk0s2 to Core Storage
Waiting for Logical Volume to appear
Mounting Logical Volume
Core Storage LVG UUID: 20F0E93C-2B46-44FE-B372-2BEACF627EE4
Core Storage PV UUID: 4B05D92C-0028-4B07-8FE0-850B626D60D8
Core Storage LV UUID: 598EDC9A-C9A1-47FD-B154-B2B7720D484C
Core Storage disk: disk3
Finished CoreStorage operation on disk0s2 Macintosh HD
```

Once a valid CS LVG has been created, you'll need to get the Physical Volume (PV) `/dev/` entry. This can be looked up using the `diskutil cs list` command:

```
$ diskutil cs list
CoreStorage logical volume groups (1 found)
|
+-- Logical Volume Group 20F0E93C-2B46-44FE-B372-2BEACF627EE4
    =========================================================
    Name:         Macintosh HD
    Status:       Online
    Size:         63564750848 B (63.6 GB)
    Free Space:   18948096 B (18.9 MB)
    |
    +-< Physical Volume 4B05D92C-0028-4B07-8FE0-850B626D60D8
    |   ----------------------------------------------------
    |   Index:    0
    |   Disk:     disk1s2
    |   Status:   Online
    |   Size:     63564750848 B (63.6 GB)
    |
    +-> Logical Volume Family 55BBD386-3404-4D39-ABD0-C70EC1716D8A
        ----------------------------------------------------------
        Encryption Type:         None
        |
        +-> Logical Volume 598EDC9A-C9A1-47FD-B154-B2B7720D484C
            ---------------------------------------------------
            Disk:                  disk3
            Status:                Online
            Size (Total):          63193481216 B (63.2 GB)
            Revertible:            Yes (no decryption required)
            LV Name:               Macintosh HD
            Volume Name:           Macintosh HD
            Content Hint:          Apple_HFS
```
As can be seen, the `/dev` entry for this particular CS LVG is `/dev/disk1s2` - this is the volume you will need for the the APFS conversion step that is next.

### APFS conversion

With the correct CS PV in hand we can run the command that will do all the hard work of converting it to APFS: `apfs_hfs_convert`

```
$ apfs_hfs_convert
Usage: apfs_hfs_convert [-e] [-v] [-S <path>] [-n] [-F n] [-M <mount_path>] [-o <option>] [--no-warning] <device_path>
    -e    Estimate apfs metadata size.
    -v    Enable verbose output.
    -S <path>
          Print statistics and information about the converson to the given <path>.
    -n    Don't finalize conversion (dry run).  Volume remains HFS.
    -f    Force conversion if volume is dirty.
    -F n  Slice #n (0-based) should be fixed size.
    -M <mount_path>
          Use the given path to mount APFS during conversion.
    -o <nx_or_apfs_format_options>
          Format options passed through to nx_format and apfs_newfs.
    --no-warning
          Do not warn about APFS being pre-release technology, nor wait for user acknowledgement.
    --watchdog=<seconds>
          Conversion will abort after <seconds> seconds.  Default is 600 seconds for dry run, unlimited otherwise.
```

Although the help text and man page (`man apfs_hfs_convert`) don't mention it, you _must_ target a CS PV for the APFS conversion to work correctly. Make sure to unmount the volume first before continuing. We can do a dry run of the conversion using the `-n` flag to verify that we're targeting the right volume and that nothing goes wrong. We'll also use the verbose flag `-v` to see more detail about what the conversion process is doing:

```
$ sudo apfs_hfs_convert -n -v --no-warning /dev/disk1s2
Mon Feb 27 16:12:57 2017: apfs_hfs_convert (apfs-249.50.206)
Mon Feb 27 16:12:57 2017: Converting device at /dev/disk1s2
    Limiting conversion to 900 seconds.  Use --watchdog=0 for no limit.
Free space extent 15493632 8721
UU space extent 15428096 65536
15428096 74257
volume not encrypted
volume is not converting
No partition map found
Mon Feb 27 16:12:57 2017: Finding inode files for slice 0
    63,564,750,848 bytes
    44,619,214,848 free (70%) before conversion
Mon Feb 27 16:12:58 2017: Creating APFS container
(       0,    32767,   32768) -> (     9,   0,            0)

-- Lots of progress follows --

	44,498,776,064 bytes free (70%)
Mon Feb 27 16:12:58 2017: Converting slice #0
Mon Feb 27 16:12:58 2017: Checking for directory hard links
Mon Feb 27 16:12:58 2017: Creating empty apfs file system
    Volume name: Macintosh HD
apfs_newfs:14765: FS will NOT be encrypted.
    Freeing journal (offset=0x1e0000, size=0x800000)
Mon Feb 27 16:12:58 2017: Converting catalog B-tree
Mon Feb 27 16:13:08 2017:     Updating 14,051 symlinks
Mon Feb 27 16:13:12 2017: Converting attributes B-tree
Mon Feb 27 16:13:22 2017: Converting extents B-tree
Mon Feb 27 16:13:22 2017: Done converting slice #0
      383,618 files
      102,123 folders
       14,051 symlinks
          872 hard link inodes
        2,307 hard links
      347,217 inline EAs
            0 extent-based EAs
            0 extent-based EA overflow records
            0 crypto state records
                0 class 0
                0 class A
                0 class B
                0 class C
                0 class D
                0 class E
                0 class F
    Conversion took 24 seconds
Mon Feb 27 16:13:22 2017: Flushing transactions
    44,005,482,496 bytes free (69%)
Mon Feb 27 16:13:23 2017: Flushing transactions
    44,005,482,496 bytes free (69%)
    613,732,352 bytes used by APFS
    554,914,124 bytes used by HFS
    822,878,208 bytes allocated by HFS
Mon Feb 27 16:13:23 2017: Freeing HFS metadata
  0: (15502353, 15508497,    6144) -> (   MLV,   1,            0)
  0: (15508497, 15514641,    6144) -> (   MLV,   0,            0)
  0: (15514641, 15515665,    1024) -> (DSKLBL,   0,            0)
  0: (15515665, 15516689,    1024) -> (DSKLBL,   1,            0)
  0: (15516689, 15517713,    1024) -> (DSKLBL,   2,            0)
  0: (15517713, 15518737,    1024) -> (DSKLBL,   3,            0)
  0: (15518737, 15518738,       1) -> (VOLHDR,   1,            0)
    44,887,089,152 bytes free (70%)
Mon Feb 27 16:13:23 2017: Unmounting APFS volumes
Mon Feb 27 16:13:23 2017: Flushing transactions
    44,887,089,152 bytes free (70%)
    267,874,304 bytes freed by conversion
Mon Feb 27 16:13:24 2017: Done

Conversion took 27 seconds
     12.062s (44.67%) user
      4.460s (16.52%) system
259,166,208 max resident set size

```

If the dry run doesn't come up with any catastrophic errors it should be safe to run without the `-n` safety net, so re-run the command:

```
$ sudo apfs_hfs_convert -v --no-warning /dev/disk1s2

-- Same output as dry run mode --
```

We can use the `apfs` verb of `diskutil` to verify we have a working APFS volume now:

```
$ diskutil apfs list
APFS Container (1 found)
|
+-- Container disk2 0193DA89-44CF-48C8-AC47-122F60790A3D
    ====================================================
    APFS Container Reference:     disk2
    Capacity Ceiling (Size):      63564750848 B (63.6 GB)
    Capacity In Use By Volumes:   19544182784 B (19.5 GB) (30.7% used)
    Capacity Available:           44020568064 B (44.0 GB) (69.3% free)
    |
    +-< Physical Store disk0s2 6DC74A63-9402-4C77-9F74-66F99F37405A
    |   -----------------------------------------------------------
    |   APFS Physical Store Disk:   disk0s2
    |   Size:                       63564750848 B (63.6 GB)
    |
    +-> Volume disk2s1 952B3A09-2FFC-39A6-BDB6-70D28952CC00
        ---------------------------------------------------
        APFS Volume Disk (Role):   disk2s1 (No specific role)
        Name:                      APFS
        Mount Point:               /
        Capacity Consumed:         19423735808 B (19.4 GB)
        Capacity Reserve:          None
        Capacity Quota:            None
        Cryptographic Security:    None
```

### Configuring the boot volume

If there were no unexpected errors you'll still need to do one more thing in order for the early boot process to know where to find the boot volume as this appears to not be handled automatically right now (this should change once APFS is deemed ready for prime time). We need to set the `boot-args` flag in nvram which by default is not allowed by System Integrity Protection (SIP) - if you have disabled this (WHY?!) you can run the following command directly from Terminal without rebooting, otherwise reboot to the Recovery partition (hold down cmd-R at boot):

```
$ nvram boot-args="rd=disk2s1"
```

From the Recovery partition you will not need `sudo` - run from anywhere else `nvram` will require that it is prepended with `sudo`.

### Reboot

If everything went right, you should now be able to boot from the APFS volume. In my experience the volume wasn't properly detected by the Startup Disk preference pane so I forced the boot volume in my tester VM using the startup disk preference in VMware Fusion. This setting overrides whatever OS preference is set, and works in our advantage here to force a boot from our newly created APFS volume. Enjoy!