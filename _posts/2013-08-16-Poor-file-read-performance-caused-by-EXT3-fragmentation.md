---
layout: post
title: Poor file read performance caused by EXT3 file fragmentation
excerpt: Observe and measure file fragmentation
---
 
I own a Synology DS409 and observe low throughput (13 Mo/s) on big files transfer over my local network. 
This is far too slow compared to the disk and network theorical speed. The following investigation shows the 
performance issue can be partially explained by the fragmentation. Let's go.
 
The NAS has four hard drives.
{% highlight bash %}
$ fdisk -l
#=> sda, sdb, sdc, sdd
{% endhighlight %}
 
The first issue to address is whether the perofmrance issue is caused by defective hardware. First let's measure the read speed through the Linux cache. This command bypass the filesystem API and is a good indicator of hardware and driver speed. The cache is cleared during the test to ensure accurate measurement.
 
{% highlight bash %}
$ hdparm -t /dev/sda
#=> /dev/sda:
#=> Timing buffered disk reads: 72 MB in 3.01 seconds = 23.91 MB/sec
{% endhighlight %}
 
The average measured preformance of the disks is `85.4 Mo/s`, `89.2 Mo/s`, `120.8 Mo/s`, `122.5 Mo/s`.
These values looks correct, the standard deviation for each individual hdd is low, and the difference between the drives is due 
to different hardware. Given the raw disk read is fast, the bottleneck is not the hardware.
 
Let's try to read a complete file through the file system API. 
{% highlight bash %}
$ time sh -c "cat /volume0/bigfile > /dev/null"
{% endhighlight %}
 
The averaged file read throughput is `14.5 Mo/s`, which is far below the previous figures. The poor performance only occurs when reading through the file system API.
 
Let's get more information about the file systems and partitions: 
{% highlight bash %}
$ mount 
#=> /dev/md2 on /volume1 type ext3 (usrquota,grpquota) 
#=> /dev/md3 on /volume2 type ext4 (usrjquota=aquota.user,grpjquota=aquota.group,jqfmt=vfsv0,synoacl) 
{% endhighlight %}
 
`md` stands for multiple device driver. There are several physical device for these logical device. The following command helps to know what is under the hood: 
{% highlight bash %}
$  mdadm --detail /dev/md2
{% endhighlight %}
`md2` and `md3` are two multiple device driver made respectively of a JBOD of `sda+sdb` and a RAID1 of `sdc+sdd`.
 
Is the file fragmented? The file system split the file into blocks. A block is the basic unit of storage 
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
 
Last but not least, here is a command to check the fragmentation level of all files over 100 Mo. I installed `ipkg` & `filefrag` to help. 
 
{% highlight bash %}
$  find /volume1 /volume2  ! \( -name @* -prune \) -type f -size +100000 -exec filefrag -B {}  \; | tee fragreport
{% endhighlight %}
 
When I duplicate the file the new instance is not fragmented, and the file read speed is `47.5 Mo/s`. 
Even without fragmentation the raw drive speed is no reached but I will have to defrag my drive first before investigating further.
 
It is common to heard that Linux file systems have no fragmentation issue, unfortunately it is not always true. 
 


