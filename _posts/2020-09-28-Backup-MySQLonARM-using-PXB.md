---
layout: post
title: Backup MySQL on ARM using PXB on ARM
img: ./images/blog13/pxb.png
author: Krunal Bauskar
update: 28th Sep 2020
---

One of the key activities that a DBA does regularly is creating backup of an active database instance. So having a working backup tool in place especially for hot-backup is important when a user think of running DB-on-ARM. Even though Percona Xtrabackup (PXB) is not yet officially offered on ARM one can compile and successfully run it on ARM. This article will help explore the same.

## <span style="color:#4885ed">Compiling</span>

Percona-Xtrabackup is one of the most widely used open source tools for backing up MySQL Server (and its variants). It offers incremental/partial/full backup. The official packages for PXB are not yet available on ARM but we decided to give it a try by building it directly from the source.

Process is pretty easy and Percona documentation further simplifies it by making it a 3 steps process:
[https://www.percona.com/doc/percona-xtrabackup/8.0/installation/compiling_xtrabackup.html](https://www.percona.com/doc/percona-xtrabackup/8.0/installation/compiling_xtrabackup.html)

**Step-1: checkout**

```
$ git clone https://github.com/percona/percona-xtrabackup.git
$ cd percona-xtrabackup
$ git checkout 8.0
```

**Step-2: install dependencies**

```
sudo apt install build-essential flex bison automake autoconf \
libtool cmake libaio-dev mysql-client libncurses-dev zlib1g-dev \
libgcrypt11-dev libev-dev libcurl4-gnutls-dev vim-common
sudo yum install cmake openssl-devel libaio libaio-devel automake autoconf \
bison libtool ncurses-devel libgcrypt-devel libev-devel libcurl-devel zlib-devel \
vim-common
```
```
sudo yum install cmake openssl-devel libaio libaio-devel automake autoconf \
bison libtool ncurses-devel libgcrypt-devel libev-devel libcurl-devel zlib-devel \
vim-common
```

**Step-3: Build**

```
cd percona-xtrabackup && mkdir build && cd build
cmake .. -DWITH_NUMA=1 -DDOWNLOAD_BOOST=1 -DWITH_BOOST=<boot-dir> -DCMAKE_INSTALL_PREFIX=<install-path>'
make && make install
(check pxb doc for more compilation options).
(You can also opt to build a release tag vs trunk).
```

Most of the steps are self-explanatory.
Since I was building PXB on ARM for first time I was bit concerned if all things will work as expected.
Fortunately, all dependencies are available and code too compiled as expected without any issues.

Build-environment:
ubuntu-18.04 (default gcc compiler)

<span style="color: #de5246">
./xtrabackup --version<br>
./xtrabackup version 8.0.14 based on MySQL server 8.0.21 Linux (aarch64) (revision id: 106dc5c9ce0)
</span>

## <span style="color:#4885ed">Testing Backup</span>

Process to take backup and other options are the same. I will suggest you to go over the documentation but here are quick snippets of commands.

```
./bin/xtrabackup --user=root --socket=/tmp/n1.sock --backup --target-dir=<target-backup-dir> --datadir=<datadir>
./bin/xtrabackup --prepare --target-dir=<target-dir>
```

As you could see, for taking backup, the tools need to be provided with a data-directory and that means the tools should be installed on the server that hosts the database. This creates a PXB on ARM dependency.

Using the steps to take backup we were able to take backup and restore it (including with and without load active while taking backup). All things worked as expected. No issues observed with the backup (based on our limited testing).

## <span style="color:#4885ed">Conclusion</span>

One of the most important and complex aspects <span style="color: #1aa260">**backup**</span> of DB (mysql*) on ARM is taken care. Thanks to Percona Xtrabackup.

<br>
<em>If you have more questions/queries do let me know. Will try to answer them.</em>
