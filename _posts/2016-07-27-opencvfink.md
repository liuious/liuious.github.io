---
layout: post
title: 基于fink的opencv搭建
author: Arthur
date: 2016-07-27 14:15:44 +0800
tags: OpenCV 关于
---

转自[在MacOSX10.6下编译安装高性能OpenCV库](http://tianchunbinghe.blog.163.com/blog/static/7001201151592834161/)

最初由 Intel 开发的 [OpenCV](http://opencv.org/)库已经逐渐成为目前计算机视觉和图像识别领域的事实标准了，教科书里几乎所有的主流图形算法都在 OpenCV 里可以找到高效的实现。为了完成公司安排给我的工作，这个库是我的必经之路，道理很简单：我不可能花费一个相关专业的硕士研究生的在校时间去把所有底层算法从头学习并实现一遍，我的时间顶多只有一两个月，因此重要的是如何用尽可能少的时间把整个工作中我最不擅长的部分用现成的开源软件来完成。

作为一个前 Linux 系统管理员，安装各种软件是我最拿手的事情。OpenCV的安装并不难，但是严格按照官方的[安装向导](https://github.com/opencv/opencv/wiki)来操作并不能得到一个真正高效的库。这个库的性能直接决定了我的工作效率：一次典型的机器学习过程在一台普通的电脑上单CPU执行通常需要花费几个小时到几天的时间，但如果我可以多核一起计算，就可以几倍地节省时间，把一次机器学习的时间减少到从几十分钟到几个小时的尺度上，从而可以轻松地调节机器学习过程中的各种参数并快速尝试多种分类器。目前我使用的MacBookPro带有4核超线程的2.2GHz Intel i7处理器，最近的实践表明它如果满负荷地跑起来至少比我另一台双核2.66GHz Intel Cuo 2 Duo 处理器的 Mac mini 快 6倍。因此，为了实现高性能的计算，我在编译安装 OpenCV 时必须达到以下两个目标：  
  1.将 OpenCV 库编译成 64 位的 (x86_64)；  
  2.OpenCV 支持 Intel Threading Building Block (TBB)以进行多线程计算，这个特性必须打开。  
本文描述的是在 Mac OS X 10.6 系统上为了得到一个支持 TBB 的 64 位 OpenCV 库所进行的所有操作步骤，以供同行参考。 

1.**安装 64 位 Fink，以及 cmake 等各种依赖工具和库**

[Fink](http://finkproject.org/)是 Mac OS X 下各种开源工具的管理系统，底层基于 Debian 的 dpkg 和 apt-get。虽然 Mac OS X 下还有另一种称为 Macports 的类似平台，但我强烈推荐使用 Fink，因为 Fink 可以在 Mac 系统里同时安装两个版本，32 位的装在 /sw 目录下，64 位的装在 /sw64 目录下，从而分别满足 Mac 系统里不同类型应用程序的需求。(比如说我就需要 LispWorks 能够使用 32 位的 Berkeley DB 4.7，但同时我还需要 OpenCV 能够使用几种 64 位的开源图形库)。

适用于 Mac OS X 10.6 系统的 Fink 必须从源代码安装。首先到 Fink 的[源代码发布站点下载](http://www.finkproject.org/download/srcdist.php)Fink 的源代码包，比如说目前的最新版 [0.39.3](http://downloads.sourceforge.net/fink/fink-0.39.3.tar.gz) ~/src 目录里，然后进入 fink 源代码目录，运行

    ./bootstrap /sw64

Fink 随后会询问各种问题，第一个问题询问根权限的获取方法，选择使用默认值 Use sudo：

```
Fink must be installed and run with superuser (root) privileges. Fink can
automatically try to become root when it's run from a user account. Since
you're currently running this script as a normal user, the method you choose
will also be used immediately for this script. Avaliable methods:

(1) Use sudo
(2) Use su
(3) None, fink must be run as root

Choose a method: [1] 
```

第二个问题是最重要的，一定要选择把所有软件包编译 **64bit-only** 的：

```
Your hardware is a 64bit-compatible intel processor, so you have the option of
running Fink in 64bit-only mode.  This is not recommended for most users, since
many more packages are available for the default mode (which is mostly 32bit
but includes some 64bit packages).  Which mode would you like to use?

(1) Default (mostly 32bit)
(2) 64bit-only

Choose a mode: [1] 2
```

剩下的几乎所有问题都跟后续各种软件包的源代码下载地址有关，全部采用默认值从主站点下载即可。Fink 接下来会自动下载所需的基础软件并编译安装到系统的 /sw64 目录下，完成后它会提示用户执行初始化脚本 /sw64/bin/init.sh 以设置必要的环境变量，尤其是 PATH（该脚本可以写入到 ~/.bash_profile 中以便日后打开终端窗口可以直接使用 64 位的 Fink，但如果还要安装 32 位 Fink 的话需要注意系统里不能同时设置两种 Fink 的环境变量）。然后先执行 **fink selfupdate** 更新一次 Fink 自身，再执行 **fink update-all** 更新所有基础软件包。接下来用 **fink install** 命令安装 OpenCV 所需要的下列第三方程序和库：  

- cmake，一个用来生成 Makefile 和其他 IDE (包括苹果的 Xcode) 的工程文件的程序:  
- libpng14，PNG 图片支持库  
- libjpeg，JPEG 图片支持库  
- libtiff，TIFF 图片支持库  
- libjasper.1，JPEG-2000 图片支持库  
- pkgconfig，一个帮助寻找各种依赖库的辅助程序  

我不需要视频相关的功能，因此以 ffmpeg 为首的各种视频相关的程序库我都没有安装，直接输入一次 fink 命令就可以安装所有上述依赖库了：(各种文件格式的支持库如果不安装的话也可以，OpenCV 源代码自带了，但我觉得还是用 Fink 系统的比较好，至少以后升级方便)

    $ fink install cmake libpng14 libjpeg libtiff libjasper.1 pkgconfig 

2.**安装 Intel Threading Building Blocks**

[Intel Threading Building Blocks](http://threadingbuildingblocks.org/) (TBB) 类似于以前的 OpenMP 和苹果系统自带的 GCD，用于帮助用户创建隐式的并行计算程序，底层依赖于操作系统的多线程库。TBB 最初是 Intel 的商业软件，但自从 3.0 以后就开源了，同时 Intel 仍然销售带有技术支持的商业版本。我在其下载站点上[下载](https://www.threadingbuildingblocks.org/download)了最新的 Commercial Aligned Release，目前的版本是 [4.4 update 5](https://www.threadingbuildingblocks.org/sites/default/files/software_releases/mac/tbb44_20160526oss_osx_2.tgz)。下载后我把它移动到了 /opt/intel 目录下并创建了一个软链接以便日后版本升级时可以随意切换到新版本：

binghe@binghe-i7:/opt/intel$ ls -l  
total 8  
lrwxr-xr-x   1 root    wheel   12  6 14 13:55 tbb -> tbb30_196oss  
drwxr-xr-x  10 binghe  lisp   340  5 10 18:28 tbb30_196oss  

然后需要做两件事，首先修改 /opt/intel/tbb/bin/tbbvars.sh 脚本文件的第一个有效行，将正确的安装位置设置到 TBB30_INSTALL_DIR 环境变量上：

    TBB30_INSTALL_DIR="/opt/intel/tbb"

然后在 **~/.bash_profile** 里添加一行以确保 TBB 库可被以后编译出来的 OpenCV 库成功加载：

    source /opt/intel/tbb/bin/tbbvars.sh

操作完成以后重新打开一个终端窗口，显示 DYLD_LIBRARY_PATH 环境变量的值，里面应该包含 TBB 的路径才对：

$ echo $DYLD_LIBRARY_PATH 
/opt/intel/tbb/lib  

3.**下载并编译 OpenCV 源代码**

找一个比较合适的专门放源代码的地方（推荐使用 ~/src 或者 ~/Source），用 git 把最新的 OpenCV 源代码 clone 下来：

    $ git clone [https://github.com/opencv/opencv.git](https://github.com/opencv/opencv.git)

然后进入 opencv 目录，执行下列 cmake 命令，创建用于编译的 Unix Makefile：

    $ cmake -G "Unix Makefiles" -D BUILD_NEW_PYTHON_SUPPORT=OFF -D CMAKE_OSX_ARCHITECTURES="x86_64" -D WITH_TBB=ON -D TBB_INCLUDE_DIR=/opt/intel/tbb/include .

这是整个编译过程的关键所在。我不编译 Python 支持，强制编译成 64 位，并且要求其打开 TBB 支持。cmake 最后会输出一个汇总报告：

-- 
-- General configuration for opencv 2.2.9 ===========  
--   
--     Built as dynamic libs?:     ON  
--     Compiler:                     
--     C++ flags (Release):          -Wall -pthread  -O3 -DNDEBUG    -fomit-frame-pointer -O3 -ffast-math -msse -msse2 -DNDEBUG 
--     C++ flags (Debug):            -Wall -pthread  -g  -O0 -ggdb3 -DDEBUG -D_DEBUG   
--     Linker flags (Release):        
--     Linker flags (Debug):          
--   
--   GUI:   
--     Cocoa:                      YES  
--   
--   Media I/O:   
--     ZLib:                       TRUE  
--     JPEG:                       TRUE  
--     PNG:                        TRUE  
--     TIFF:                       TRUE  
--     JPEG 2000:                  TRUE  
--     OpenEXR:                    NO  
--     OpenNI:                     FALSE  
--   
--   Video I/O:                    QTKit  
--   
--   Interfaces:   
--     Python:                     OFF  
--     Python interpreter:           
--     Python numpy:               NO (Python interface will not cover OpenCV 2.x API)  
--     Use IPP:                    NO  
--     Use TBB:                    YES  
--     Use Cuda:                   NO  
--     Use Eigen:                  NO  
--   
--   Documentation:   
--     Build Documentation:        NO   
--   
--     Install path:               /usr/local  
--   
--     cvconfig.h is in:           /users/binghe/src/o/opencv  
-- -----------------------------------------------------------------  
--   
-- Configuring done  
-- Generating done  
-- Build files have been written to: /users/binghe/src/o/opencv  

其中最关键的部分是，Use TBB 一项必须正确地显示为 YES，否则说明 TBB 库没有安装好，那么最后编译出来的 OpenCV 也就是废材一个了。如果已经成功走到这一步了，接下来就可以用 make 命令编译了。并行编译是允许了，我的系统有 2 个逻辑 CPU，所以可以全部用来加速编译过程，编译成功后再用 sudo make install 把它安装到 /usr/local 下：

    $ make -j 2  
    $ sudo make install

编译和安装成功的标志是查看 /usr/local/lib 下的一个 OpenCV 库文件的依赖库时应该可以看到 libtbb.dylib，然后用 file 命令可以确认这是一个纯 64 位的库：

$ otool -L /usr/local/lib/libopencv_core.dylib 
   /usr/local/lib/libopencv_core.dylib:
 lib/libopencv_core.2.2.dylib (compatibility version 2.2.0, current version 2.2.9)  
    libtbb.dylib (compatibility version 0.0.0, current version 0.0.0)  
    /usr/lib/libz.1.dylib (compatibility version 1.0.0, current version 1.2.3)  
    /usr/lib/libstdc++.6.dylib (compatibility version 7.0.0, current version 7.9.0)  
    /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 125.2.0)  
$ file /usr/local/lib/libopencv_core.dylib  
   /usr/local/lib/libopencv_core.dylib: Mach-O 64-bit dynamically linked shared library x86_64

4.**测试**

这样就顺利编译完成了。最后我们用一个简单的程序测试一下其可用性：

``` c
#include "opencv/highgui.h"

int main (int argc, char** argv) {
  IplImage* img = cvLoadImage(argv[1], -1);
  cvNamedWindow("Example1", CV_WINDOW_AUTOSIZE);
  cvShowImage("Example1", img);
  cvWaitKey(0);
  cvReleaseImage(&img);
  cvDestroyWindow("Example1");
  return 0;
}
```

上述代码保存成 hello.c 以后用下列命令行应当能够成功编译成一个可执行文件 hello：

    $ gcc -lopencv_highgui -lopencv_core -o hello hello.c

然后以一个图片文件名为参数调用该程序应该可以在一个图形窗口里显示该图片的内容。如果程序正常运行，那么就说明 OpenCV 的安装成功了。