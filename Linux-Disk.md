# df/du command
For a data analysis platform, new data comes every data. If there are more data sources or old data is not deleted, the disk is becoming more and more critical. We need to know where the disk costs.

For clickhouse, we can get table's disk usage from system.tables, system.parts, system.columns. But if you compare the result with **df**, it may have some gap. In my cases, all the table used 4.8TB, but df returned 5.1TB used.

```
$ clickhouse-client "select formatReadableSize(sum(total_bytes)) from system.tables"
4.80 TiB

$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       126G   28G   86G  25% /
/dev/md0        5.2T  5.1T  141G  98% /mnt/test
tmpfs            23G     0   23G   0% /run/user/1003
```
There is no other disk costing folders. Why there has about 300G diff?

# Linux Block

## File size VS Disk usage



Linux has minimal disk unit: block. Each file will cost at east one block even if only 1 byte. We can see the block information by many commands.  For example:

```
$ stat -f .
  File: "."
    ID: 100000000 Namelen: 255     Type: wslfs
Block size: 4096       Fundamental block size: 4096
Blocks: Total: 124738303  Free: 75580063   Available: 75580063
Inodes: Total: 999        Free: 1000000

$ sudo tune2fs -l /dev/md0

```

For Linux block, I referenced link [block](http://linuxintro.org/wiki/Blocks,_block_devices_and_block_sizes) and [Unix File System](https://web.cs.wpi.edu/~rek/DCS/D04/UnixFileSystems.html).


Since we have many parts, sometimes for one partitions, there are many parts folder. Also our table has too many columns. The result is that there are many files. More files, more extra disk wasting since each file will used several blocks.