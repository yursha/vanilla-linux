Download from archlinux-2018.07.01-x86_64.iso https://www.archlinux.org

Insert USB stick into MacBook port.

```
shell> diskutil list
[...]
/dev/disk3 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *15.9 GB    disk3
   1:                       0xEF 
   
shell> diskutil unmountDisk /dev/disk3
Unmount of all volumes on disk3 was successful

shell> sudo dd if=~/Downloads/archlinux-2018.07.01-x86_64.iso of=/dev/rdisk3 bs=1m
574+0 records in
574+0 records out
601882624 bytes transferred in 21.534520 secs (27949665 bytes/sec)

shell> diskutil eject /dev/disk3
Disk /dev/disk3 ejected
```

NOTE: `dd` copying will take some time. You cant press Ctrl+T to see the progress.
