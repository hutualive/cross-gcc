# introduction

the idea is to understand how to build a cross compiler to stay edge(like gcc 9.1) or bootstrap a target(like arm) with your favorite distribution(like alpine linux).

to understand the principal behind, you do not need depends on the "blackbox" generated by auto build tools like yocto, buildroot, crosstool-ng etc.

# gcc target glibc

the best reference I find for background info is:

how a toolchain is constructed  -->

https://crosstool-ng.github.io/docs/toolchain-construction/

for step by step guide:

how to build a gcc cross-compiler  -->

https://preshing.com/20141119/how-to-build-a-gcc-cross-compiler/

the guide is for armv8(aarch64) with tuple aarch64-linux-gnu

for armv7(arm), you need tweak with tuple arm-linux-gnueabihf, as latest armv7 based soc typically with hard float point unit(fpu - vfpv4-d16)

# gcc target musl

manually build gcc target musl is tough, as you can find very few documents...

the best reference I find is:

simple makefile-based build for musl cross compiler

https://github.com/richfelker/musl-cross-make

it's a build script and support many targets, for our purpose, I suggest refer to the 1st link(how a toolchain is constructed) first. when you understand the principal, it's easier to understand the script under the hood.

here my experimental target tuple is arm-linux-musleabihf as I have a stm32mp1 board(armv7 based with vfpv4-d16/neon).

to summarize here again for the steps:

1. gcc need pre-requests like gmp, mpfr, mpc, isl, cloog for math and optimation
2. gcc need binutils for assembler and linker
3. gcc need libc to use the system(userspace to linux kernel)
4. libc need the linux kernel headers(system calls, data structures etc.)
5. build libc need gcc
6. to jailbreak, need a gcc without need of libc
7. typically breakdown to two stages build: bare minimal gcc and 2nd stage gcc
8. build bare minimal gcc to build c runtime for libc with libc and kernel headers
9. build 2nd stage gcc to build libc
10. build full gcc  

for musl case, the script is single stage build. once gcc_toolchain_source(gcc with gmp, mpfr, mpc, isl, cloog + binutils) prepared, build gcc(xgcc) to build musl(libgcc.a), then build musl libc.a and libc.so, then final "make all" build the full gcc  -->

1. host gcc build out xgcc
2. xgcc build out libgcc.a
3. xgcc build out libc.a and libc.so
4. host gcc build out arm-linux-musleabihf-gcc  
