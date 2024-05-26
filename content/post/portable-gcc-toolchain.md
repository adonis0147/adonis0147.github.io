---
title: "Build a Portable GCC Toolchain"
date: 2024-05-09T16:49:12+08:00
draft: false
summary: A guide to build a portable GCC toolchain
tags: ["GCC", "x86_64", "aarch64"]
categories: ["English"]
---

This post is a guide to build a GCC toolchain. The GCC toolchain is **portable** and **self-contained** 
(_the only dependency is the Linux kernel_).

I use [docker](https://www.docker.com/) to set the environment up.

**_TL;DR_**: Just download the all-in-one script in the [release page](https://github.com/adonis0147/devel-env/releases)
and [use it](/post/portable-gcc-toolchain/#use-the-all-in-one-script).

# Prerequisite

- [Docker](https://www.docker.com/)
- [Binutils 2.42](https://ftpmirror.gnu.org/binutils/binutils-2.42.tar.xz)
- [Linux Kernel 6.6.30](https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.6.30.tar.xz)
- [glibc 2.39](https://ftpmirror.gnu.org/glibc/glibc-2.39.tar.xz)
- [GCC 14.1](https://ftpmirror.gnu.org/gcc/gcc-14.1.0/gcc-14.1.0.tar.xz)
- [libxcrypt 4.4.36](https://github.com/besser82/libxcrypt/releases/download/v4.4.36/libxcrypt-4.4.36.tar.xz)
- [patchelf 0.16.1](https://github.com/NixOS/patchelf/releases/tag/0.16.1)

## Preparation

```shell
# Use ubuntu:22.04 image
docker run -it ubuntu:22.04 bash

cd /root

apt update && apt upgrade --yes

# Install dependencies
DEBIAN_FRONTEND=noninteractive apt install --yes \
    build-essential texinfo bison git curl rsync gawk \
    python-is-python3 help2man file

# Download sources
mkdir packages
curl -L 'https://ftpmirror.gnu.org/binutils/binutils-2.42.tar.xz' \
    -o packages/binutils-2.42.tar.xz

curl -L 'https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.6.30.tar.xz' \
    -o packages/linux-6.6.30.tar.xz

curl -L 'https://ftpmirror.gnu.org/glibc/glibc-2.39.tar.xz' \
    -o packages/glibc-2.39.tar.xz

curl -L 'https://ftpmirror.gnu.org/gcc/gcc-14.1.0/gcc-14.1.0.tar.xz' \
    -o packages/gcc-14.1.0.tar.xz

curl -L 'https://github.com/besser82/libxcrypt/releases/download/v4.4.36/libxcrypt-4.4.36.tar.xz' \
    -o packages/libxcrypt-4.4.36.tar.xz
```

The layout in the disk:

```shell
packages/
|-- binutils-2.42.tar.xz
|-- gcc-14.1.0.tar.xz
|-- glibc-2.39.tar.xz
|-- libxcrypt-4.4.36.tar.xz
`-- linux-6.6.30.tar.xz
```

Extract all sources.

```shell
find packages/ -mindepth 1 -exec tar -Jxf {} \;
```

```shell
# Layout
.
|-- binutils-2.42
|-- gcc-14.1.0
|-- glibc-2.39
|-- libxcrypt-4.4.36
|-- linux-6.6.30
`-- packages
```

## Overview

1. Build and install cross GCC toolchain (**NOTE**: the _**corss**_ here is not accurate, because the target architecture is
   the same as the build one).

    * Build and install cross Binutils.
    * Install Linux Kernel headers.
    * Build stage1 cross GCC
        - Build and install `gcc`
    * Build stage1 glibc
        - Build and install headers `stubs.h` and libraries `crt1.o` `crti.o` `crtn.o` `libc.so` ...
    * Build stage2 cross GCC
        - Build and install `libgcc`
    * Build and install final glibc
    * Build and install final cross GCC

2. Build final GCC toolchain by the cross GCC toolchain.
3. Build and install libxcrypt by the final GCC toolchain.

## Environment

```shell
export PREFIX='/opt/toolchain'
export TARGET='x86_64-linux-gnu'
export CROSS_PREFIX="${PREFIX}/cross"
export TARGET_PREFIX="${CROSS_PREFIX}/${TARGET}"
export ARCH='x86'
export PATH="/opt/toolchain/bin:/opt/toolchain/cross/bin:${PATH}"
```

**NOTE**: The path `/opt/toolchain/cross` is temporary and will be removed after setting all up. The final path is
`/opt/toolchain`.

```shell
mkdir -p "${CROSS_PREFIX}"
ln -snf .. "${CROSS_PREFIX}/${TARGET}"

mkdir -p "${TARGET_PREFIX}/lib"
ln -snf "lib" "${TARGET_PREFIX}/lib64"
```

```shell
# Layout
/opt/
`-- toolchain
    |-- cross
    |   `-- x86_64-linux-gnu -> ..
    |-- lib
    `-- lib64 -> lib
```

# Build and install cross GCC toolchain

1. Build and install cross Binutils.

```shell
pushd binutils-2.42

./configure --prefix="${CROSS_PREFIX}" \
    --target="${TARGET}" \
    --disable-multilib
make -j "$(nproc)"
make install

popd
```

* We specified `x86_64-linux-gnu` as the target system type.
* `--disable-multilib` means that we only want our Binutils installation to work with programs and libraries using the
64-bits instruction set.

2. Install Linux Kernel headers.

```shell
pushd linux-6.6.30

make ARCH="${ARCH}" INSTALL_HDR_PATH="${TARGET_PREFIX}" headers_install

popd
```

* We installed the headers to `/opt/toolchain/include`.

3. Build stage1 cross GCC

```shell
pushd gcc-14.1.0

# Download prerequisites
contrib/download_prerequisites

mkdir build
cd build

../configure --prefix="${CROSS_PREFIX}" \
    --target="${TARGET}" \
    --libdir="${TARGET_PREFIX}/lib" \
    --libexecdir="${TARGET_PREFIX}/libexec" \
    --with-local-prefix="${TARGET_PREFIX}" \
    --with-gxx-include-dir="${TARGET_PREFIX}/include/c++" \
    --enable-languages=c,c++ \
    --enable-default-pie \
    --disable-multilib \
    --disable-libssp \
    --disable-threads \
    --disable-libatomic \
    --disable-libffi \
    --disable-libgomp \
    --disable-libitm \
    --disable-libquadmath \
    --disable-libquadmath-support \
    --disable-libsanitizer \
    --disable-bootstrap
make -j "$(nproc)" all-gcc
make install-gcc

popd
```

* We specified `x86_64-linux-gnu` as the target system type.
* We installed the executables to `/opt/toolchain/cross/bin` and installed headers and libraries to `/opt/toolchain/include`
and `/opt/toolchain/lib` respectively.
* We enabled `C` and `C++` and disabled many libraries to accelerate the build time.

4. Build stage1 glibc

```shell
pushd glibc-2.39

mkdir build
cd build

../configure --prefix="${TARGET_PREFIX}" \
    --build="${MACHTYPE}" \
    --host="${TARGET}" \
    --target="${TARGET}" \
    --with-headers="${TARGET_PREFIX}"/include \
    --disable-multilib \
    libc_cv_forced_unwind=yes
make install-bootstrap-headers=yes install-headers
make -j "$(nproc)" csu/subdir_lib

install csu/crt1.o csu/crti.o csu/crtn.o "${TARGET_PREFIX}"/lib
"${TARGET}-gcc" -nostdlib -nostartfiles -shared -x c /dev/null -o "${TARGET_PREFIX}"/lib/libc.so
touch "${TARGET_PREFIX}"/include/gnu/stubs.h

popd
```

* We installed the libraries to `/opt/toolchain/lib`.
* We installed the C library’s startup files, `crt1.o`, `crti.o` and `crtn.o` to the installation directory manually.
There’s doesn’t seem to a make rule that does this without having other side effects.
* We created a couple of dummy files, `libc.so` and `stubs.h`, which are expected in step `#5`, but which will be
replaced by step `#6`.

5. Build stage2 cross GCC

```shell
pushd gcc-14.1.0

cd build

make -j "$(nproc)" all-target-libgcc
make install-target-libgcc

popd
```

6. Build and install final glibc

```shell
pushd glibc-2.39

cd build

make -j "$(nproc)"
make install

popd
```

7. Build and install final cross GCC

```shell
pushd gcc-14.1.0

cd build

make -j "$(nproc)"
make install

popd
```

## Validation

Now, we have a corss GCC toolchain, we can test the toolchain whether it works as expect.

```hello.cc
#include <iostream>

auto main() -> int {
    std::cout << "Hello, world!" << std::endl;
    return 0;
}
```

```shell
$ x86_64-linux-gnu-g++ -Wl,-rpath,/opt/toolchain/lib \
    -Wl,-dynamic-linker,/opt/toolchain/lib/ld-linux-x86-64.so.2 \
    hello.cc -o hello
$ ./hello
Hello, world!
```

It works!

# Build final GCC toolchain by the cross GCC toolchain.

1. Build and install final Binutils.
```shell
rm -rf binutils-2.42
tar -Jxf packages/binutils-2.42.tar.xz

pushd binutils-2.42

LDFLAGS="-L${PREFIX}/lib -Wl,-rpath,${PREFIX}/lib \
    -Wl,-dynamic-linker,$(find "${PREFIX}/lib" -name 'ld-linux-*')" \
    ./configure --prefix="${PREFIX}" \
    --host="${TARGET}" \
    --enable-gold \
    --enable-plugins \
    --disable-multilib
make -j "$(nproc)"
make install-strip

# Remove the old files
rm -rf "${PREFIX}/lib/ldscripts"

popd
```

2. Build and install final GCC

```shell
rm -rf gcc-14.1.0
tar -Jxf packages/gcc-14.1.0.tar.xz

# Download prerequisites
pushd gcc-14.1.0

contrib/download_prerequisites

mkdir build
cd build

ldflags="-L${PREFIX}/lib -Wl,-rpath,${PREFIX}/lib \
    -Wl,-dynamic-linker,$(find "${PREFIX}/lib" -name 'ld-linux-*')"

LDFLAGS="${ldflags}" LDFLAGS_FOR_TARGET="${ldflags}" \
    ../configure --prefix="${PREFIX}" \
    --host="${TARGET}" \
    --with-local-prefix="${PREFIX}" \
    --enable-languages=c,c++ \
    --enable-default-pie \
    --disable-multilib

make BOOT_LDFLAGS="${ldflags}" -j "$(nproc)"

# Remove the old files
rm -rf "${PREFIX}/include/c++"

make install-strip

popd
```

3. Build and install final glibc

```shell
rm -rf glibc-2.39
tar -Jxf packages/glibc-2.39.tar.xz

pushd glibc-2.39

mkdir build
cd build

../configure --prefix="${PREFIX}" \
    --with-headers="${PREFIX}"/include \
    --disable-multilib
make -j "$(nproc)"
make install

popd
```

4. Build and install libxcrypt

```shell
pushd libxcrypt-4.4.36

./configure --prefix="${PREFIX}"
make -j "$(nproc)"
make install

rm -rf "${PREFIX}/lib/pkgconfig"

popd
```

# Use the GCC toolchain seamlessly

We can use the GCC toolchain like this.

```shell
$ g++ -Wl,-rpath,/opt/toolchain/lib \
    -Wl,-dynamic-linker,/opt/toolchain/lib/ld-linux-x86-64.so.2 \
    hello.cc -o hello
$ ./hello
Hello, world!
```

However, it is verbose. We can use GCC specs file to simplify the usage.

```shell
toolchain_home='/opt/toolchain'
interpreter="$(find "${toolchain_home}" -name "ld-linux-*")"
filename="$(basename "${interpreter}")"
dirname="$(dirname "${interpreter}")"

"${toolchain_home}/bin/gcc" -dumpspecs | sed "{
    s/:\([^;}:]*\)\/\(${filename/.so*/}\)/:${dirname//\//\\/}\/\2/g
    s/collect2/collect2 -rpath ${dirname//\//\\/}/
}" >"${toolchain_home}/lib/gcc/$(uname -m)-linux-gnu/specs"
```

## Validation

```shell
$ gcc -v
Using built-in specs.
Reading specs from /opt/toolchain/lib/gcc/x86_64-linux-gnu/specs
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/opt/toolchain/libexec/gcc/x86_64-linux-gnu/14.1.0/lto-wrapper
Target: x86_64-linux-gnu
Configured with: ../configure --prefix=/opt/toolchain --host=x86_64-linux-gnu --with-local-prefix=/opt/toolchain --enable-languages=c,c++ --enable-default-pie --disable-multilib
Thread model: posix
Supported LTO compression algorithms: zlib
gcc version 14.1.0 (GCC)
```

Now, `GCC` uses the specs file `/opt/toolchain/lib/gcc/x86_64-linux-gnu/specs` which was created by us.

```shell
$ g++ hello.cc -o hello
$ ./hello
Hello, world!
```

It works!

# Make GCC toolchain portable by patchelf

So far, we have built our GCC toolchain. In this section, we use `patchelf` to make the toolchain portable.

```shell
mkdir -p /opt/patchelf

curl -L "https://github.com/NixOS/patchelf/releases/download/0.16.1/patchelf-0.16.1-$(uname -m).tar.gz" \
    -o - | tar -zxv -C /opt/patchelf

export PATH="/opt/patchelf/bin:${PATH}"
```

We move the GCC toolchain to a different location to pretend that we extracted the toolchain to a new machine.

```shell
mv /opt/toolchain /root/
```

We will find the executable `GCC` can't be executed.

```shell
$ cd /root/toolchain/bin
$ ./gcc --version
bash: ./gcc: No such file or directory
```

We can use the tool `patchelf` to fix this issue.

```shell
$ patchelf --set-rpath /root/toolchain/lib gcc
$ patchelf --set-interpreter /root/toolchain/lib/ld-linux-x86-64.so.2 gcc
$ ./gcc --version
gcc (GCC) 14.1.0
Copyright (C) 2024 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

For the sake of convenience, we can use the following function.

```bash
function relocate() {
	local opts
	local overwrite=false
	if ! opts="$(getopt -o '' -l 'overwrite' -- "${@}")"; then
		return 1
	fi
	eval "set -- ${opts}"

	while true; do
		case "${1}" in
		--overwrite)
			overwrite=true
			shift
			;;
		--)
			shift
			break
			;;
		*)
			echo 'Invalid arguments'
			return 1
			;;
		esac
	done

	local search_path="${1}"
	local base_path="${2}"
	local libc_so
	local library_path
	local interpreter
	libc_so="$(find "${search_path}" -name 'libc.so.6')"
	library_path="$(dirname "${libc_so}")"
	interpreter="$(find "${search_path}" -name 'ld-linux-*.so.*')"

	local old_rpath
	local new_rpath
	while read -r file; do
		if old_rpath="$(patchelf --print-rpath "${file}" 2>/dev/null)"; then
			if ${overwrite}; then
				old_rpath="\$ORIGIN:\$ORIGIN/lib:\$ORIGIN/lib64:\$ORIGIN/../lib:\$ORIGIN/../lib64"
			fi
			new_rpath="${old_rpath:+${old_rpath}:}${library_path}"
			patchelf --set-rpath "${new_rpath}" "${file}"
			if readelf -S "${file}" | grep '.interp' >/dev/null; then
				patchelf --set-interpreter "${interpreter}" "${file}"
			fi
		fi
	done < <(find "$(readlink -f "${base_path}")" -type f)
}
```

```shell
$ ./g++ --version
bash: ./g++: No such file or directory

$ relocate --overwrite /root/toolchain g++

$ patchelf --print-rpath g++
$ORIGIN:$ORIGIN/lib:$ORIGIN/lib64:$ORIGIN/../lib:$ORIGIN/../lib64:/root/toolchain/lib

$ patchelf --print-interpreter g++
/root/toolchain/lib/ld-linux-x86-64.so.2

$ ./g++ --version
g++ (GCC) 14.1.0
Copyright (C) 2024 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

Next, we fix the whole GCC toolchain by apply the following function.

```bash
function configure_toolchain() {
	local PATCHELF='patchelf'
	local LIBC_SO='libc.so.6'
	local INTERPRETER='ld-linux-*'
	local READELF='readelf'

	local prefix="${1}"
	echo -e 'Configuring the toolchain...'
	if ! cd "${prefix}"; then
		echo -e "Failed to change the current directory to \033[34;1m${prefix}/${TOOLCHAIN_DIRNAME}\033[0m ."
	fi

	local patchelf
	patchelf="$(command -v "${PATCHELF}")"
	if [[ ! -f "${patchelf}" ]]; then
		echo -e "Failed to find the tool \033[34;1m${patchelf}\033[0m ."
	fi
	echo -e "Program ${PATCHELF} was found: \033[34;1m${patchelf}\033[0m"

	local readelf="/usr/bin/readelf"
	if [[ ! -f "${readelf}" ]]; then
		echo -e "Failed to find the tool \033[34;1m${readelf}\033[0m ."
	fi
	echo -e "Program ${READELF} was found: \033[34;1m${readelf}\033[0m"

	local interpreter
	if ! interpreter="$(find "$(pwd)" -name "${INTERPRETER}")" || [[ -z "${interpreter}" ]]; then
		echo -e "Failed to find the interpreter \033[34;1m${INTERPRETER}\033[0m"
	fi
	echo -e "Interpreter was found: \033[34;1m${interpreter}\033[0m"

	local libc_so
	if ! libc_so="$(find "$(pwd)" -name "${LIBC_SO}")" || [[ -z "${libc_so}" ]]; then
		echo -e "Failed to find the library \033[34;1m${LIBC_SO}\033[0m"
	fi
	echo -e "Library was found: \033[34;1m${libc_so}\033[0m"

	# Remove useless files
	rm -rf "$(pwd)/$(uname -m)-linux-gnu"/*
	find "$(pwd)" -name '*.la' -delete

	mkdir "$(pwd)/$(uname -m)-linux-gnu/include"
	mkdir "$(pwd)/$(uname -m)-linux-gnu/lib"
	ln -snf lib "$(pwd)/$(uname -m)-linux-gnu/lib64"

	# Link headers
	local include_path
	include_path="$(pwd)/include"
	pushd "$(uname -m)-linux-gnu/include" >/dev/null || exit
	while read -r path; do
		local absolute_path="${include_path}/${path}"
		if [[ -d "${absolute_path}" ]]; then
			mkdir -p "${path}"
		else
			ln -snf "${absolute_path}" "${path}"
		fi
	done < <(find "${include_path}" -mindepth 1 -printf '%P\n')
	popd >/dev/null || exit

	local rpaths=(
		"\$ORIGIN"
		"\$ORIGIN/lib"
		"\$ORIGIN/lib64"
		"\$ORIGIN/../lib"
		"\$ORIGIN/../lib64"
		"$(pwd)/$(uname -m)-linux-gnu/lib"
		"$(dirname "${libc_so}")"
	)
	local rpaths_in_line
	rpaths_in_line="$(
		IFS=':'
		echo "${rpaths[*]}"
	)"

	while read -r file; do
		if ! file "${file}" | grep ELF >/dev/null; then
			continue
		fi
		"${patchelf}" --set-rpath "${rpaths_in_line}" "${file}"
		if "${readelf}" -S "${file}" | grep '.interp' >/dev/null; then
			"${patchelf}" --set-interpreter "${interpreter}" "${file}"
		fi
	done < <(
		find "$(pwd)" -type f ! -name '*.o' ! -name '*.a' \
			! -name "${PATCHELF}" ! -name "${INTERPRETER}" ! -name "${LIBC_SO}"
	)
	"${patchelf}" --set-interpreter "${interpreter}" "${libc_so}"

	pushd bin >/dev/null || exit
	ln -snf gcc cc
	popd >/dev/null || exit

	# Modify files
	local current_path
	current_path="$(pwd)"
	while read -r file; do
		sed -i "s/\/opt\/toolchain/${current_path//\//\\/}/g" "${file}"
	done < <(
		find . -type f -exec grep -E -I -l $'[=\'" ]/opt/toolchain' {} \;
	)

	# Modify ldd
	sed -i "/RTLDLIST=/s/${current_path//\//\\/}//g" bin/ldd

	# Configure gcc specs
	local filename
	filename="$(basename "${interpreter}")"
	local dirname
	dirname="$(dirname "${interpreter}")"
	"$(pwd)/bin/gcc" -dumpspecs | sed "{
		s/:\([^;}:]*\)\/\(${filename/.so*/}\)/:${dirname//\//\\/}\/\2/g
		s/collect2/collect2 -rpath ${rpaths_in_line//\//\\/}/
	}" >"$(pwd)/lib/gcc/specs"
}
```

```shell
$ configure_toolchain /root/toolchain
```

## Validation

```shell
$ export PATH="/root/toolchain/bin:${PATH}"
$ cd /root/
$ g++ hello.cc -o hello
$ ./hello
Hello, world!
```

It works!

# Use the all-in-one script

I made an all-in-one script to install a portable GCC toolchain. See [the release page](https://github.com/adonis0147/devel-env/releases).

## Example

```shell
curl -LO 'https://github.com/adonis0147/devel-env/releases/download/v2024.05/install_toolchain_x86_64.sh'

bash install_toolchain_x86_64.sh /root

export PATH="/root/compiler/bin:${PATH}"
```

# References

1. [How to Build a GCC Cross-Compiler](https://preshing.com/20141119/how-to-build-a-gcc-cross-compiler/)

