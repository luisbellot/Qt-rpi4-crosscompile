# How to crosscompile Qt libraries for Raspberry pi 4 in Windows with no pain

This is not a brief tutorial on how to croscompile Qt libraries for Raspberry pi4 on Windows 10 the contents are:
1. How to crosscompile in Windows.
2. How to add gdb support for python scripting to debug with gdbserver.
3. How to setup Qt Creator to crosscompile for Raspberry Pi.
This tutorial es heavily based in others from internet like  https://github.com/UvinduW/Cross-Compiling-Qt-for-Raspberry-Pi-4 (this tutorial is only for linux), https://visualgdb.com/tutorials/raspberry/qt/embedded/ and https://mechatronicsblog.com/cross-compile-and-deploy-qt-5-12-for-raspberry-pi/ (only for linux but includes great tutorial on how to setup Qt Creator).
I have compiled and tested Qt version 5.15.2, for the latest version of rapbian Os (May 2021) probably it will work with other versions as well.

## 1. How Crosscompile Qt in Windows.

we will need to install following tools in __windows machine__:
1. Python, I have installed version 2.7 from the official site but latest version can work also, you need to add the execuatble to the path.
2. Strawberry perl, install it in C:\ to avoid have long name paths (always is best have short name paths whitout spaces).
3. SmarTTY from sysprogs: http://sysprogs.com/SmarTTY/download/. You need this tool because is the tool to sync the sysroot with the Raspberry Pi. Also add the program route to the system path variable.
4. An image burn software (Balena Etcher or Win32DiskImager work fine).
5. The mingw32 tool chain you can download it from https://gnutoolchains.com/mingw32/. I have choose the 8.1 version. Optionally you can download and install Msys2 and install the toolchain with *pacman* command inside MSys32 environment (we will need it to compile gdb with python support), both of them work.
6. PowerShell version 7.1 or bigger.
7. The windows to gnuabihf cross compiler from: https://gnutoolchains.com/raspberry/ here you can choose the version that maches with the gcc in your raspberry Pi, you can find it executing *gcc --version* from the console of the raspberry Pi. In my case I have downloaded the latest image in Raspberry pi official page and it is gcc version 8.3.0. You don't need to download the image from the gnutoolchanins.com site if you don't want.
8. Get the Qt version of your choice from: https://download.qt.io/official_releases/qt/ in my case: https://download.qt.io/official_releases/qt/5.15/5.15.2/single/qt-everywhere-src-5.15.2.tar.xz (note that you have choosen the tar.xz version you will need to install a tool like 7zip to extract it).

Now you must crate a working dir something like C:/rpi/ and then extract the qt sources inside.

