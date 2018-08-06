# Toolchain

We need to build a host-independent toolchain to be able to compile everything we need for Vanilla linux without interference from the host.

```
sh> mkdir /mnt/lfs/tools
sh> ln -s /mnt/lfs/tools /
```

The symlink enables the toolchain to be compiled so that it always refers to `/tools`. 
So that the compiler, assembler, and linker will work before and after chroot.

## Create a user

```
sh> groupadd lfs
sh> useradd -s /bin/bash -g lfs -m -k /dev/null lfs
sh> passwd lfs
sh> chown lfs:lfs /mnt/lfs/tools
sh> chown lfs:lfs /mnt/lfs/sources
sh> su - lfs
```

## Create a clean shell environment

Prevent environment variables from the host system from leaking into the build environment.

```
sh> cat > ~/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
EOF
```

```
sh> cat > ~/.bashrc << "EOF"
set +h # turn of bash hashing of binary paths
umask 022
LFS=/mnt/lfs
LC_ALL=C
LFS_TGT=$(uname -m)-lfs-linux-gnu
PATH=/tools/bin:/bin:/usr/bin
export LFS LC_ALL LFS_TGT PATH
EOF
```

```
sh> source ~/.bashrc
```

