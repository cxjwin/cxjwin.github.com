---
layout: post
title: "Xcode5下编译amr和speex库"
description: ""
categories: iOS
tags: [shell]
---
   
&emsp;&emsp;Xcode5之后多了对arm64的支持，所以之前编译的库也要适时的更新下，而且gcc和g++在arm平台暂时还木有对arm64进行支持（这个是个人推断）所以编译采用clang和clang++，当然Mac平台采用这两个编译器编译c和c++是要比gcc和g++好的，因为前面两个是apple当初为了Mac平台专门开发的编译器。  
&emsp;&emsp;下面进入正题，相较于之前的在终端模式下，逐行敲命令，我们可选择更为灵活和便利的shell脚本。  
1.编译libamr

	#!/bin/sh
	
	#  opencore_amr.sh
	#  
	#
	#  Created by cxjwin on 13-10-17.
	#
	
	# 打开调试回响模式,并配置环境
	set -xe
	
	# /Applications/Xcode.app/Contents/Developer
	DEVELOPER=`xcode-select -print-path`
	# 我们编译到桌面的opencore_amr_ios文件夹
	DEST=${HOME}/Desktop/opencore_amr_ios
	# 我们需要编译的平台,一共是5个,arm64是64位amr处理器
	ARCHS="i386 x86_64 armv7 armv7s arm64"
	# 一共是两种类型的静态库
	LIBS="libopencore-amrnb.a libopencore-amrwb.a"  
	
	# Note that AMR-NB is for narrow band http://en.wikipedia.org/wiki/Adaptive_Multi-Rate_audio_codec
	# for AMR-WB encoding, refer to http://sourceforge.net/projects/opencore-amr/files/vo-amrwbenc/
	# or AMR Codecs as Shared Libraries http://www.penguin.cz/~utx/amr
	
	# 每一个平台对应一个文件夹
	for arch in $ARCHS;
	do
	    mkdir -p $DEST/$arch
	done
	
	./configure
	
	# libamr是C++库所以只要设置CXX变量就可以了，采用clang++编译
	for arch in $ARCHS; 
	do  
	    make clean
	    # 现版Xcode支持SDK4.3以上
	    IOSMV="-miphoneos-version-min=4.3"
	    case $arch in
	    arm*)  
	        echo "Building opencore-amr for iPhoneOS $arch ****************"
	        # amr64只支持iOS7
	        if [ $arch == "arm64" ]
	        then
	            IOSMV="-miphoneos-version-min=7.0"
	        fi
	        PATH=`xcodebuild -version -sdk iphoneos PlatformPath`"/Developer/usr/bin:$PATH" \
	        SDK=`xcodebuild -version -sdk iphoneos Path` \
	        CXX="xcrun --sdk iphoneos clang++ -arch $arch $IOSMV --sysroot=$SDK -isystem $SDK/usr/include" \
	        LDFLAGS="-Wl,-syslibroot,$SDK" \
	        ./configure \
	        --host=arm-apple-darwin \
	        --prefix=$DEST/$arch \
	        --disable-shared
	        ;;
	    *)
	        echo "Building opencore-amr for iPhoneSimulator $arch *****************"
	        PATH=`xcodebuild -version -sdk iphonesimulator PlatformPath`"/Developer/usr/bin:$PATH" \
	        CXX="xcrun --sdk iphonesimulator clang++ -arch $arch $IOSMV" \
	        ./configure \
	        --prefix=$DEST/$arch \
	        --disable-shared
	        ;;
	    esac
	    make -j5
	    make install
	done
	
	make clean
	# lipo打包合并静态库
	echo "Merge into universal binary."
	
	for i in $LIBS;
	do
	    input=""
	    for arch in $ARCHS; do
	        input="$input $DEST/$arch/lib/$i"
	    done
	    lipo -create $input -output $DEST/$i
	done 
	
