---
title: "Wrapper_c - Yet Another Clang's JSON Compilation Database Generator"
date: 2022-06-05T16:47:56+08:00
draft: false
summary: An introduction to [wrapper_c](https://github.com/adonis0147/wrapper_c).
tags: ["wrapper_c", "compile_commands.json", "PostgreSQL"]
categories: ["English"]
---

_**WRAPPER_C**_ is a compiler wrapper. It is another 
[Clang's JSON compilation database](https://clang.llvm.org/docs/JSONCompilationDatabase.html) 
generator for **make-based** build systems.

Unlike [Bear](https://github.com/rizsotto/Bear) which uses **LD_PRELOAD**, 
wrapper_c exploits the implicit variables `CC` and `CXX` to intercept the compilation commands 
from `make` and forwards the commands to the compiler.

In this post, I will show how to use [wrapper_c](https://github.com/adonis0147/wrapper_c) to 
generate `compile_commands.json` for [PostgreSQL](https://www.postgresql.org/).

I use [docker](https://www.docker.com/) to set the environment up. The followings are steps.

1. Create a file named `Dockerfile` and edit it using the following content.
    ```docker
    FROM ubuntu:22.04

    RUN apt-get update && apt-get --yes upgrade

    RUN apt-get install --yes build-essential python-is-python3 git python3-pip libreadline-dev bison flex

    RUN pip install meson ninja

    # Install wrapper_c.
    RUN git clone https://github.com/adonis0147/wrapper_c && \
        cd /wrapper_c && \
        git submodule update --init --progress && \
        meson build --buildtype=release -Dprefix=/opt/wrapper_c && \
        ninja install -C build

    CMD ["/bin/bash"]
    ```

2. Build the docker image.
    ```shell
    docker build --platform=linux/x86_64 -t wrapper_c .
    ```

3. Run the image.
    ```shell
    mkdir -p data
    docker run -it --mount type=bind,source="$(pwd)/data",target=/data wrapper_c bash
    ```

4. Run the following commands to finish the preparation.
    ```shell
    # Download the source code of PostgreSQL.
    cd /data
    git clone https://github.com/postgres/postgres
    cd postgres
    mkdir build
    cd build
    ```

5. The usage of [wrapper_c](https://github.com/adonis0147/wrapper_c) is simple.
Just run the following commands and `compile_commands.json` will be generated.

    ```shell
    CC=/opt/wrapper_c/bin/wrapper_gcc \
    CXX=/opt/wrapper_c/bin/wrapper_g++ \
    ../configure --prefix="$(pwd)/output" --enable-debug --enable-cassert

    /opt/wrapper_c/bin/wrapper_make -j "$(nproc)"
    ```
