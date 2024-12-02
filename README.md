# Build FreeFileSync on 64bit Raspberry Pi OS
FreeFileSync is a great open source file synchronization tool.
Building from source on linux is straightfoward *if* all the necessary dependencies are installed.
These instruction capture the necessary steps for installing the various dependencies and compiling FreeFileSync on 64-bit Raspberry Pi OS.
This specific set of instructions was cloned from [subere](https://github.com/Subere/build-FreeFileSync-on-raspberry-pi), itself a fork that originated with jeffli

## Sources of information
These instructions try to reference discussions on the FreeFileSync forums where applicable and the [debian patches](https://sources.debian.org/patches/freefilesync/) associated with the unofficial FreeFileSync Debian build. 

These instructions are applicable to the following versions:

Item  | Release/Version
------------ | -------------
64 Bit Raspberry Pi OS (Raspbian) | Linux raspberrypi 6.6.62+rpt-rpi-v8 + #1 SMP PREEMPT Debian 1:6.6.62-1+rtp1 (2024-11-25) aarch64 GNU/Linux (from ```uname -a```)
FreeFileSync | ```v13.8```

## 1. Download and extract the FreeFilesSync source code

As of this writing, the latest version of FreeFileSync is 13.8 and it can be downloaded from: 

https://freefilesync.org/download/FreeFileSync_13.8_Source.zip

Move the .zip file to the desired directory and uncompress
```unzip FreeFileSync_13.8_Source.zip```

## 2. Install available dependencies via apt-get
These instructions reflect building FreeFileSync using libgtk-3 but using libgtk-3 may lead to a non-optimal user experience- see:
https://freefilesync.org/forum/viewtopic.php?t=7660#p26057

The following dependencies need to be installed to compile:
- libgtk-3-dev (will pull in many, many other dependencies)
- libxtst-dev
- libssh2-1-dev

```
sudo apt-get update
sudo apt-get install libgtk-3-dev 
sudo apt-get install libxtst-dev
sudo apt-get install libssh2-1-dev
```

## 3. Compile dependencies not available via apt-get

The following dependencies could not be installed via `apt-get` and need to be compiled from their source code.


### 3.3 libcurl
Starting with FreeFileSync v13.8, the minimum curl version (that provides the needed libcurl library) needed is 8.8

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
Perform additional step so that the newly created libcurl libraries get put into the appropriate place for 64-bit RaspberryPi OS
```
sudo cp /usr/local/lib/libcurl.so.4.8.0 /usr/lib/aarch64-linux-gnu/
sudo ldconfig
```

### 3.4 wxWidgets
Starting with FreeFileSync v13.2, the minimum version for WxWidgets is 3.2.3.

Build instructions are:
```
wget https://github.com/wxWidgets/wxWidgets/releases/download/v3.2.3/wxWidgets-3.2.3.tar.bz2
tar xvf wxWidgets-3.2.3.tar.bz2
cd wxWidgets-3.2.3/
mkdir gtk-build
cd gtk-build/
../configure --disable-shared --enable-unicode --enable-no_exceptions
make
sudo make install
```
The need to disable WxWidget exception handling (using the '--enable-no_exceptions' options) was mentioned with the introduction of FFSv13.2 in the forums at:
https://freefilesync.org/forum/viewtopic.php?t=10794
If exceptions are enable on wxWidgets, FreeFileSync code could be modified to remove the check or to throw a warning instead of an error.

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

### 4.3 Add workaround for libglibc weirndess

Right after this section:
```
     //the remaining icon types won't block!
     assert(GDK_IS_PIXBUF(gicon) || G_IS_THEMED_ICON(gicon) || G_IS_EMBLEMED_ICON(gicon));
 ```

Add the following:
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

The good news is that Google allows users to set up their own Google project with its own Client ID and Client Secret (free for personal testing/use). Once you get your own application registered, add the provided information in between the " " on lines 92 and 93:
```
std::string getGdriveClientId    () { return ""; } // => replace with live credentials
std::string getGdriveClientSecret() { return ""; } //
```

## 5. Compile in FreefileSync/Source directory

Run ```make``` in the folder FreeFileSync/Source. 

Assuming the command completed without fatal errors, the binary should be waiting for you in FreeFileSync/Build/Bin. 

## 6. Run FreeFileSync
Go to the FreeFileSync/Build/Bin directory and enter:
```
./FreeFileSync_xxx
```

# Troubleshooting & Known Issues

##  Image used in 'About' diaglog is missing
When opening the 'About' dialog, a reference image file is missing. The lack of image generates an error but doesn't seem to have any other impact.
The issue seems to have been reported at:
https://freefilesync.org/forum/viewtopic.php?t=9444

## Other issues could exist
Other issues could certainly exist as overall usage of FreeFileSync on Raspberry Pi is presumably small. It seems likely there could be issues with the more demanding or complex use-cases.
