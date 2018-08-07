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

## Cross-compile binutils

The temporary libraries are cross-compiled. Because a cross-compiler by its nature cannot rely on anything from its host system, this method removes potential contamination of the target system by lessening the chance of headers or libraries from the host being incorporated into the new tools.

```shell
pacman -S flex bison
```

```shell
git clone git://sourceware.org/git/binutils-gdb.git
cd binutils-gdb
git checkout binutils-2_31_1
mkdir build
cd build
../configure --prefix=/tools               \
            --with-build-sysroot=/mnt/lfs \
            --target=x86_64-lfs-linux-gnu \
            --disable-nls                 \
            --disable-werror 
make
mkdir -v /tools/lib && ln -sv lib /tools/lib64
make install
```

## Cross-compile GCC

```shell
pacman -S subversion # for MPFR
pacman -S mercurial  # for GMP
```

```shell
git clone git://gcc.gnu.org/git/gcc.git
svn ls https://scm.gforge.inria.fr/anonscm/svn/mpfr/tags
svn co https://scm.gforge.inria.fr/anonscm/svn/mpfr/tags/4.0.1 mpfr-4.0.1
hg clone https://gmplib.org/repo/gmp/ gmp
git clone https://scm.gforge.inria.fr/anonscm/git/mpc/mpc.git 
cd gcc
```