In the __Raspberry Pi__ we need to:
1. Enable the SSH.
2. Enable 3D drivers.
3. Enable the camera (optional).
4. Enable the souece code repository (uncoment the line starting with: *deb-src* in */etc/apt/sources.list* file.
5. then 'sudo apt update' and 'sudo apt upgrade' to install all updates.
6. Install build dependencies for qtbase-opensource-src: 'sudo apt build-dep qtbase-opensource-src'.
7. If you want to build them install also:
    1. apt build-dep libqt5webengine-data.
    2. apt build-dep libqt5webkit5.
8. Install libts-dev and gdbserver packages.
9. Install any other development packages you need for your projects.

Back to the __Windows machine__ to sync the sysroot:

From powerShell you must run a command like: `smartty.exe /UpdateSysroot:C:\SysGCC\raspberry\arm-linux-gnueabihf\sysroot`. Also you can create an UpdateSysroot.bat file containing this line. This starts a special mode of smartty that copy all files in the folders selected (it automatically substitutes the hard link files for the source file avoiding problems with missing references).

Prepare the build dir for Qt, usually in the same root folder you have unziped the sources, you can make a new dir named build or any name you want.

### Changes in Qt scripts to configure.

Qt configure scripts depends on packageConf tool that is not working for Windows enviorement (or I didn't find the way to make it working) if we try to do a configure command at this point we will get that it is not a gbm device:

    EGLFS EGLDevice ...................... no
    EGLFS GBM ............................ no

This is because the test for GBM relays only in packageConf tool and also it is not able to find the drm library. You can check it in the configure.log file.
To solve the problem with GBM we must modify the file: '\<your Qt source folder\>\\qtbase\\src\\gui\\configure.json' in the line 269 we can find something like:

    "gbm": {
                "label": "GBM",
                "test": {
                    "head": [
                        "#include <stdlib.h>",
                        "#include <stdint.h>",
                        "extern \"C\" {"
                    ],
                    "tail": [
                        "}"
                    ],
                    "main": "gbm_surface *surface = 0;"
                },
                "headers": "gbm.h",
                "sources": [
                    { "type": "pkgConfig", "args": "gbm"}
                ]
            },
We must modify the sources array to let it like:

`      "sources": [
                      { "type": "pkgConfig", "args": "gbm"},
              "-lgbm"
                  ] `
                  
To solve the problem with the drm library we must modify the file qmake.conf inside 'qtbase\\mkspecs\\devices\\linux-rasp-pi4-v3d-g++' folder and add the path to the drm library. The file must look something like:

    include(../common/linux_device_pre.conf)
    QMAKE_LIBS_EGL         += -lEGL
    QMAKE_LIBS_OPENGL_ES2  += -lGLESv2 -lEGL
    QMAKE_INCDIR_DRM 		= $$[QT_SYSROOT]/usr/include/libdrm
    QMAKE_CFLAGS            = -march=armv8-a -mtune=cortex-a72 -mfpu=crypto-neon-fp-armv8
    QMAKE_CXXFLAGS          = $$QMAKE_CFLAGS
    DISTRO_OPTS            += hard-float
    DISTRO_OPTS            += deb-multi-arch
    EGLFS_DEVICE_INTEGRATION = eglfs_kms
    include(../common/linux_arm_device_post.conf)
    load(qt_config) 

Where the line `QMAKE_INCDIR_DRM 		= $$[QT_SYSROOT]/usr/include/libdrm` is the line containing the right path to the drm library.
There is another step that I'm not sure if we need to do but in any case it doesn't hurt. this is open the file \qtbase\mkspecs\linux-arm-gnueabi-g++\qmake.conf and replace all occurences of arm-linux-gnueabi- with arm-linux-gnueabihf-, like we can find in https://visualgdb.com/tutorials/raspberry/qt/embedded/ reference.
At this point we can try the configure command (this is the command that fits my needs, may be you need to change it to fit your needs):
>../qt-everywhere-src-5.15.2/configure  -release -opengl es2 -device linux-rasp-pi4-v3d-g++ -sysroot C:/SysGCC/Raspberry/arm-linux-gnueabihf/sysroot -prefix /usr/local/qt5 -device-option CROSS_COMPILE=C:/SysGCC/Raspberry/bin/arm-linux-gnueabihf- -opensource -confirm-license -make libs -nomake tests -skip qtwayland -skip qtscript -skip qtwebengine -v -no-use-gold-linker 
>
After configure you must get the right options:

     QPA backends:
      DirectFB ............................... no
      EGLFS .................................. yes
      EGLFS details:
        EGLFS OpenWFD ........................ no
        EGLFS i.Mx6 .......................... no
        EGLFS i.Mx6 Wayland .................. no
        EGLFS RCAR ........................... no
        EGLFS EGLDevice ...................... yes
        EGLFS GBM ............................ yes
        EGLFS VSP2 ........................... no
        EGLFS Mali ........................... no
        EGLFS Raspberry Pi ................... no
        EGLFS X11 ............................ yes
      LinuxFB ................................ yes
      VNC .................................... yes 

Once configured we must do `mingw32-make -j4` and after some time if everything goes right `mingw32-make install` 

Remember we are using PowerShell to type all these commands.

### Upload the libraries to the Raspberry Pi.

To do it in the raspberry pi we must create a folder: `usr/local/qt5` it must match with our instal-prefix in the configure command:
> sudo mkdir /usr/local/qt5

and give it write and read grants for the pi user:
> sudo chown pi /usr/local/qt5

Then we can use smarTTY program to upload the contents of our `sysroot\usr\local\qt5` to the remote `/usr/local/qt5` folder. You can refer to https://visualgdb.com/tutorials/raspberry/qt/embedded/ for more details in how to do it.

## 2.Building gdb for Raspberry Pi with python support.

The gdb version in the toolchain comes without support for python then we need to build it from the QtCreator sources. First make a folder to work and download the latest version of sources for QtCreator at the moment of writing they were: https://download.qt.io/official_releases/qtcreator/4.15/4.15.1/qt-creator-opensource-src-4.15.1.zip that match with my QtCreator version. Unzip the contents go to the `\dist\gdb` folder, edit *Makefile.mingw* file and add suport for arm-linux-gnueabihf:
>targets=arm-none-eabi,arm-none-linux-gnueabi,i686-pc-mingw32,arm-linux-gnueabihf

The versions of expat and iconv are out of date and the download step will fail you must change the version numbers for:
>expatversion=2.4.1
>iconvversion=1.16

The line in make file to download expat source is wrong also you need to remove the `/download` at the end an let it like:
>wget -q http://sourceforge.net/projects/expat/files/expat/${expatversion}/expat-${expatversion}.tar.bz2 && \

After these modifications we need to download and intall if we have not done yet, the MSYS2: https://www.msys2.org/ here you have some tutorials on how to setup it and how to install the development requirements. To compile gdb we must open MSYS32 (not 64, it will fail because the python libraries in the Qt Creator folder are for 32 bits), intall the mingw packages plus unzip command and inside the MSYS32 environment run the command `make -f Makefile.mingw`.
At this point in the lattest sources you will get an error in file location.c:
```
/dist/gdb/staging/gdb-7.12/gdb/location.c:527:16: error: ISO C++ forbids comparison between pointer and integer [-fpermissive]
  527 |       || *argp == '\0'
     |          ~~~~~~^~~~~~~
make[3]: *** [Makefile:1134: location.o] Error 1
```
Just open the file and go to line 527 and correct it:
The original line is like:
      
      if (argp == NULL
            || *argp == '\0'
            || *argp[0] != '-'
            || !isalpha ((*argp)[1])
            || ((*argp)[0] == '-' && (*argp)[1] == 'p'))
          return NULL;

You can let it like:

    if (argp == NULL
          || *argp[0] == '\0'
          || *argp[0] != '-'
          || !isalpha ((*argp)[1])
          || ((*argp)[0] == '-' && (*argp)[1] == 'p'))
        return NULL;
        
Probably we will get another problem with redefinition in file `readline\terminal.c` the problem is in lines:
```
#if !defined (__linux__) && !defined (NCURSES_VERSION)
#  if defined (__EMX__) || defined (NEED_EXTERN_PC)
extern 
#  endif /* __EMX__ || NEED_EXTERN_PC */
char PC, *BC, *UP;
#endif /* !__linux__ && !NCURSES_VERSION */
```
We can add a `#define NEED_EXTERN_PC 1` to quickly solve it and try to compile again.

Now it must compile and packing without problems.

## 3. Qt Creator configuration.

You can find a good description with screenshoots in this page https://mechatronicsblog.com/cross-compile-and-deploy-qt-5-12-for-raspberry-pi/. Here I will put only a brief of the steps we must follow.
1. In Qt Creator menu *tools* select *options...* in the new window select devices and add a new linux generic device.
  1. Put a descriptive name like RaspBerry Pi4 or whatever you want.
  2. Put the ip address and the user name. 
  3. Select *specific Key* radio button to avoid type the password every time.
  4. Generate the key pair if you need it 
  5. Click on deploy public key.
  6. Test the device.
2. Now select *Kits*.
  1. Go to *Qt Versions* tab and click on *add* button. 
  2. Select the qmake.exe file you will find in your `sysroot\usr\local\qt5\bin\` folder.
  3. Give it a name like *Qt 5.15.2 Raspi* or something like this.
3. Go to tab *Compilers* 
  1. Cilck on *add* button and add a new gcc compiler.
  2. Select the gcc compiler for Raspberry Pi you have used for crosscompilations (`C:/SysGCC/Raspberry/bin/arm-linux-gnueabihf-gcc`).
4. Go to tab *Debugers* and add the gdb you have compiled in the previous section.
5. Go to tab *Kits* and add a new one maching the raspberry device, the new qt version, new compilers and new debuger. 

## Final words.

This is a tutorial made to remember myself the steps and possible errors in the crosscompilation procedure. I'm not and expert in Qt nor embeded devices programing and I have missed some things like how to modify `configure.json` files when I was finding a way to crosscompile for Raspberry Pi with windows. 

Why using windows not linux?. I'm working in a company developing software for CAM, all the programs I develop are Windows based, and for me is more confortable have only one development machine with all the tools I need.
