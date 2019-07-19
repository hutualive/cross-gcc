TARGET = arm-linux-musleabihf
OUTPUT = $(PWD)/output

GCC_CONFIG =
GCC_CONFIG_FOR_TARGET =

COMMON_CONFIG += CFLAGS="-g0 -Os" CXXFLAGS="-g0 -Os" LDFLAGS="-s"
COMMON_CONFIG += --disable-nls
GCC_CONFIG += --enable-languages=c,c++
GCC_CONFIG += --disable-libquadmath --disable-decimal-float
GCC_CONFIG += --disable-multilib

GCC_SRCDIR = $(PWD)/gcc 
BINUTILS_SRCDIR = $(PWD)/binutils
MUSL_SRCDIR = $(PWD)/musl
LINUX_SRCDIR = $(PWD)/src/linux

XGCC_DIR = $(PWD)/obj_gcc/gcc
XGCC = $(XGCC_DIR)/xgcc -B $(XGCC_DIR)

MAKE_ARG += MULTILIB_OSDIRNAMES=
MAKE_ARG += INFO_DEPS= infodir=
MAKE_ARG += ac_cv_prog_lex_root=lex.yy.c
MAKE_ARG += MAKEINFO=false

SYSROOT = /$(TARGET)

FULL_BINUTILS_CONFIG = \
	--disable-separate-code \
	$(COMMON_CONFIG) \
	--disable-werror \
	--target=$(TARGET) --prefix= \
	--libdir=/lib --disable-multilib \
	--with-sysroot=$(SYSROOT) \
	--enable-deterministic-archives

FULL_GCC_CONFIG = --enable-languages=c,c++ \
	$(GCC_CONFIG_FOR_TARGET) \
	$(COMMON_CONFIG) $(GCC_CONFIG) \
	--disable-werror \
	--target=$(TARGET) --prefix= \
	--libdir=/lib --disable-multilib \
	--with-sysroot=$(SYSROOT) \
	--enable-tls \
	--disable-libmudflap --disable-libsanitizer \
	--disable-gnu-indirect-function \
	--disable-libmpx \
	--enable-libstdcxx-time

FULL_MUSL_CONFIG = $(MUSL_CONFIG) \
	--prefix= --host=$(TARGET)

FULL_GCC_CONFIG += --with-build-sysroot=$(PWD)/obj_sysroot \
	AR_FOR_TARGET=$(PWD)/obj_binutils/binutils/ar \
	AS_FOR_TARGET=$(PWD)/obj_binutils/gas/as-new \
	LD_FOR_TARGET=$(PWD)/obj_binutils/ld/ld-new \
	NM_FOR_TARGET=$(PWD)/obj_binutils/binutils/nm-new \
	OBJCOPY_FOR_TARGET=$(PWD)/obj_binutils/binutils/objcopy \
	OBJDUMP_FOR_TARGET=$(PWD)/obj_binutils/binutils/objdump \
	RANLIB_FOR_TARGET=$(PWD)/obj_binutils/binutils/ranlib \
	READELF_FOR_TARGET=$(PWD)/obj_binutils/binutils/readelf \
	STRIP_FOR_TARGET=$(PWD)/obj_binutils/binutils/strip-new 

FULL_MUSL_CONFIG += CC="$(XGCC)" LIBCC="$(PWD)/obj_gcc/$(TARGET)/libgcc/libgcc.a" 
MUSL_VARS = AR=$(PWD)/obj_binutils/binutils/ar RANLIB=$(PWD)/obj_binutils/binutils/ranlib

obj_musl/.lc_configured: | obj_gcc/gcc/.lc_built
obj_musl/.lc_built: | obj_gcc/$(TARGET)/libgcc/libgcc.a

obj_gcc/gcc/.lc_built: | obj_sysroot/usr obj_sysroot/lib32 obj_sysroot/lib64 obj_sysroot/include
obj_gcc/.lc_built: | obj_sysroot/.lc_libs obj_sysroot/.lc_headers
obj_gcc/.lc_configured: obj_binutils/.lc_built

all: musl gcc binutils

toolchain: gcc binutils

install: install-musl install-gcc install-binutils

