**Step-by-step instructions for reproducing C2's SDC22 results**

[![License](https://licensebuttons.net/l/by/4.0/88x31.png)](https://creativecommons.org/licenses/by/4.0/)

C2 SDC22
================

```
XX              XXXXX XXX         XX XX           XX       XX XX XXX         XXX
XX             XXX XX XXXX        XX XX           XX       XX XX    XX     XX   XX
XX            XX   XX XX XX       XX XX           XX       XX XX      XX XX       XX
XX           XX    XX XX  XX      XX XX           XX       XX XX      XX XX       XX
XX          XX     XX XX   XX     XX XX           XX XXXXX XX XX      XX XX       XX
XX         XX      XX XX    XX    XX XX           XX       XX XX     XX  XX
XX        XX       XX XX     XX   XX XX           XX       XX XX    XX   XX
XX       XX XX XX XXX XX      XX  XX XX           XX XXXXX XX XX XXX     XX       XX
XX      XX         XX XX       XX XX XX           XX       XX XX         XX       XX
XX     XX          XX XX        X XX XX           XX       XX XX         XX       XX
XX    XX           XX XX          XX XX           XX       XX XX          XX     XX
XXXX XX            XX XX          XX XXXXXXXXXX   XX       XX XX            XXXXXX
```

C2 is Los Alamos National Laboratory (LANL)'s next-generation campaign storage prototype built upon Seagate's Kinetic Active Disk technology to enable direct query processing in disk drives reducing data movement speeding up large-scale scientific data analytics on the lab's future computing platforms. Early results of C2 were presented at [2022 SNIA Storage Developer Conference (SDC22)](https://storagedeveloper.org/) taking place September 12-15, 2022 in Fremont, CA. This guide provides step-by-step instructions for reproducing C2's results in SDC22. Please see our [presentation slides](c2-sdc22-slides.pdf) for more information regarding this research prototype.

C2 is developed under U.S. Government contract 89233218CNA000001 for LANL, which is operated by Triad National Security, LLC for the U.S. Department of Energy/National Nuclear Security Administration.

# Platform

In this guide, we will focus on CentOS 8 and ZFS 2.1.5. We assume that there is one ZFS host and 5 Kinetic drives for a 4+1 raidz pool. The 5 drives each act as an NVMeOF target. The host acts as an NVMeOF initiator and names the drives as /dev/nvme1n1, /dev/nvme2n1, ..., and /dev/nvme5n1.

# Step 1: Install ZFS

We can use the following commands to install zfs on centos 8.

```bash
sudo dnf install https://zfsonlinux.org/epel/zfs-release-2-2$(rpm --eval "%{dist}").noarch.rpm
sudo dnf install -y epel-release
sudo dnf install -y kernel-devel
sudo dnf install -y zfs
```

# Step 2: Create a zpool

To create a zpool, we first load the zfs kernel module. We then configure zfs to enable large zfs record sizes that are beyond the limit set by zfs by default (we need 4MB records while the default maximum is just 1MB). After that, we will be able to create our zpool using the 5 Kinetic drives that we have.

In this guide, we will name our zpool `mypool`. We will use `-f` to force pool creation (even if there was a pool previously created on the drives) and `-O recordsize=4M` to apply the right record size (which is 4MB) for our pool.

```bash
sudo modprobe zfs
echo 16777216 | sudo tee /sys/module/zfs/parameters/zfs_max_recordsize 
sudo zpool create -f -O recordsize=4M mypool raidz1 /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1 /dev/nvme4n1 /dev/nvme5n1 
```

Once a pool is created, we can use `sudo zpool status` to check its status and view its member drives.

```bash
  pool: mypool
 state: ONLINE
config:

	NAME           STATE     READ WRITE CKSUM
	mypool         ONLINE       0     0     0
	  raidz1-0     ONLINE       0     0     0
	    nvme1n1    ONLINE       0     0     0
	    nvme2n1    ONLINE       0     0     0
	    nvme3n1    ONLINE       0     0     0
	    nvme4n1    ONLINE       0     0     0
	    nvme5n1    ONLINE       0     0     0

errors: No known data errors
```

# Step 3: Insert data into ZFS

In this guide, we will use a [sample dataset](https://github.com/lanl-future-campaign/c2-vpic-sample-dataset) generated from a real-world scientific application called [VPIC](https://github.com/lanl/vpic) which models kinetic plasmas in 3D space and writes particle state to storage for subsequnet analytics.

To insert this dataset into zfs, we first download it to a temporary location on the zfs host. We then run a converter to convert the data from its original binary format to c2's custom RAID-aligned parquet format (required for in-drive data analytics) and insert that into zfs.

As data is written to zfs, zfs will raid the data and distribute the data and parity chunks across the 5 kinetic drives for storage. C2's custom RAID-aligned parquet format ensures that each drive will always see a complete data record (in our case a full particle) despite the raid process which makes in-drive analytics possible (see our [presentation slides](c2-sdc22-slides) for details on c2's custom data format).

To download the sample dataset to the zfs host, we do:

```bash
cd /tmp
git clone https://github.com/lanl-future-campaign/c2-vpic-sample-dataset.git
```

To convert the data into the right format and insert it into zfs, we first build the converter program available at https://github.com/lanl-future-campaign/c2-parquet-writer. We then run the following:

```bash
sudo c2-parquet-writer/build/writer -j 8 /tmp/c2-vpic-sample-dataset/particles /mypool
```

The entire conversion process should finish in just a few seconds. Here's a sample output of it (shortened to reduce space).

```
[FROM] /tmp/c2-vpic-sample-dataset/particles/eparticle.336.0.bin [TO] /mypool/eparticle.336.0.bin.parquet [WHERE] 131072 particles were processed
[FROM] /tmp/c2-vpic-sample-dataset/particles/eparticle.360.0.bin [TO] /mypool/eparticle.360.0.bin.parquet [WHERE] 131072 particles were processed
[FROM] /tmp/c2-vpic-sample-dataset/particles/eparticle.432.0.bin [TO] /mypool/eparticle.432.0.bin.parquet [WHERE] 131072 particles were processed
...
Done
```

Note also that the resulting dataset should be exactly 168MB in size.

```bash
# sudo du -sh /mypool
168M	/mypool
```

# Step 4: Transform ZFS filenames to low-level disk LBAs

First, we build C2's custom zfs tool named libzdb2 available at https://github.com/lanl-future-campaign/c2-libzdb2/tree/sdc22. We then execute the following to obtain the low-level disk LBAs of our parquet dataset. When building libzdb2, please make sure to use the `sdc22` branch and set `CMAKE_BUILD_TYPE` to `Release`.

```bash
sudo c2-libzdb2/build/src/zdb mypool .
````

Libzdb2 expects two parameters. One is the name of the target zpool (mypool in our case). The other is the relative path of the directory that stores the data within the zpool (`.` in our case which refers to the root directory of the zpool).

The transformation process should finish in just about 1 second. Here's a sample output of it.

```
#4,903985664,1048576
#0,903986176,1048576
#1,903986176,1048576
#2,903986176,1048576
#3,895314432,1048576
...
ZDB query time: 0.226 s
Total: 1.556 s
Done
```

Here, each '#'-labeled line shows the location of a particular data block on one of the 5 kinetic drives that stores our data. Each row is a tuple of 3 elements. The first element is the index of the drive. 0 refers to the first drive which is /dev/nvme1n1. 4 refers to the last drive which is /dev/nvme5n1. The second element is the offset of the block on the drive. The last element is the size of the block, which is always 1MB due to c2's RAID-alignment control.

Because our dataset is 168MB in size, this output should contain exactly 168 lines of data locations. Each points to a 1MB data chunk of our dataset on storage. Libzbd2 only prints data chunks. Parity chunks are not printed. In the next step, we will use this location data to inform our 5 kinetic drives to perform in-drive operations.

# Step 5: Prepare For In-Drive Analytics

Now that we have the low-level LBAs for our data reported by zfs, the next step is to convert it to something that each kinetic disk can use directly. This includes separating LBAs for different disks to different files, specifying the disk partition (/dev/sda4) that was exposed to zfs to store the data, and offseting each LBA by 1MB to account for the disk partition table occupying space at the beginnig of that parition.

Libzdb2 carries a small post-processing program that can accomplish this very task. Just compile and run it.

```
c2-libzdb2/build/src/zdb_pp 5 mypool
```

# Reference

[1] https://openzfs.github.io/openzfs-docs/Getting%20Started/RHEL-based%20distro/index.html
