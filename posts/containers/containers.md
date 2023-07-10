---
title: "Errors, builds, and containers"
description: "Solving unhelpful errors in build systems with containerization"
date: "2023-07-09"
categories:
    - others
image: icons.svg
toc: true
toc-depth: 4
---

## Introduction

### Unhelpful errors

You have probably experienced it more than once. You try to run some code and instead of a result, you get a strange error saying that something is broken. You are scratching your head because the error doesn't say clearly what is broken but points to some esoteric part of the code that should be right, or points very deep into subsystem or dependency, which points to another, only for you to discover in the end that the issue is in a completely different part.

Previously, this happened to me with `devtools` or `testthat`, when both packages added another dependency (or their dependency added a dependency) `cli`, which incorrectly handled the version string of one of the most common terminals on one of the most common Linux distro. What was more infuriating is finding that it was supposed to be fixed 6 months ago. Why the main branch wasn't patched, and the bugged version was released on CRAN instead is a mystery to me.

Anyway, enough of this rant, this blogpost is not about this, nor it is about R, but about another completely different issue.

### Complex build systems

In the beginning, well I don't know since I wasn't there, but much later than that we had `GNU make` to build C code. But the build became more and more complex, so more complex tools were developed. And these tools became, and still are, more and more complex. From my tiny experience, it was easier to write a package in Java, than to write and setup Ant or Gradle from scratch. But this might be due to my inexperience, I write code frequently although in compiled languages but I don't write build scripts often enough.

### Dependencies

