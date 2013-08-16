---
layout: post
title: Poor NAS read performance caused by EX3 file fragmentation
---

I own a Synology DS409 and observe low throughput (13Mo/s) on big files transfer over my local network. 
This is far too slow compared to the disk and network throughput. The following investigation shows the 
performance issue can be partially explained by the fragmentation. Let's go.

List drives:
{% highlight bash %}
$ fdisk -l
#=> sda, sdb, sdc, sdd
{% endhighlight %}
The NAS has four hard drives.

Bench disk read speed through the Linux cache. This command clear clear the cache during the processing 
to ensure accurate measurement.

{% highlight bash %}
$ hdparm -t /dev/sda
#=> /dev/sda:
#=> Timing buffered disk reads: 72 MB in 3.01 seconds = 23.91 MB/sec
{% endhighlight %}

The average Multiple run of the Retruns `85.4 Mo/s, 89.2 Mo/s, 120.8 Mo/s, 122.5 Mo/s`.
These perfs looks good, the standard deviation for each individual hdd is low, and the difference is due 
to different hardware. 
 
Let's try to read a complete file through the file system api. 
{% highlight bash %}
$ time sh -c "cat /volume0/big > /dev/null
{% endhighlight %}
 
Hey 14.5 Mo/s is far below the hard drive capabilities! There is something to dig here. 
Let's enumerate the file systems and partitions: 
{% highlight bash %}
$ mount 
#=> /dev/md2 on /volume1 type ext3 (usrquota,grpquota) 
#=> /dev/md3 on /volume2 type ext4 (usrjquota=aquota.user,grpjquota=aquota.group,jqfmt=vfsv0,synoacl) 
{% endhighlight %}
 
`md` stands for multiple device driver. The following command helps to know what is under the hood: 
{% highlight bash %}
$  mdadm --detail /dev/md2
{% endhighlight %}
`md2` and `md3` are two multiple device driver made respectively of a JBOD of `sda+sdb` and a RAID1 of `sdc+sdd`.
 
Is the file fragmented ? The file system split the file into blocks. A block is the basic unit of storage 
on a disk. The following command shows the block size on this partition: 
 
{% highlight bash %}
$ tune2fs -l /dev/md2
{% endhighlight %}
 
The blocks are 4096 octets. Let's display the number of the block that have been picked to store the file on the hard drive. 
{% highlight bash %}
$ debugfs /dev/md2 -C "stat /public/big"
{% endhighlight %}
 
I processed the output with a python program and detected 28352 contiguous bloc segment. Wow, 
I picked that file randomly, I guess the whole drive is scrambled that way. 
 
LAst but not least here is a command to check the fragmentation level of all files over 100Mo. I installed ipkg & filefrag to help. 
 
{% highlight bash %}
$  find /volume1 /volume2  ! \( -name @* -prune \) -type f -size +100000 -exec filefrag -B {}  \; | tee fragreport
{% endhighlight %}

When I duplicate the file the new instance is not fragmented, and the file read speed is 47.5 Mo/s. 
Even without fragmentation the raw drive speed is no reached but I will have to defrag my drive first before investigating further.

It is common to heard that Linux file systems have no fragmentation issue, unfortunately it is not always true. 












