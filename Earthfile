VERSION 0.8

# Base build environment - shared across all versions for efficient caching
base-build-env:
    FROM alpine:3.18
    ENV TZ=Europe/London
    
    # Install all the Linux build dependencies
    # after clang-14 we get errors
    RUN apk add --no-cache alpine-sdk git patch wget clang14 make build-base musl-dev
    RUN apk add --no-cache clang14-dev gcc lld
    RUN apk add --no-cache llvm14 curl libarchive-tools xz

    # Create convenience symlinks for clang
    RUN ln -s /usr/bin/clang-14 /usr/local/bin/clang
    RUN ln -s /usr/bin/clang++-14 /usr/local/bin/clang++

    # Build UASM assembler on Linux - this is expensive, so we do it once
    RUN mkdir /usr/local/src && cd /usr/local/src && git clone --branch v2.57r https://github.com/Terraspace/UASM.git
    RUN cd /usr/local/src/UASM && make CC="clang -fcommon -static -std=gnu99 -Wno-error=int-conversion" -f Makefile-Linux-GCC-64.mak
    RUN cp /usr/local/src/UASM/GccUnixR/uasm /usr/local/bin/uasm

    # Save the built environment for reuse
    SAVE IMAGE build-env:latest

# Build 7zz binary for specific version
build-7zz:
    # ARG VERSION - 7zip build version (e.g., 2409)
    ARG VERSION=2107
    FROM +base-build-env
    
    # Download 7-zip source in Tar XZ format
    RUN curl -o /tmp/7z${VERSION}-src.tar.xz "https://www.7-zip.org/a/7z${VERSION}-src.tar.xz"
    RUN mkdir /usr/local/src/7z${VERSION} && cd /usr/local/src/7z${VERSION} && tar -xf /tmp/7z${VERSION}-src.tar.xz
    RUN rm -f /tmp/7z${VERSION}-src.tar.xz

    # MUSL doesn't support pthread_attr_setaffinity_np so we have to disable affinity
    # we also have to amend the warnings so we don't trip over "disabled expansion of recursive macro"
    # we need a small patch to ensure UASM doesn't try to align the stack in any assembler functions - this mimics expected asmc behaviour
    RUN cd /usr/local/src/7z${VERSION} && sed -i -e '1i\OPTION FRAMEPRESERVEFLAGS:ON\nOPTION PROLOGUE:NONE\nOPTION EPILOGUE:NONE' Asm/x86/7zAsm.asm

    # Create the Clang version
    RUN cd /usr/local/src/7z${VERSION}/CPP/7zip/Bundles/Alone2 && make CFLAGS_BASE_LIST="-c -static -D_7ZIP_AFFINITY_DISABLE=1 -DZ7_AFFINITY_DISABLE=1 -D_GNU_SOURCE=1" MY_ASM=uasm MY_ARCH="-static" CFLAGS_WARN_WALL="-Wall -Wextra" -f ../../cmpl_clang_x64.mak
    RUN strip /usr/local/src/7z${VERSION}/CPP/7zip/Bundles/Alone2/b/c_x64/7zz
    
    # Create dist directory structure and copy the binary
    RUN mkdir -p dist/${VERSION}
    RUN cp /usr/local/src/7z${VERSION}/CPP/7zip/Bundles/Alone2/b/c_x64/7zz dist/${VERSION}/7zz

    # Clean up the version-specific source files
    RUN rm -rf /usr/local/src/7z${VERSION}
    
    # Save the 7zz binary to the output directory
    SAVE ARTIFACT dist/${VERSION}/7zz AS LOCAL dist/${VERSION}/7zz