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

C2 is Los Alamos National Laboratory (LANL)'s next-generation campaign storage prototype built upon Seagate's Kinetic Active Disk technology to enable direct query processing in disk drives reducing data movement speeding up large-scale scientific data analytics on the lab's future computing platforms. Early results of C2 were presented at [2022 SNIA Storage Developer Conference (SDC22)](https://storagedeveloper.org/) taking place September 12-15, 2022 in Fremont, CA. This guide provides step-by-step instructions for reproducing C2's results in SDC22. Please see our [presentation slides](c2-sdc22-slides.pdf) for more information regarding this research project.

C2 is developed under U.S. Government contract 89233218CNA000001 for LANL, which is operated by Triad National Security, LLC for the U.S. Department of Energy/National Nuclear Security Administration.

# Platform

We focus on CentOS 8 and ZFS 2.1.5. We assume that there is one ZFS host and 5 Kinetic drives for a 4+1 raidz pool. We further assume that each Kinetic drive acts as an NVMeOF target and that the 5 drives are presented as /dev/nvme1n1, /dev/nvme2n1, ..., /dev/nvme5n1 on the ZFS host.

# Step 1: Install ZFS

One may use the following to install zfs on centos 8. More complete information can be found at https://openzfs.github.io/openzfs-docs/Getting%20Started/RHEL-based%20distro/index.html.

```bash
sudo dnf install https://zfsonlinux.org/epel/zfs-release-2-2$(rpm --eval "%{dist}").noarch.rpm
sudo dnf install -y epel-release
sudo dnf install -y kernel-devel
sudo dnf install -y zfs
```

# Step 2: Create the ZPOOL

To create a zpool, we first load the zfs kernel module. We then configure zfs to enable large zfs record sizes that are beyond the limit set by zfs by default (we need 4MB records while the default maximum is just 1MB). After that, we can then create our zpool on top of the 5 Kinetic drives that we have.

In this guide, we will name our zpool `mypool`. We will use `-f` to force pool creation even if there was a pool previously created on the 5 drives. We will then use `-O recordsize=4M` to apply the right record size for our pool.

```bash
sudo modprobe zfs
echo 16777216 | sudo tee /sys/module/zfs/parameters/zfs_max_recordsize 
sudo zpool create -f -O recordsize=4M mypool raidz1 /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1 /dev/nvme4n1 /dev/nvme5n1 
```

Once a pool is created, we can use `sudo zpool status` to check its status and view its member drives.

