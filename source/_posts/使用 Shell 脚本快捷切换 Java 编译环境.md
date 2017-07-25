---
title: 使用脚本快捷切换编译环境
date: 2016-10-29 16:50:56
categories: 工具分享
tags: [Shell, JDK, Linux]
keywords: Shell, JDK, Linux
comments: true
description: 猿最近在 Ubuntu 14.04.5 环境下搞 Android 代码编译，但是由于重复开窗口、切换分支、切换 JDK 版本，要重复敲很多命令。深感不便，于是自己参考网上的语法，写了个简单脚本，现在share给大家，希望大家喜欢（＊＠ο＠＊）　哇～。
---

猿最近在Ubuntu 14.04.5环境下搞Android代码编译，但是由于重复开窗口、切换分支、切换JDK版本，要重复敲很多命令。深感不便，于是自己参考网上的语法，写了个简单脚本，现在share给大家，希望大家喜欢（＊＠ο＠＊）　哇～。

### 总入口build.sh

    #!bin/sh
    echo -e "#### open target folder ####\n 1.branch1\n 2.branch2\n 3.branch3\n 4.branch4\n"
    echo -n  "### input the type:"
    read target
    case "$target" in
        1)
        	echo -e "\033[1;32m build target is branch1 \033[0m"
            . /home/user/script/build_branch1_script.sh
    	;;
        2)
        	echo -e "\033[1;32m build target is branch2 \033[0m"
            . /home/user/script/build_branch2_script.sh
        	;;
        3)
        	echo -e "\033[1;32m build target is branch3 \033[0m"
        	. /home/user/script/build_branch3_script.sh
        	;;
        4)
        	echo -e "\033[1;32m build target is branch4 \033[0m"
        	. /home/user/script/build_branch4_script.sh
        	;;
    esac

### 编译branch1版本脚本:build_branch1_script.sh

    #!bin/sh
    echo "### open target folder ###"
    cd /home/user/src/android-branch1-dev
    echo -e "#### change java environment ####"
    sudo sed -ri 's#^.*JAVA_HOME=.*$#export JAVA_HOME=/usr/lib/java/jdk1.6.0_41#' /etc/profile
    echo -e "#### source the profile"
    . /etc/profile
    echo -e "#### output java version"
    java -version
    echo "### source envsetup.sh ###"
    . build/envsetup.sh
    echo "### lunch source file ###"
    lunch branch1-userdebug

### 编译branch2版本脚本:build_branch2_script.sh

    #!bin/sh
    echo -e "#### open target folder ####"
    cd /home/user/src/android-branch2-dev
    echo -e "#### change java environment ####"
    sudo sed -ri 's#^.*JAVA_HOME=.*$#export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64#' /etc/profile
    echo -e "#### source the profile"
    . /etc/profile
    echo -e "#### output java version"
    java -version
    echo -e "#### source envsetup.sh ####"
    . build/envsetup.sh
    echo -e "#### lunch source file ####"
    lunch branch2-userdebug


### 编译branch3版本脚本:build_branch3_script.sh

    #!bin/sh
    echo "##### open target folder #####"
    cd /home/user/src/android-branch3-dev
    echo -e "#### change java environment ####"
    sudo sed -ri 's#^.*JAVA_HOME=.*$#export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64#' /etc/profile
    echo -e "#### source the profile"
    . /etc/profile
    echo -e "#### output java version"
    java -version
    echo "##### source envsetup.sh #####"
    . build/envsetup.sh
    echo "##### lunch source file #####"
    lunch branch3-userdebug

### 编译branch4版本脚本:build_branch4_script.sh

    #!bin/sh
    echo "##### open target folder #####"
    cd /home/user/src/android-branch4-dev
    echo -e "#### change java environment ####"
    sudo sed -ri 's#^.*JAVA_HOME=.*$#export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64#' /etc/profile
    echo -e "#### source the profile"
    . /etc/profile
    echo -e "#### output java version"
    java -version
    echo "##### source envsetup.sh #####"
    . build/envsetup.sh
    echo "##### lunch source file #####"
    lunch branch4-userdebug

这几段脚本组合起来的作用其实特别简单，就是打开对应的分支目录，同时修改JAVA_HOME的配置，启动Android编译。当然有不少同学会说Ubuntu切换JDK可以使用update-alternatives进行切换。但是这里本猿要说明一下，这个命令配置也挺繁琐的，而且猿配置了以后，发现切换了使用java -version输出的一直是默认版本，切换不过来╮（╯▽╰）╭，后来就放弃了，改用最熟悉的手动切换，而且现在配合自己的脚本，用的还挺顺手。当然，如果有人对于Ubuntu的java多版本切换有心得或者对于脚本的改进和书写有什么建议，都可以给猿留言哦～Ｏ（∩＿∩）Ｏ哈哈～
