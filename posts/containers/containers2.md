---
title: "Errors, builds, and containers (2)"
description: "Multi-stage dockerfile"
date: "2024-03-11"
categories:
    - others
image: icons.svg
toc: true
toc-depth: 4
---

## Introduction

[Last time](containers.md) we created a _Containerfile_ to build [Seven Kingdoms: Ancient Adversaries](https://en.wikipedia.org/wiki/Seven_Kingdoms_(video_game)) from source. We then utilized [Appimage](https://en.wikipedia.org/wiki/AppImage) to distribute the executable with all its dependencies for various Linux systems.

They way we created the containerfile is that we iteratively made it work and fixed any issues we have encountered. We did it by splitting steps into dependency install part, building binary, and then finally building the AppImage. To make the life a bit easier, these steps are cached, so they don't need to be ran again if they are not changed. However, if we were missing dependency for AppImage, we had to rebuild everything from scratch, which wasn't very efficient.

I think we can do better. Since we first build the game from sources, and only then we created the AppImage, I think we could reasonably break this into two step process, essentially have a containerfile for build from the source, and another one for AppImage only. But connected. And only in a single file. That is what Multi-staged builds are.

## Multi-staged builds

As described above, multi-staged builds are essentially several different separated steps that are chained together. They make things easier as reason about and reduce potential cross-contamination between different unrelated parts of containerfile as you need to define everything explicitely, including copying any files or _artifacts_ from one stage to another.

So let's prepare a multi-staged `Containerfile2`:

```
FROM ubuntu:22.04 as binary

# you can add multiple labels in a field=value format
LABEL maintainer="Jiří Moravec"

# tool required to make binary
# alphabetically sorted (kind of) according to best practices
RUN apt-get update && apt-get install -y \
    automake \
    autopoint \
    autoconf \
    autoconf-archive \
    g++ \
    gettext \
    git \
    make

# 7kaa dependencies
RUN apt-get install -y \
    libcurl4-openssl-dev \
    libenet-dev \
    libopenal-dev \
    libsdl2-dev

RUN git clone https://github.com/the3dfxdude/7kaa /home/7kaa


WORKDIR /home/7kaa

RUN ./autogen.sh && \
    ./configure && \
    make install DESTDIR=/home/7kaa/inst && \
    cp src/7k.ico /home/7kaa/inst
```
This is our first part. We call it `binary` (coz we are building a binary) and to make it more general, we install it into `inst` directory instead of the `AppImage`.

This is everything we need to build `7kaa` binary straight from source.
If there are other unstated dependencies, they are installed with what we are installing or they are part of the Ubuntu image.

Note that if we were to run `make` instead of `make install` and copied only the binary, we wouldn't get locale files, and the run would run only in English. But we already have German, Polish, Spanish and other translations, so why not use them as well!

Now, we will create the second stage of our `Containerfile2`, where we will utilize the artifacts from the `binary` part to build an AppImage.

```
FROM ubuntu:22.04 as appimage

# copying artifacts
COPY --from=binary /home/7kaa/inst/* /home/7kaa/AppDir

# 7kaa dependencies, no dev variants required!
RUN apt-get update && apt-get install -y \
    libcurl4 \
    libenet7 \
    libopenal1 \
    libsdl2-2.0-0


RUN apt-get install -y \
    imagemagick \
    file \
    lbzip2 \
    libfuse2 \
    wget


WORKDIR /home/7kaa/AppDir

RUN mv usr/local/* usr/ && \
    rm -r usr/local

RUN convert 7k.ico 7k.png && rm 7k.ico

RUN echo '#!/bin/env sh' > 7krun && \
    echo '' 7krun && \
    echo 'SKDATA=share/7kaa/ 7kaa' >> 7krun && \
    chmod +x bin/7krun && \
    mv 7krun usr/bin

RUN echo '[Desktop Entry]' > 7kaa.desktop && \
    echo 'Type=Application' >> 7kaa.desktop && \
    echo 'Name=7kaa' >> 7kaa.desktop && \
    echo 'Comment=Seven Kingdoms: Ancient Adversaries' >> 7kaa.desktop && \
    echo 'Path=/usr/bin' >> 7kaa.desktop && \
    echo 'Exec=7krun' >> 7kaa.desktop && \
    echo 'Icon=7k' >> 7kaa.desktop && \
    echo 'Categories=Game;' >> 7kaa.desktop

RUN wget -nv -c -4 https://www.7kfans.com/downloads/7kaa-music-2.15.tar.bz2 && \
    tar -xf 7kaa-music-2.15.tar.bz2 && \
    mv 7kaa-music/MUSIC/ usr/share/7kaa/ && \
    rm -r 7kaa-music

WORKDIR /home/7kaa/

RUN wget -nv -c https://github.com/AppImage/AppImageKit/releases/download/13/AppRun-x86_64 -O AppDir/AppRun && \
    chmod +x AppDir/AppRun && \
    wget -nv -c https://github.com/linuxdeploy/linuxdeploy/releases/download/1-alpha-20220822-1/linuxdeploy-x86_64.AppImage && \
    chmod +x linuxdeploy-x86_64.AppImage && \
    VERSION=dev ./linuxdeploy-x86_64.AppImage \
        --appimage-extract-and-run \
        --appdir AppDir \
        -d AppDir/7kaa.desktop \
        -i AppDir/7k.png \
        --output appimage
```

Some of these steps, like the one creating `7kaa.desktop` are relatively simple, but require a lot of weird bash code. I can see files like that be included in either the game repository, or a repository with a dockerfile and copied over. But also, there is an advantage having everything self-contained.

A similar issue is with the icon. For the purpose of this build, I did not modified the original source (and now, 6 months after I started writing this document, the development have moved from GitHub to SourceForge), but a png icon should surely be in the source code. That would remove quite a hefty dependency.

## Conclusion

Multi-staged builds are powerful tool. If setup correctly, they can save a lot of time rebuilding container after changes. They are also great tool during development since it takes quite a lot of effort to catch all missing dependencies and make everything working as it should.
And during this time, you will be rebuilding a lot. You could think about it as different targets in a makefile.

However, they are not without issues. You might have noticed that the final containerfile is quite a bit more complicated. In addition to this, each layer of multi-staged build adds to the size of the image.
