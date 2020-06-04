---
layout: post
title: Building Wine 3.0 for Android
category: general
description: A guide on how to get Wine working on your Android device
---

![Wine on Android ARM](https://i.imgur.com/e5pd4Ab.png?1)

Yesterday marked the official release of [`Wine 3.0`](https://www.winehq.org/announce/3.0), bringing a multitude
of bugfixes, support improvements, and long-awaited features. Among these is the initial release of the Android
support drivers for the Wine platform. Naturally, when I heard the news I was excited to try it out! However,
I was disappointed to find a complete and utter lack of documentation on building it for Android. This post is a
collection of test notes and build steps to jump right into getting Wine running on your ARM/ARM64/x86 Android,
as well as Wine on Android development for potential contributors.

***Disclaimer: Wine on Android is still very much under development and may not work for all devices, architectures,
and graphics drivers. Additionally, Wine Qemu integration for Android is not yet fully complete, meaning you will
currently ONLY be able to run Win32 applications on x86 devices and WinRT applications on ARM devices.
See <https://forum.xda-developers.com/showthread.php?t=2092348> for a list of programs ported to WinRT.***

<!--description-->

## Prerequisites

In order to build Wine for Android, you're going to need a few prerequisites:

 - For starters, you're going to need `gradle`, which can easily be acquired on Debian-based distros by running
    ```bash
    sudo apt install gradle
    ```

 - Additionally, you're going to need to build Wine for your host system first, so go ahead and grab all the necessary
    dependencies. For Debian-based operating systems, that can be done by running
    ```bash
    sudo apt build-dep wine
    ```

 - Next, you're going to need the `Android SDK`. Installation instructions depend on your platform and more information
    on how to install it can be attained at the Android Studio website <https://developer.android.com/studio/index.html>.
    Once you've finished installing Android Studio, open SDK manager and install the Android 21 (5.0) SDK tools. When this
    is done, export the environment variable `$ANDROID_HOME` to point to the SDK manager directory.

 - Finally, you'll want to get the `Android NDK` to compile Wine for the Android platform. Information on how to install
    this component is over at <https://developer.android.com/ndk/index.html>. When you've installed the NDK somewhere,
    export the environment variable `$NDK_ROOT` to point to the top directory of the NDK (i.e. the one with `ndk-build`
    in it).

## Building

Okay, so you made it this far. Congratulations!

This guide assumes that all building will take place in `~/wine-android`, so go ahead and create that:
```bash
mkdir ~/wine-android && cd ~/wine-android
```
Next, grab a copy of Wine from <http://winehq.org> and extract it:
```bash
wget https://dl.winehq.org/wine/source/3.0/wine-3.0.tar.xz
tar xf wine-3.0.tar.xz
```
Now you should have a directory called `wine-3.0` in `~/wine-android`, go ahead and copy it (we're going to need
to build a native version of Wine first).
```bash
cp -r wine-3.0 wine-3.0-native
```
`cd` into it:
```bash
cd wine-3.0-native
```
And build Wine for your host system. A guide on how to do this can be found [`here`](https://wiki.winehq.org/Building_Wine)
but basically the commands to build are as follows:

**For 32-bit host:**
```bash
./configure
```
**For 64-bit host:**
```bash
./configure --enable-win64 # No use in building useless i686 support
```
Next run:
```bash
make
```
and you're done!
Building for the host first is necessary because we'll need everything in the `tools/` directory to cross-compile.
`cd` back to the parent directory:
```bash
cd ..
```
Now it's time to get ready to cross-compile! Depending on what architecture you want, you'll have to use
a different toolchain. Some common ones have been listed:

**For 32-bit ARM targets:**
```bash
export TOOLCHAIN_VERSION="arm-linux-androideabi-4.9"
export TOOLCHAIN_TRIPLE="arm-linux-androideabi"
```
**For 64-bit ARM targets:**
```bash
export TOOLCHAIN_VERSION="aarch64-linux-android-4.9"
export TOOLCHAIN_TRIPLE="aarch64-linux-android"
```
**For 32-bit x86 targets:**
```bash
export TOOLCHAIN_VERSION="x86-4.9"
export TOOLCHAIN_TRIPLE="i686-linux-android"
```
When you've exported the proper toolchain version, build a standalone toolchain using the following command:
```bash
$NDK_ROOT/build/tools/make-standalone-toolchain.sh --platform=android-21 --install-dir=android-toolchain --toolchain=$TOOLCHAIN_VERSION
```
This should hopefully complete without error and create the directory `android-toolchain/`. Export it to your `$PATH`:
```bash
export PATH=`pwd`/android-toolchain/bin:$PATH
```
Next, we're going to need to download and build `Freetype2` for Android. Freetype is necessary to render fonts in Wine.
Download and extract Freetype using the following commands:
```bash
wget https://download.savannah.gnu.org/releases/freetype/freetype-2.6.tar.gz
```
```bash
tar xf freetype-2.6.tar.gz
```
Now `cd` into the newly-created directory to begin cross-compiling Freetype.
```bash
cd freetype-2.6
```
Now that you're in the source tree of Freetype, run the `configure` script like so:
```bash
./configure --host=$TOOLCHAIN_TRIPLE --prefix=`pwd`/output --without-zlib --with-png=no --with-harfbuzz=no
```
After that's complete, build and install Freetype to `output/`:
```bash
make -j`nproc` && make install
```
Congratulations! You've successfully compiled Freetype for Android, now onto Wine!
```bash
cd ../wine-3.0
```
Wine configuration is a bit trickier, but if we set a few environment variables it should work just fine.
First, let's work around an optimization bug for certain versions of `gcc` when compiling Wine:
```bash
export CFLAGS="-O2"
```
Now we're going to need to tell the configuration scripts where we placed the Freetype library and headers.
This can be done by exporting their respective directory paths:
```bash
export FREETYPE_CFLAGS="-I`pwd`/../freetype-2.6/output/include/freetype2"
export FREETYPE_LIBS="-L`pwd`/../freetype-2.6/output/lib"
```
Now run `configure` in the Wine directory like so:
```bash
./configure --host=$TOOLCHAIN_TRIPLE host_alias=$TOOLCHAIN_TRIPLE \
    --with-wine-tools=../wine-3.0-native --prefix=`pwd`/dlls/wineandroid.drv/assets
```
The configuration should work just fine, however if you get any errors related to Freetype, you likely
exported your `Freetype2` build locations incorrectly. Assuming everything worked correctly,
you can now build Wine for Android like so:
```bash
make -j`nproc` && make install
```
If there were no compilation errors, you can now proceed to copy the Freetype shared library you built
over to the Wine Android driver assets.
```bash
# YOUR_ARCH will be the architecture code of the platform you're building for (e.g. armv7-a, x86, etc.)
cp ../freetype-2.6/output/lib/libfreetype.so dlls/wineandroid.drv/assets/YOUR_ARCH/lib/
```
Next, `cd` into the Wine Android driver directory to begin packaging:
```bash
cd dlls/wineandroid.drv/
```
There will already be an APK in this directory, but it's missing the necessary assets, so clean up.
```bash
make clean
```
Now run `make` again to build the proper APK:
```bash
make
```
If everything works correctly, you should have a file `wine-debug.apk` that's `~50MB` in size!
Copy it over to your device's storage, or install it using `adb`.

## Issues
One of the biggest issues I've noticed so far is that Wine refuses to load the desktop shell
on certain devices. This seems to be related to certain problems with some graphics hardware/drivers.
Unfortunately there's not a whole lot I can do about it at this point. I'm actively hacking the Wine
codebase in order to try and fix this up. One notorious example of this is trying to load it on
an Atom AVD using native graphics on desktop NVIDIA hardware. Disable native graphics to make it work.
For phones and tablets that have trouble, you may have some luck toggling `Force GPU Rendering`
in developer options.

Another issue surrounds the Task Bar and Window sizing. On some devices the Start menu is unusable
and the only way to access programs is through the Wine console. I'm working in getting this issue
fixed up as it seems to be okay in the automated builds. As for the window sizing issue, just launch
`winecfg.exe` and set the display DPI to your liking.

While Wine supports input through the touchscreen and virtual keyboard, it doesn't open up automatically
when you select an input box. Once again, I'm working on getting this fixed. In the meantime, you can
pair a bluetooth keyboard up to your device, or install [`Hacker's Keyboard`](https://play.google.com/store/apps/details?id=org.pocketworkstation.pckeyboard) and enable the persistent
notification to open the keyboard when it's pressed.

Finally, for those who compiled Wine for an ARM-based Android, in order to run WinRT applications, you
will need to make sure you have the [`tpidrurw`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a4780adeefd042482f624f5e0d577bf9cdcbb760) patch applied to your device's kernel. There's a tool that you
can build to check this over at <https://github.com/AndreRH/tpidrurw-test>.

## For the Lazy
There's already some automated builds for Wine on Android on the official download site.
Check them out here if your device is supported: <https://dl.winehq.org/wine-builds/android>.
