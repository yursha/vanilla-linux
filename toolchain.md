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
ln -s /tools/lib/crt* /tools/lib/gcc/x86_64-lfs-linux-gnu/9.0.0/
```

## Cross-compile GCC

```shell
pacman -S subversion # for MPFR
pacman -S mercurial  # for GMP
```

```shell
git clone git://gcc.gnu.org/git/gcc.git
git tag
git checkout gcc-8_2_0-release

svn ls -v https://scm.gforge.inria.fr/anonscm/svn/mpfr/tags
svn co -r 12202 https://scm.gforge.inria.fr/anonscm/svn/mpfr/tags/4.0.1 mpfr-4.0.1
# svn ls -v ^/tags
# svn info --show-item revision
mkdir gcc/mpfr
cp -r mpfr-4.0.1/* gcc/mpfr

hg clone https://gmplib.org/repo/gmp/ gmp
hg tags
hg up gmp-6.1.0
mkdir gcc/gmp
cp -r gmp/* gcc/gmp

git clone https://scm.gforge.inria.fr/anonscm/git/mpc/mpc.git 
git tag
git checkout 1.1.0
mkdir gcc/mpc
cp -r mpc/* gcc/mpc
```

Verify, that our linker is for x86-64.

```
readelf -l /bin/vim | grep interpreter
    [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

Apply the following patch so to-be-compiled GCC uses the linker from `/tools`, that
we just built.

```
diff --git a/gcc/config/i386/linux64.h b/gcc/config/i386/linux64.h
index f2d913e30ac..d30bb45c705 100644
--- a/gcc/config/i386/linux64.h
+++ b/gcc/config/i386/linux64.h
@@ -28,7 +28,7 @@ see the files COPYING3 and COPYING.RUNTIME respectively.  If not, see
 #define GNU_USER_LINK_EMULATIONX32 "elf32_x86_64"
 
 #define GLIBC_DYNAMIC_LINKER32 "/lib/ld-linux.so.2"
-#define GLIBC_DYNAMIC_LINKER64 "/lib64/ld-linux-x86-64.so.2"
+#define GLIBC_DYNAMIC_LINKER64 "/tools/lib64/ld-linux-x86-64.so.2"
 #define GLIBC_DYNAMIC_LINKERX32 "/libx32/ld-linux-x32.so.2"
 
 #undef MUSL_DYNAMIC_LINKER32
 ```

Apply the following patch so to-be-compiled GCC installs x86-64 libs to `/lib`
rather than `/lib64`.

```
diff --git a/gcc/config/i386/t-linux64 b/gcc/config/i386/t-linux64
index 8ea0faff369..f25895eed59 100644
--- a/gcc/config/i386/t-linux64
+++ b/gcc/config/i386/t-linux64
@@ -33,6 +33,6 @@
 comma=,
 MULTILIB_OPTIONS    = $(subst $(comma),/,$(TM_MULTILIB_CONFIG))
 MULTILIB_DIRNAMES   = $(patsubst m%, %, $(subst /, ,$(MULTILIB_OPTIONS)))
-MULTILIB_OSDIRNAMES = m64=../lib64$(call if_multiarch,:x86_64-linux-gnu)
+MULTILIB_OSDIRNAMES = m64=../lib$(call if_multiarch,:x86_64-linux-gnu)
 MULTILIB_OSDIRNAMES+= m32=$(if $(wildcard $(shell echo $(SYSTEM_HEADER_DIR))/../../usr/lib32),../lib32,../lib)$(call if_multiarch,:i386-linux-gnu)
 MULTILIB_OSDIRNAMES+= mx32=../libx32$(call if_multiarch,:x86_64-linux-gnux32)
```

Append the following lines to `gcc/config/linux.h`, `gcc/config/i386/linux.h` and `gcc/config/i386/linux64.h`:

```
#undef STANDARD_STARTFILE_PREFIX_1
#undef STANDARD_STARTFILE_PREFIX_2
#define STANDARD_STARTFILE_PREFIX_1 "/tools/lib/"
#define STANDARD_STARTFILE_PREFIX_2 ""
```

```
mkdir build
cd build
../configure                                       \
    --target=x86_64-lfs-linux-gnu                  \
    --prefix=/tools                                \
    --with-glibc-version=2.11                      \
    --with-sysroot=/mnt/lfs                        \
    --with-newlib                                  \
    --without-headers                              \
    --with-local-prefix=/tools                     \
    --with-native-system-header-dir=/tools/include \
    --disable-nls                                  \
    --disable-shared                               \
    --disable-multilib                             \
    --disable-decimal-float                        \
    --disable-threads                              \
    --disable-libatomic                            \
    --disable-libgomp                              \
    --disable-libmpx                               \
    --disable-libquadmath                          \
    --disable-libssp                               \
    --disable-libvtv                               \
    --disable-libstdcxx                            \
    --enable-languages=c,c++
make
make install
```

## Install linux api headers for libc

```
cd linux
git tags
git checkout v4.17
make mrprober
make INSTALL_HDR_PATH=/mnt/lfs/mnt/sdcard/headers-install-output headers_install
cp -rv /mnt/lfs/mnt/sdcard/headers-install-output/include /tools/include
```

TODO: The last two could be one command. Intermediate location is not needed.

## Install libc

```
git clone git://sourceware.org/git/glibc.git
cd glibc
git checkout release/2.28/master
mkdir build
cd build
../configure                             \
      --prefix=/tools                    \
      --host=x86_64-lfs-linux-gnu        \
      --build=x86_64-pc-linux-gnu        \
      --enable-kernel=3.2                \
      --with-headers=/tools/include      \
      libc_cv_forced_unwind=yes          \
      libc_cv_c_cleanup=yes
make
make install
```

## Sanity check

```
sh> echo 'int main(){}' > dummy.c
sh> x86_64-lfs-linux-gnu-gcc dummy.c
sh> readelf -l a.out | grep ': /tools'
    [Requesting program interpreter: /tools/lib64/ld-linux-x86-64.so.2]
sh> rm -v dummy.c a.out
```

## Libstd++

```
cd gcc
mkdir build-libstd++
cd build-libstd++
../libstdc++-v3/configure           \
    --host=x86_64-lfs-linux-gnu     \
    --prefix=/tools                 \
    --disable-multilib              \
    --disable-nls                   \
    --disable-libstdcxx-threads     \
    --disable-libstdcxx-pch         \
    --with-gxx-include-dir=/tools/x86_64-lfs-linux-gnu/include/c++/8.2.0
make
make install
```

## Binutils - second pass

```
cd binutils-gdb/build
rm -rf *
CC=x86_64-lfs-linux-gnu-gcc                \
AR=x86_64-lfs-linux-gnu-ar                 \
RANLIB=x86_64-lfs-linux-gnu-ranlib         \
../configure                   \
    --prefix=/tools            \
    --disable-nls              \
    --disable-werror           \
    --with-lib-path=/tools/lib \
    --with-sysroot
make
make install
```

Readjust the linker

```
make -C ld clean
make -C ld LIB_PATH=/usr/lib:/lib
cp -v ld/ld-new /tools/bin
```