*PS:完整的库文件在我的[github](https://github.com/cxjwin/opencore-amr-0.1.3.git)里面clone后运行opencore_amr.sh即可*

2.编译speex库
这个相对复杂些，因为speex还以来libogg库，所以这里我们先要编译libogg库。  
2.1.编译libogg
shell脚本其实也是大同小异

	#!/bin/sh

	#  build_ogg_ios.sh
	#  
	#
	#  Created by cxjwin on 15-4-19.
	#

	set -xe

	DEVELOPER=`xcode-select -print-path`
	ROOTDIR=`pwd`
	DEST=${ROOTDIR}/../speexLibrary/libogg-1.3.2
	ARCHS="i386 x86_64 armv7 armv7s arm64"
	LIBS="libogg.a"
	
	for arch in $ARCHS;
	do 
	    mkdir -p $DEST/$arch
	done
	
	for arch in $ARCHS; 
	do  
	    make clean
	    IOSMV="-miphoneos-version-min=5.0"
	    case $arch in
	    arm*)  
	        echo "Building opencore-amr for iPhoneOS $arch ****************"
	        if [ $arch == "arm64" ]
	        then
	            IOSMV="-miphoneos-version-min=7.0"
	        fi
	        PATH=`xcodebuild -version -sdk iphoneos PlatformPath`"/Developer/usr/bin:$PATH" \
	        SDK=`xcodebuild -version -sdk iphoneos Path` \
	        CC="xcrun --sdk iphoneos clang -arch $arch $IOSMV --sysroot=$SDK -isystem $SDK/usr/include" \
	        CXX="xcrun --sdk iphoneos clang++ -arch $arch $IOSMV --sysroot=$SDK -isystem $SDK/usr/include" \
	        LDFLAGS="-Wl,-syslibroot,$SDK" \
	        ./configure \
	        --host=arm-apple-darwin \
	        --prefix=$DEST/$arch \
	        ;;
	    *)
	        echo "Building opencore-amr for iPhoneSimulator $arch *****************"
	        PATH=`xcodebuild -version -sdk iphonesimulator PlatformPath`"/Developer/usr/bin:$PATH" \
	        CC="xcrun --sdk iphonesimulator clang -arch $arch $IOSMV" \
	        CXX="xcrun --sdk iphonesimulator clang++ -arch $arch $IOSMV" \
	        ./configure \
	        --prefix=$DEST/$arch \
	        ;;
	    esac
	    make -j
	    make install
	done
	make clean
	echo "Merge into universal binary."
	for i in $LIBS;
	do
	    input=""
	    for arch in $ARCHS; 
	    do
	        input="$input $DEST/$arch/lib/$i"
	    done
	    lipo -create $input -output $DEST/$i
	done
	
*PS:完整的库文件在我的[github](https://github.com/cxjwin/speex_libs.git)里面clone后,找到libogg-1.3.2文件夹运行里面build_ogg_ios.sh即可*  
   
2.3编译speexdsp

	#!/bin/sh
	
	#  build_speexdsp_ios.sh
	#  
	#
	#  Created by cxjwin on 15-4-19.
	#
	
	set -xe
	
	DEVELOPER=`xcode-select -print-path`
	ROOTDIR=`pwd`
	DEST=${ROOTDIR}/../speexLibrary/speexdsp-1.2rc3
	
	ARCHS="i386 x86_64 armv7 armv7s arm64"
	LIBS="libspeexdsp.a"
	
	for arch in $ARCHS;
	do
	    mkdir -p $DEST/$arch
	done
	
	for arch in $ARCHS;
	do  
	    make clean
	    IOSMV="-miphoneos-version-min=5.0"
	    case $arch in
	    arm*)  
	        echo "Building opencore-amr for iPhoneOS $arch ****************"
	        if [ $arch == "arm64" ]
	        then
	            IOSMV="-miphoneos-version-min=7.0"
	        fi
	        PATH=`xcodebuild -version -sdk iphoneos PlatformPath`"/Developer/usr/bin:$PATH" \
	        SDK=`xcodebuild -version -sdk iphoneos Path` \
	        CC="xcrun --sdk iphoneos clang -arch $arch $IOSMV --sysroot=$SDK -isystem $SDK/usr/include" \
	        CXX="xcrun --sdk iphoneos clang++ -arch $arch $IOSMV --sysroot=$SDK -isystem $SDK/usr/include" \
	        LDFLAGS="-Wl,-syslibroot,$SDK" \
	        ./configure --disable-neon  \
	        --host=arm-apple-darwin \
	        --prefix=$DEST/$arch
	        ;;
	    *)
	        echo "Building opencore-amr for iPhoneSimulator $arch *****************"
	        PATH=`xcodebuild -version -sdk iphonesimulator PlatformPath`"/Developer/usr/bin:$PATH" \
	        CC="xcrun --sdk iphonesimulator clang -arch $arch $IOSMV" \
	        CXX="xcrun --sdk iphonesimulator clang++ -arch $arch $IOSMV" \
	        ./configure \
	        --prefix=$DEST/$arch
	        ;;
	    esac
	    make -j
	    make install
	done
	
	make clean
	
	echo "Merge into universal binary."
	
	for i in $LIBS; 
	do
	    input=""
	    for arch in $ARCHS; 
	    do
	        input="$input $DEST/$arch/lib/$i"
	    done
	    lipo -create $input -output $DEST/$i 
	done 