install-toolchain: install-gcc install-binutils

binutils: obj_binutils/.lc_built

gcc: obj_gcc/.lc_built

musl: obj_musl/.lc_built

.PHONY: all musl toolchain install-musl install-toolchain clean

obj_sysroot/include:
	mkdir -p $@

obj_sysroot/usr: | obj_sysroot
	ln -sf . $@

obj_sysroot/lib32: | obj_sysroot
	ln -sf lib $@

obj_sysroot/lib64: | obj_sysroot
	ln -sf lib $@

obj_binutils/.lc_configured: | obj_binutils binutils
	cd obj_binutils && ../binutils/configure $(FULL_BINUTILS_CONFIG)
	touch $@

obj_binutils/.lc_built: | obj_binutils/.lc_configured
	cd obj_binutils && make all
	touch $@

obj_gcc/.lc_configured: | obj_gcc gcc
	cd obj_gcc && ../gcc/configure $(FULL_GCC_CONFIG)
	touch $@

obj_gcc/gcc/.lc_built: | obj_gcc/.lc_configured
	cd obj_gcc && make all-gcc
	touch $@

obj_musl/.lc_configured: | obj_musl musl
	cd obj_musl && ../musl/configure $(FULL_MUSL_CONFIG)
	touch $@

obj_sysroot/.lc_headers: | obj_musl/.lc_configured obj_sysroot
	cd obj_musl && make DESTDIR=$(PWD)/obj_sysroot install-headers
	touch $@

obj_gcc/$(TARGET)/libgcc/libgcc.a: | obj_sysroot/.lc_headers
	cd obj_gcc && make enable_shared=no all-target-libgcc

obj_musl/.lc_built: | obj_musl/.lc_configured
	cd obj_musl && make $(MUSL_VARS)
	touch $@

obj_sysroot/.lc_libs: | obj_musl/.lc_built
	cd obj_musl && make $(MUSL_VARS) DESTDIR=$(PWD)/obj_sysroot install
	touch $@

obj_gcc/.lc_built: | obj_gcc/.lc_configured obj_gcc/gcc/.lc_built
	cd obj_gcc && make
	touch $@

install-musl: | obj_musl/.lc_built
	cd obj_musl && make $(MUSL_VARS) DESTDIR=$(OUTPUT)$(SYSROOT) install

install-binutils: | obj_binutils/.lc_built
	cd obj_binutils && make DESTDIR=$(OUTPUT) install

install-gcc: | obj_gcc/.lc_built
	cd obj_gcc && make DESTDIR=$(OUTPUT) install
	ln -sf $(TARGET)-gcc $(OUTPUT)/bin/$(TARGET)-cc

ifneq ($(LINUX_SRCDIR),)
TARGET_ARCH = $(firstword $(subst -, ,$(TARGET)))
TARGET_ARCH_MANGLED = $(patsubst i%86,x86,$(patsubst aarch64%,arm64%,$(TARGET_ARCH)))
LINUX_ARCH_LIST = $(sort $(notdir $(wildcard $(LINUX_SRCDIR)/arch/*)))
LINUX_ARCH = $(lastword $(foreach a,$(LINUX_ARCH_LIST),$(if $(filter $(a)%,$(TARGET_ARCH_MANGLED)),$(a))))
ifneq ($(LINUX_ARCH),)
all: kernel-headers
install: install-kernel-headers
kernel-headers: | obj_kernel_headers/.lc_built
obj_kernel_headers/.lc_built: | linux
	mkdir -p $(PWD)/obj_kernel_headers/staged
	cd linux && make ARCH=$(LINUX_ARCH) O=$(PWD)/obj_kernel_headers INSTALL_HDR_PATH=$(PWD)/obj_kernel_headers/staged headers_install
	find obj_kernel_headers/staged/include '(' -name .install -o -name ..install.cmd ')' -exec rm {} +
	touch $@
install-kernel-headers: | obj_kernel_headers/.lc_built
	mkdir -p $(OUTPUT)$(SYSROOT)/include
	cp -R obj_kernel_headers/staged/include/* $(OUTPUT)$(SYSROOT)/include
endif
endif

clean:
	rm -rf obj_*
