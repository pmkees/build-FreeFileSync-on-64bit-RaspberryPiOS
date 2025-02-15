# Build FreeFileSync on 64bit Raspberry Pi OS
FreeFileSync is a great open source file synchronization tool.
Building from source is straightfoward *if* all the necessary dependencies are installed.
These instruction capture the necessary steps for installing the various dependencies and compiling FreeFileSync on 64-bit Raspberry Pi OS.
This specific set of instructions was cloned from [subere](https://github.com/Subere/build-FreeFileSync-on-raspberry-pi), itself a fork-of-a-fork (of a fork) that originated with [jeffli](https://github.com/jeffli678/build-FreeFileSync)

## Sources of information
These instructions try to reference discussions on the FreeFileSync forums where applicable and the [debian patches](https://sources.debian.org/patches/freefilesync/) associated with the unofficial FreeFileSync Debian build.
If you frequent the FreeFileSync forms, you might see some familiar names mentioned in the debian patches (shoutout to bgstack15!) 

These instructions are applicable to the following versions:

Item  | Release/Version
------------ | -------------
64 Bit Raspberry Pi OS (Raspbian) | Linux raspberrypi 6.6.62+rpt-rpi-v8 + #1 SMP PREEMPT Debian 1:6.6.62-1+rtp1 (2024-11-25) aarch64 GNU/Linux (from ```uname -a```)
FreeFileSync | ```v13.9```

## 1. Download and extract the FreeFilesSync source code

As of this writing, the latest version of FreeFileSync is 13.9 and it can be downloaded from: 

https://freefilesync.org/download/FreeFileSync_13.9_Source.zip

Move the .zip file to the desired directory and uncompress
```unzip FreeFileSync_13.9_Source.zip```

## 2. Install available dependencies via apt
These instructions reflect building FreeFileSync using libgtk-3 but using libgtk-3 may lead to a non-optimal user experience- see:
https://freefilesync.org/forum/viewtopic.php?t=7660#p26057

The following dependencies need to be installed to compile:
- libgtk-3-dev (will pull in many, many other dependencies)
- libssl-dev
- libpsl-dev
```
sudo apt update
sudo apt install libgtk-3-dev 
sudo apt install libssl-dev
sudo apt install libpsl-dev
```

## 3. Compile dependencies not available via apt

The following dependencies could not be installed via `apt` and need to be compiled from their source code.

### 3.1 libssh2
The minimum libssh2 version needed is 1.11

Acquire, build and install with the following steps:
```
wget https://libssh2.org/download/libssh2-1.11.1.tar.gz
tar xvf libssh2-1.11.1.tar.gz
cd libssh2-1.11.1
mkdir build
cd build/
../configure
make
sudo make install
```
Perform additional step to move the newly created library and overwrite the existing version used by 64-bit RaspberryPi OS.
```
sudo cp /usr/local/lib/libssh2.so.1.0.1 /usr/lib/aarch64-linux-gnu/
sudo ldconfig
```

### 3.2 libcurl
The minimum curl version (that provides the needed libcurl library) needed is 8.8

Acquire, build and install with the following steps:
```
wget https://curl.se/download/curl-8.8.0.tar.gz
tar xvf curl-8.8.0.tar.gz
cd curl-8.8.0/
mkdir build
cd build/
../configure --with-openssl --with-libssh2 --enable-versioned-symbols
make
sudo make install
```
Perform additional step to move the newly created library and overwrite the existing version used by 64-bit RaspberryPi OS.
```
sudo cp /usr/local/lib/libcurl.so.4.8.0 /usr/lib/aarch64-linux-gnu/
sudo ldconfig
```

### 3.3 wxWidgets

Build instructions are:
```
wget https://github.com/wxWidgets/wxWidgets/releases/download/v3.2.6/wxWidgets-3.2.6.tar.bz2
tar xvf wxWidgets-3.2.6.tar.bz2
cd wxWidgets-3.2.6/
mkdir gtk-build
cd gtk-build/
../configure --disable-shared --enable-unicode
make
sudo make install
```
The need to disable WxWidget exception handling (using the '--enable-no_exceptions' options) was mentioned with the introduction of FFSv13.2 in the forums at:
https://freefilesync.org/forum/viewtopic.php?t=10794
It appears that the use of "--enable-no_exceptions" generates other compilation errors and so FileSync code could be modified to remove the check or to throw a warning instead of an error.

## 4. Tweak FreeFileSync code

In additional to providing the dependencies, some code tweaks are needed (see Bugs.txt in the top-level directory for a complete reference)

### 4.1 FreeFileSync/Source/afs/sftp.cpp

Add these constant definitions starting at line 21
```
#define MAX_SFTP_OUTGOING_SIZE 30000
#define MAX_SFTP_READ_SIZE 30000
```

### 4.2 Update FreeFileSync/Source/Makefile to use GTK3 instead of GTK2 
While previously mentioned, use of GTK3 can result in poor UI experience (see thread at: https://freefilesync.org/forum/viewtopic.php?t=7660) the various dependencies for GTK2 building became too arduous for me on Raspbery Pi OS and so, picking my poison, I switched to GTK3 by doing the following:

On line 20:
```
change: cxxFlags  += `pkg-config --cflags gtk+-2.0`
to:     cxxFlags  += `pkg-config --cflags gtk+-3.0`
```

On line 22:
```
change: cxxFlags  += -isystem/usr/include/gtk-2.0
to:     cxxFlags  += -isystem/usr/include/gtk-3.0
```
### 4.3 Update FreeFileSync/Source/application.cpp to change wxWidget exception check from #error to only a #warning

On line 247:
```
change: #error why is wxWidgets uncaught exception handling enabled!?
to:     #warning why is wxWidgets uncaught exception handling enabled!?
```

This will allow compilation and execution - but any logfiles collected for troubleshooting may not be useful.

### 4.4 Add workaround for libglibc weirndess in FreeFileSync/Source/base/icon_loader.cpp

Deep within the libglibc library, a macro is rewritting the line inappropriately resulting in a failed compilation.
The libglibc fix will eventually be available but until then, this workaround is needed.

#### References
* FreeFileSync Forum: https://freefilesync.org/forum/viewtopic.php?t=8780
* Debian patch: https://sources.debian.org/patches/freefilesync/12.0-2/ffs_icon_loader.patch/

Replace this line (line 230)
```
    ::g_object_ref(gicon);                   //pass ownership
```

With the following set of lines:
```
#if (GLIB_CHECK_VERSION (2, 67, 0))
    g_object_ref(gicon);                   //pass ownership
#else
    ::g_object_ref(gicon);                 //pass ownership
#endif
```

### 4.4 [Optional] Populate Google client_id and client_key in Freefilesync/Source/afs/gdrive.cpp
Information about Google Drive support on self-compiled instances was mentioned at https://freefilesync.org/forum/viewtopic.php?t=8171

To set up and use a google cloud location for syncing, your compiled version of Free File Sync needs to be registered with Google (you can't reuse the registration credentials of the official FreeFileSync release for a number of reasons perhaps most fundamentally, you could modify the code in any shape/way/form and no longer use it as the original author intended)

The good news is that Google allows users to set up their own Google project with its own Client ID and Client Secret (free for personal testing/use, see https://developers.google.com/identity/oauth2/web/guides/get-google-api-clientid ). Once you get your own application registered, add the provided information in between the " " on lines 92 and 93:
```
std::string getGdriveClientId    () { return ""; } // => replace with live credentials
std::string getGdriveClientSecret() { return ""; } //
```

## 5. Compile in FreefileSync/Source directory

Run ```make``` in the folder FreeFileSync/Source. 

Assuming the command completed without fatal errors, the binary should be waiting for you in FreeFileSync/Build/Bin. 

## 6. Run FreeFileSync
Go to the FreeFileSync/Build/Bin directory and run by entering:
```
./FreeFileSync_aarch64
```

# Troubleshooting & Known Issues

##  Image used in 'About' diaglog is missing generating an error dialog window
When opening the 'About' dialog, a reference image file is missing. The lack of image generates an error but doesn't seem to have any other impact.
The issue seems to have been reported at:
https://freefilesync.org/forum/viewtopic.php?t=9444

## Other issues could exist
Other issues could certainly exist as overall usage of FreeFileSync on Raspberry Pi is presumably small. It seems likely there could be issues with the more demanding or complex use-cases.