*PS:完整的库文件在我的[github](https://github.com/cxjwin/speex_libs.git)里面clone后,找到speexdsp-1.2rc3文件夹运行里面build_speexsdp_ios.sh即可*    

2.2编译speex  
   
	#!/bin/sh
	
	#  build_speex_ios.sh
	#  
	#
	#  Created by cxjwin on 15-4-19.
	#
	
	set -xe
	
	DEVELOPER=`xcode-select -print-path`
	ROOTDIR=`pwd`
	OGG=${ROOTDIR}/../speexLibrary/libogg-1.3.2
	DEST=${ROOTDIR}/../speexLibrary/speex-1.2rc2
	
	ARCHS="i386 x86_64 armv7 armv7s arm64"
	LIBS="libspeex.a"
	
	for arch in $ARCHS;
	do
	    mkdir -p $DEST/$arch
	done
	
	# --with-ogg是关联之前得libogg库
	for arch in $ARCHS;
	do  
	    make clean
	    IOSMV="-miphoneos-version-min=5.0"
	    case $arch in
	    arm*)  
	        echo "Building opencore-amr for iPhoneOS $arch ****************"
	        if [ $arch == "arm64" ]
	        then
	            IOSMV="-miphoneos-version-min=7.0"
	        fi
	        PATH=`xcodebuild -version -sdk iphoneos PlatformPath`"/Developer/usr/bin:$PATH" \
	        SDK=`xcodebuild -version -sdk iphoneos Path` \
	        CC="xcrun --sdk iphoneos clang -arch $arch $IOSMV --sysroot=$SDK -isystem $SDK/usr/include" \
	        CXX="xcrun --sdk iphoneos clang++ -arch $arch $IOSMV --sysroot=$SDK -isystem $SDK/usr/include" \
	        LDFLAGS="-Wl,-syslibroot,$SDK" \
	        ./configure \
	        --host=arm-apple-darwin \
	        --prefix=$DEST/$arch \
	        --with-ogg=${OGG}/$arch
	        ;;
	    *)
	        echo "Building opencore-amr for iPhoneSimulator $arch *****************"
	        PATH=`xcodebuild -version -sdk iphonesimulator PlatformPath`"/Developer/usr/bin:$PATH" \
	        CC="xcrun --sdk iphonesimulator clang -arch $arch $IOSMV" \
	        CXX="xcrun --sdk iphonesimulator clang++ -arch $arch $IOSMV" \
	        ./configure \
	        --prefix=$DEST/$arch \
	        --with-ogg=${OGG}/$arch
	        ;;
	    esac
	    make -j
	    make install
	done
	
	make clean
	
	echo "Merge into universal binary."
	
	for i in $LIBS; 
	do
	    input=""
	    for arch in $ARCHS; 
	    do
	        input="$input $DEST/$arch/lib/$i"
	    done
	    lipo -create $input -output $DEST/$i 
	done    
	
*PS:完整的库文件在我的[github](https://github.com/cxjwin/speex_libs.git)里面clone后,找到speex-1.2rc2文件夹运行里面build_speex_ios.sh即可*   

&emsp;&emsp;因为speex是依赖libogg库的所以一定要注意编译顺序。speex-1.2rc2以后speexdsp-1.2rc3从speex里面抽离了出来,所以需要单独编译.  
   
&emsp;&emsp;当然这里编译的过程中，自己也学习了下shell脚本。Xcode本身就支持shell脚本，Xcode编译的时候我们就可以把工程的.a文件一并打包成一个，以便于模拟器和真机同时引用，当然这又是另外一个主题了。