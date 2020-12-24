+++ 
date = "2020-08-22"
title = "Converting Raid 1 to Raid 5 on Linux file systems"
tags = ["linux", "opensource", "devops"]
categories = ["blog"]
authors = ["csgeek"]
+++

Converting from Raid 1 to Raid 5
This assumes you have a functional Raid 1 and wish to convert it to a Raid 5. 

Disclaimer:  At some point during this process I realized that I had a bad mother board.  The reason my /dev/sdd1 failed wasn't the drive, but the bus on the board.  That being said, this is the unverified procedure. 

I'm running on Ubuntu 12.10 but you should be able to do this on any modern Linux distribution.

1.  Stop all mdadm related tasks.

```bash
sudo umount /media/data #of whatever mountpoint is appropriate.
sudo /etc/init.d/mdadm stop 
sudo mdadm --stop /dev/md0
```

2.  Change the raid layout

This part is kind of scary, and I wouldn't advice mounting the raid at this point.  I especially didn't like the fact that it looks like it's overwriting my raid.. that made me nerveous, but it's essentially restrucuting how data is stored, and putting  a raid 5 mapping on 2 drives.  ie.  creating a degraded raid 5.

```
#update this as appropriate.
mdadm --create /dev/md0 --level=5 --raid-devices=2 /dev/sda1 /dev/sdd1 
```

WARNING: wait for this step to complete.. look at /prod/mdstats and wait for it to finish before proceeding.



3.  Add a 3rd drive.

#in my case I started with /dev/sdd1, added /dev/sda1 to create a raid 1, and then adding /dev/sdb1 as the final device.  You don't have to follow my convention.  That made sense for my use case, since my dead drive was /dev/sdd you can simply start with /dev/sda1 and go alphabetical. 

```
mdadm --add /dev/md0 /dev/sdb1
mdadm --grow /dev/md0 --raid-devices=3
```

This part took around 15 hours for me.  

At this point, I'd be okay with mounting my raid partition.  Again, it's safer not to... but.. it's won't break the process if you do.

4.  Expand File System.

At this point what we have is the equivalent of have a large hard drive, but a smaller partition on it.  We need to grow the local file system.

I'm covering 3 use cases. 

a.  LVM

I've have had lvm on my raid before.  Actually, I used to have raid + lukefs encryption + lvm.  Too many layers though the performance isn't as bad as you might expect.

TODO:  I have to look this up... I'll update this eventually.



b. XFS

xfs is a bit odd, you need to have the file system mounted in order to grow it. 

xfs_repair -n /dev/md0  #just to be safe.
mount /dev/md0 /media/data

xfs_growfs /media/data


c. EXT3/4

```
e2fsck -f /dev/md0 #check file system
resize2fs /dev/md0   #grow file system
```

5.  Update fstab

This should not need any changes, but just in case:

```
/dev/md0        /media/data     xfs     defaults
```


6. Update mdadm.conf

```
sudo su - 
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
```

##edit the file and remove the next to last line.  ie.  The command above appends the new mdadm config to your config file.  So remove the previous raid 1 line.  There should be a single line defining md0 which looks something like this:

```
ARRAY /dev/md/md0 metadata=1.2  UUID=0ec3c5aa:5cee600b:ef1e8f7d:09b20cc8
```

This is the line I removed:

```
#ARRAY /dev/md/md0 metadata=0.90 UUID=bf8a2737:554e654c:c2eab133:b01f9710
```

In other words, assuming you only have 1 raid setup, your mdadm.conf should only have a single ARRAY configured.

References:  

  1.  http://www.davelachapelle.ca/2008/07/25/converting-raid-1-to-raid-5/