You can encounter a similar issue if you want to build existing code as well.
For instance, [7kaa](https://github.com/the3dfxdude/7kaa/), a commercial game that was open-sourced by Enlight, uses `autotools`. As you might get from the name, autotools is not a single tool, but a collection of tools. And if you don't have any of those installed, you will get strange errors, especially if you didn't notice a single warning at the start of the very long log file.

In the end, I managed to solve this issue by installing dependencies and potentially polluting my system. But what if I wanted to do this next time again? Would I remember what did I do wrong and how I solved the issue? Can I do something better to isolate the dependency issue in a testable and replicable way so that a completely new clean system can be guaranteed to succeed?

## Podman

[podman](https://podman.io/) is a lesser-known cousin of Docker. The advantage of Podman is that it allows the creation of so-called rootless containers. This makes it a little easier to work with.

But what are Podman/Docker containers? Simply said, containers are packages containing an application, including dependencies and even OS. This means that with containers, you can create a replicable installation and deployment of applications on clean systems, which are isolated from your host system by the Docker/Podman engine.

### Installing podman

On my system (Ubuntu 22.04), `podman` is easy to install:

`sudo apt install podman`

However, there is currently a [bug](https://github.com/containers/podman/issues/8896), the config in `/etc/containers/registries.conf` does not come with correct presets. In fact, it does not come with any presets at all, it only contains documentation. This means that it is not connected to any repository, and the commands mentioned in the tutorial, such as:

```
podman search [search term]
```

do not output anything.

To fix this, we need to append:

```
unqualified-search-registries = ["docker.io"]

[[registry]]
location = "docker.io
```
to the `/etc/containers/registries.conf`. This will allow us to reach the `docker.io` repository.

For instance, to find an `ubuntu` image, we can now do:

```
podman search ubuntu --limit 5
```

### Setting up container

First, we download an image:
```
podman pull docker.io/library/ubuntu
```

Now we can run the container. We run it with the detach `-d` option, as otherwise the container would just execute all required code and stopped working. We also run the container with the `-t` option, to enable tty, or terminal. For repeatability, we provide a custom name with the `--name` command. After that, we `attach` to the container.
```
podman run  --name "test" -dt docker.io/library/ubuntu:latest
podman attach "test"

# to clean the container after finishing, run:
podman rm "test"
```

We are now inside the container, a brand new and clean Ubuntu 22.04 install, and we can start installing dependencies and building our app. After a few tries trying to find out the required libraries and dependencies, I reached this:

```
# update apt
apt update

# install build tools
apt install -y git automake autopoint autoconf autoconf-archive g++ make

# install dependencies
apt install -y libsdl2-dev libenet-dev libopenal-dev libcurl4-openssl-dev gettext

# clone repo and build binary
git clone https://github.com/the3dfxdude/7kaa/ home/7kaa
cd home/7kaa
./autogen.sh && ./configure && make
```

All we need now is to copy the binary and data with `podman container cp [source] [dest]`

### Writing a Containerfile

We have commands that we want to run, but having to run them manually is annoying. To automate this, we can write a `Containerfile`! Or `Dockerfile`, as Docker calls it, but the syntax is identical.

Most container files consist of `FROM`, which specifies the image one is working with, `RUN` which runs various commands used to build the container, `COPY` which allows one to copy content to or from the container, and `CMD` which is then used to launch the applications themselves.
```
# Containerfile
FROM ubuntu:22.04

# you can add multiple labels in a field=value format
LABEL maintainer="j-moravec"


RUN apt-get update && apt-get install -y \
    git automake autopoint autoconf autoconf-archive g++ make \
    libsdl2-dev libenet-dev libopenal-dev libcurl4-openssl-dev gettext

RUN git clone https://github.com/the3dfxdude/7kaa /home/7kaa

WORKDIR /home/7kaa
RUN ./autogen.sh && ./configure && make
```

Build this image with `podman -t "7kaa" -f Containerfile`.

## AppImage

We have a replicable way to build a binary. The issue is that it is built against a particular version of Ubuntu and against a particular version of SDL2, Enet, OpenAL and Curl, so the binary won't work on every system. One way to bundle these dependencies is with AppImage.

Note that to run AppImages, you need to have `libfuse2` installed. Also, I will be using `wget` to download some files, so make sure you have it installed as well. Everything will be provided in the final containerfile.

To create AppImage, we need to create `AppDir` directory with a pre-specified format either manually, or with the help of the [linuxdeploy](https://docs.appimage.org/packaging-guide/from-source/linuxdeploy-user-guide.html#) tool. Linuxdeploy is a tool to create AppImages, so it not only creates the required directory structure, but also copies dependencies for provided binary, compiles provided icon and desktop file (which are required), and also creates AppImage itself. To make an AppImage with linuxdeploy, we need to:

* create `AppDir` structure
* create binary, icon and desktop files in a pre-specified format
* create AppImage

Linuxdeploy allows working in an iterative format. That basically means that you mess around until it works. When you do this, note that provided binary, icon, and a desktop file are currently not updated with repeated runs, so the iterative approach does not work completely. Alas, after mocking around for a bit and trying to figure it out since the documentation is not great and in many areas resembles stump, this is what we will do:

* Install 7kaa into `AppDir`
* convert icon 7k.ico to the png format
* create desktop file
* run linuxdeploy to put it all together and create AppImage

### Installing into AppDir
First, we need to build the application in a way that it is installed. But we do not want to actually install it into the image filesystem, but into the `AppDir`:

```
make install  DESTDIR=/home/7kaa/AppDir
```

Note that due to the way the build system for 7kaa is set up, with multiple makefiles for various subsystems, we need to specify AppDir with an absolute path instead of relative.

Another thing that we need to do is move everything from `/usr/local/` to `/usr/`. While both are valid locations and in fact `/usr/local/` is more suitable from an administrative perspective, linuxdeploy does not recognize binary in `/usr/local/bin/`

```
mv AppDir/usr/local/* AppDir/usr/
rm -r AppDir/usr/local
```
### Converting icon to png
The icon that we have is in `ico` format, which comes from Windows and is not supported by the XDG Linux desktop specification, which accepts only png, svg and xpm.

We can use imagemagick to convert it with:

```
convert src/7k.ico 7k.png
```

But it is another dependency we need to include in our containerfile.

### Creating desktop file
Desktop files are files that allow integration of your program/binary with your desktop, and they are required to create an AppImage. An example can be found on [archlinux wiki](https://wiki.archlinux.org/title/Desktop_entries), with full specification on [freedesktop.org](https://specifications.freedesktop.org/desktop-entry-spec/desktop-entry-spec-latest.html#recognized-keys). Note that quite a few elements appear to be required by the linuxdeploy.

A sample desktop file `7kaa.desktop` might look like this:
```
[Desktop Entry]
Type=Application
Name=7kaa
Comment=Seven Kingdoms: Ancient Adversaries
Path=/usr/bin
Exec=7kaa
Icon=7k # should not contain an extension
Categories=Game; # one of several categories in specification
```

### Running linuxdeploy
Now, all we need to do is to run [linuxdeploy](https://docs.appimage.org/packaging-guide/from-source/linuxdeploy-user-guide.html#), which will copy dependencies required by our binary (such as SDL2), and output an AppImage.

```
wget -nv https://github.com/linuxdeploy/linuxdeploy/releases/download/1-alpha-20220822-1/linuxdeploy-x86_64.AppImage -o linuxdeploy.AppImage
chmod +x linuxdeploy.AppImage
VERSION=dev ./linuxdeploy.AppImage --appdir AppDir -d 7kaa.desktop -i 7k.png --output appimage
```
We have also set a version of the resulting appimage to `dev` by specifying the recognized environment variable `VERSION`.

### Pointing 7kaa to its data
Normally, this is all we would have to do. But if you try to run the AppImage, you will find out that the binary can't find the data, this is because the binary is not in the same directory and the `SKDATA` environment variable is empty. We need to run the `7kaa` binary in the same way that we run the `linuxdeploy.AppImage`, which essentially means we need to write our own runner script.

There are two ways how we can do it, either run a complex bash script that does some environment parsing and settings as required for AppImage, like [imagemagick](https://github.com/KurtPfeifle/ImageMagick/blob/master/appimage/AppRun) does, or we use just a simple runner script and use provided [AppRun](https://github.com/AppImage/AppImageKit/releases), which is a simple binary that does what we need and also parses the desktop file.

First, create an `AppDir/usr/bin/7krun`
```
#!/bin/env sh

SKDATA=share/7kaa/ 7kaa
```
and don't forget to make it executable with `chmod +x AppDir/usr/bin/7krun`.

Then download the `AppRun` binary:
```
wget -nv https://github.com/AppImage/AppImageKit/releases/download/13/AppRun-x86_64 -o AppDir/AppRun
```

and modify the desktop file so it executes `7krun` instead of `7kaa`:
```
[Desktop Entry]
Type=Application
Name=7kaa
Comment=Seven Kingdoms: Ancient Adversaries
Path=/usr/bin
Exec=7krun
Icon=7k
Categories=Game;
```
and now just build the AppImage!

```
VERSION=dev ./linuxdeploy.AppImage --appdir AppDir -d 7kaa.desktop -i 7k.png --output appimage
```

and test with:

```
./7kaa-dev-x86_64.AppImage
```

### Adding music
Due to copyright reasons, music is distributed separately. In fact, one of the motivations for creating an AppImage was that on some Linux distributions, music is not distributed at all due to it not being FOSS.

The music is available at the [7kfans website](https://www.7kfans.com/downloads) as a separate download. We will download the archive, unpack it, transfer the music into the `DATA` directory and rebuild the AppImage.

```
# force IPv4, the default IPv6 seems to be broken and just hang on
wget -nv -c -4 https://www.7kfans.com/downloads/7kaa-music-2.15.tar.bz2
tar -xf 7kaa-music-2.15.tar.bz2
mv 7kaa-music/MUSIC/ AppDir/usr/share/7kaa/

VERSION=dev ./linuxdeploy.AppImage --appdir AppDir -d 7kaa.desktop -i 7k.png --output appimage
```

and test with:

```
./7kaa-dev-x86_64.AppImage
```

### Putting it all together:
In the end, we will get this containerfile:

```
FROM ubuntu:22.04

# you can add multiple labels in a field=value format
LABEL maintainer="j-moravec"


RUN apt-get update && apt-get install -y \
    git automake autopoint autoconf autoconf-archive g++ make \
    libsdl2-dev libenet-dev libopenal-dev libcurl4-openssl-dev gettext \
    imagemagick wget libfuse2 file

RUN git clone https://github.com/the3dfxdude/7kaa /home/7kaa

WORKDIR /home/7kaa

RUN ./autogen.sh && \
    ./configure && \
    make install DESTDIR=/home/7kaa/AppDir && \
    mv AppDir/usr/local/* AppDir/usr/ && \
    rm -r AppDir/usr/local

RUN convert src/7k.ico 7k.png && \
    echo '#!/bin/env sh' > AppDir/usr/bin/7krun && \
    echo '' AppDir/usr/bin/7krun && \
    chmod +x AppDir/usr/bin/7krun && \
    echo 'SKDATA=share/7kaa/ 7kaa' >> AppDir/usr/bin/7krun && \
    echo '[Desktop Entry]' > 7kaa.desktop && \
    echo 'Type=Application' >> 7kaa.desktop && \
    echo 'Name=7kaa' >> 7kaa.desktop && \
    echo 'Comment=Seven Kingdoms: Ancient Adversaries' >> 7kaa.desktop && \
    echo 'Path=/usr/bin' >> 7kaa.desktop && \
    echo 'Exec=7krun' >> 7kaa.desktop && \
    echo 'Icon=7k' >> 7kaa.desktop && \
    echo 'Categories=Game;' >> 7kaa.desktop

RUN wget -nv -c -4 https://www.7kfans.com/downloads/7kaa-music-2.15.tar.bz2 && \
    tar -xf 7kaa-music-2.15.tar.bz2 && \
    mv 7kaa-music/MUSIC/ AppDir/usr/share/7kaa/

RUN wget -nv -c https://github.com/AppImage/AppImageKit/releases/download/13/AppRun-x86_64 -O AppDir/AppRun && \
    chmod +x AppDir/AppRun && \
    wget -nv -c https://github.com/linuxdeploy/linuxdeploy/releases/download/1-alpha-20220822-1/linuxdeploy-x86_64.AppImage && \
    chmod +x linuxdeploy-x86_64.AppImage && \
    VERSION=dev ./linuxdeploy-x86_64.AppImage \
        --appimage-extract-and-run \
        --appdir AppDir \
        -d 7kaa.desktop \
        -i 7k.png \
        --output appimage
```

Note that I did some minor tweeks to iron out potential issues.

Now we should be able to run:
```
podman build -t 7kaa -f Containerfile
```
And half an hour later, when the image is finally built:

```
podman create --name 7kaaAppImage 7kaa
podman cp 7kaaAppImage:/home/7kaa/7kaa-dev-x86_64.AppImage .
podman rm 7kaaAppImage
chmod +x 7kaa-dev-x86_64.AppImage
./7kaa-dev-x86_64.AppImage
```

and everything should work.

## Conclusion

We were able to use podman and dockerfile to create a replicable build of an AppImage for the Seven Kingdoms: Ancient Adversaries.

Handling podman and contairnerfiles was rather easy, the biggest issue was making AppImages themselves. But we managed that as well.
