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

C2 is developed under U.S. Government contract 89233218CNA000001 for Los Alamos National Laboratory (LANL), which is operated by Triad National Security, LLC for the U.S. Department of Energy/National Nuclear Security Administration. Early results of C2 were presented at [2022 SNIA Storage Developer Conference (SDC22)](https://storagedeveloper.org/) taking place September 12-15, 2022 in Fremont, CA. C2 is developed by LANL in collaboration with Seagate with their novel Kinetic Active Disk research platform. Please see our presentation slides for more information.

This guide provides step-by-step instructions for reproducing C2's results in SDC22.

# Platform

We focus on CentOS 8 and ZFS 2.1.5.

# Step 1: Install ZFS

One may use the following to install zfs on centos 8. More complete information can be found at https://openzfs.github.io/openzfs-docs/Getting%20Started/RHEL-based%20distro/index.html.

```bash
sudo dnf install https://zfsonlinux.org/epel/zfs-release-2-2$(rpm --eval "%{dist}").noarch.rpm
sudo dnf install -y epel-release
sudo dnf install -y kernel-devel
sudo dnf install -y zfs
```
