---
ID: 62734
post_title: 'Building ARM toolchain – part 2: gcc and eglibc'
author: Piotr Król
post_excerpt: ""
layout: post
permalink: >
  http://3mdeb.kleder.co/os-dev/building-arm-toolchain-part-2-gcc-and-eglibc/
published: true
post_date: 2012-04-12 23:54:00
tags:
  - toolchain
  - gcc
  - eglibc
  - arm toolchain
  - arm
  - build
  - embbeded
categories:
  - App Dev
  - OS Dev
---
Unfortunately after few tries of cross compiling eglibc using different source 
for instructions I alway end with hard to solve issues. Luckily, in the sources 
of eglibc I noticed instructions for cross-compiling written long time ago by 
Jim Blandy(I know i should start [here](http://www.eglibc.org/cgi-bin/viewvc.cgi/trunk/libc/EGLIBC.cross-building?revision=2037&view=markup)). 
Lot of thanks to him for it. Below I describe my experience which I gained 
during eglibc cross compilation for `arm-unknown-linux-gnueabi` and procedure that 
I used. Commands below contain some constants that I used in previous works. See 
[this post.](/2012/03/20/building-arm-toolchain-part-1-libs-and) 
Eglibc library and the compiler itself is built with many various parameters 
this post is not the place to explain their meaning, please RTFM.

1. Checkout eglibc from svn (as alwyas I try to use a latest sources possible).
Version used r17815:
```
svn co http://www.eglibc.org/svn/trunk eglibc
```
2. Link working ports to GNU/Linux on some machine architectures. They are not
maintained in the official glibc source tree so we need to add it in this way:
```
ln -s ../ports eglibc/libc/ports/
```
3. Create eglibc-headers directory:
```
mkdir eglib-headers; cd eglib-headers
```
4. Configure eglibc and preliminary objects:
```
BUILD_CC=gcc CC=arm-unknown-linux-gnueabi-gcc CXX=arm-unknown-linux-gnueabi-cpp 
AR=arm-unknown-linux-gnueabi-ar RANLIB=arm-unknown-linux-gnueabi-ranlib 
../eglibc/libc/configure --prefix=/usr --with-headers=$TARGET/usr/include 
--build=x86_64-pc-linux-gnu --host=arm-unknown-linux-gnueabi --disable-profile 
--without-gd --without-cvs --enable-add-ons
```
5. Install eglibc headers:
```
make install-headers install_root=$TARGET install-bootstrap-headers=yes
```
6. We need few object file to link shared libraries, which will be built and installed by hand:
```
mkdir -p $TARGET/usr/lib make csu/subdir_lib cp csu/crt1.o csu/crti.o csu/crtn.o $TARGET/usr/lib
```
7. To produce libgcc_s.so we need libc.so, but only need its dummy version because
we'll never use it. It doesn't matter what we will point as a libc.so we use /dev/null as C file.
```
arm-unknown-linux-gnueabi-gcc -nostdlib -nostartfiles -shared -x c /dev/null -o 
$TARGET/usr/lib/libc.so
```
8. Get latest gcc sources using git repository mirror. Latest commit while writing
this post was 5b9a8c3:
```
cd .. git clone git://repo.or.cz/official-gcc.git
```
9. Now, we can build gcc which can compile eglibc.
```
mkdir eglibc-gcc; cd eglibc-gcc ../official-gcc/configure --target=arm-unknown-linux-gnueabi 
--prefix=$TARGET/arm-x-tools --with-sysroot=$TARGET --disable-libssp 
--disable-libgomp --disable-libmudflap --enable-languages=c 
--with-gmp=$TARGET/arm-x-tools --with-mpfr=$TARGET/arm-x-tools 
--with-mpc=$TARGET/arm-x-tools --disable-libquadmath --build=$MACHTYPE 
--host=$MACHTYPE --with-local-prefix=$TARGET/arm-x-tools --disable-multilib 
--with-float=soft --with-pkgversion=&quot;pietrushnic&quot; --enable-threads=no 
--enable-target-optspace --disable-nls --enable-c99 --enable-long-long
make -j4
make install
```
10. Confugure and compile final version of eglibc.
```
mkdir eglibc-final
cd eglibc-final/ BUILD_CC=gcc CC=arm-unknown-linux-gnueabi-gcc CXX=arm-unknown-linux-gnueabi-cpp 
AR=arm-unknown-linux-gnueabi-ar RANLIB=arm-unknown-linux-gnueabi-ranlib 
../eglibc/libc/configure --prefix=/usr --with-headers=$TARGET/usr/include 
--build=x86_64-pc-linux-gnu --host=arm-unknown-linux-gnueabi --disable-profile 
--without-gd --without-cvs --enable-add-ons
make
make install install_root=$TARGET
```
11. Install libelf library
```
wget http://www.mr511.de/software/libelf-0.8.13.tar.gz
tar zxvf libelf-0.8.13.tar.gz
cd libelf-0.8.13/
./configure --prefix=$TARGET/arm-x-tools --disable-shared --enable-static
make;make install
```
12. Prepare final version of gcc.
```
cd ..
mkdir final-gcc
cd final-gcc
../official-gcc/configure --target=arm-unknown-linux-gnueabi 
--prefix=$TARGET/arm-x-tools --with-sysroot=$TARGET --disable-libssp 
--disable-libgomp --disable-libmudflap --enable-languages=c,c++ --with-gmp=$TARGET/arm-x-tools 
--with-mpfr=$TARGET/arm-x-tools --with-mpc=$TARGET/arm-x-tools --disable-libquadmath 
--build=$MACHTYPE --host=$MACHTYPE --with-local-prefix=$TARGET/arm-x-tools --disable-multilib 
--with-float=soft --with-pkgversion=&quot;pietrushnic&quot; --enable-threads=posix 
--enable-target-optspace --disable-nls --enable-c99 --enable-long-long 
--enable-__cxa_atexit --enable-symvers=gnu --with-libelf=$TARGET/arm-x-tools 
--enable-lto
make
make install
```
13. Few libraries should be copied manualy
```
cp -d $TARGET/arm-x-tools/arm-unknown-linux-gnueabi/lib/libgcc_s.so* $TARGET/lib
cp -d $TARGET/arm-x-tools/arm-unknown-linux-gnueabi/lib/libstdc++.so* $TARGET/lib
```
14. Compile and install chrpath - this is useful tool to remove the rpath or runpath setting from binary.
```
cd ..
sudo apt-get install libc6-i386 gcc-multilib
apt-get source chrpath
cd chrpath-0.13/ CFLAGS=-m32
./configure --prefix=$TARGET/arm-x-tools 
--program-prefix=arm-unknown-linux-gnueabi-
make
make install
```
15. Strip debug symbols
```
strip --strip-debug $TARGET/arm-x-tools/lib/* 
$TARGET/arm-x-tools/arm-unknown-linux-gnueabi/lib/* $TARGET/arm-x-tools/libexec/*
strip --strip-unneeded $TARGET/arm-x-tools/bin/* 
$TARGET/arm-x-tools/arm-unknown-linux-gnueabi/bin/*
arm-unknown-linux-gnueabi-strip --strip-debug $TARGET/lib/* $TARGET/usr/lib/*
```
16. At the end simple test to find out if basic functionality works:
```c
cat &gt; hello.c &lt;&lt; EOF 
&gt; #include &lt;stdio.h&gt;
&gt; int
&gt; main (int argc, char **argv) 
&gt; { 
&gt; puts (&quot;Hello, world!&quot;); 
&gt; return 0; 
&gt; } 
&gt; EOF
```
Try to cross compile C file:
```
$TARGET/arm-x-tools/bin/arm-unknown-linux-gnueabi-gcc -Wall hello.c -o hello
```
```cpp
cat &gt; c++-hello.cc &lt;&lt;EOF 
&gt; #include &lt;iostream&gt; 
&gt; int 
&gt; main (int argc, char **argv) 
&gt; { 
&gt; std::cout return 0; 
&gt; } 
&gt; EOF
Try to cross compile C++ file:
```
$TARGET/arm-x-tools/bin/arm-unknown-linux-gnueabi-g++ -Wall c++-hello.cc -o 
c++-hello
```
Displays the information contained in the ELF header and in the file&#039;s segment headers:
```
$TARGET/arm-x-tools/bin/arm-unknown-linux-gnueabi-readelf -hl hello
$TARGET/arm-x-tools/bin/arm-unknown-linux-gnueabi-readelf -hl c++-hello
```
Result should look like that:
```
ELF Header:
  Magic: 7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class: ELF32
  Data: 2's complement, little endian
  Version: 1 (current)
  OS/ABI: UNIX - System V ABI
  Version: 0
  Type: EXEC (Executable file)
  Machine: ARM
  (...)
  Flags: 0x5000002, has entry point, Version5 EABI
  (...)
  Program Headers:
  (...)
  INTERP 0x000134 0x00008134 0x00008134 0x00013 0x00013 R 0x1
      [Requesting program interpreter: /lib/ld-linux.so.3]
  LOAD 0x000000 0x00008000 0x00008000 0x004b8 0x004b8 R E 0x8000
  (...)
  ```
```
$TARGET/arm-x-tools/bin/arm-unknown-linux-gnueabi-readelf -d 
$TARGET/lib/libgcc_s.so.1
```
Result should look like that:
```
(...)
Tag          Type           Name/Value
0x00000001 (NEEDED) Shared library: [libc.so.6]
0x0000000e (SONAME) Library soname: [libgcc_s.so.1]
0x0000000c (INIT) 0xcc2c (...)
```

I hope you find above manual useful. If you need more detailed descriptions it 
can be found [here](http://www.eglibc.org/cgi-bin/viewvc.cgi/trunk/libc/EGLIBC.cross-building?view=markup).
Also don't bother to ask me by comment below.