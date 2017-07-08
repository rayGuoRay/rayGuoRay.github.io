---
title: 使用脚本快捷切换编译环境改良
date: 2016-11-18 21:37:56
categories: 工具分享
tags: [Shell, JDK, Linux]
keywords: Shell, JDK, Linux
comments: true
---

猿之前自己搞了个脚本([工作分享｜使用脚本快捷切换编译环境](http://www.jianshu.com/p/989d0d3fb29d))来切换自己的编译环境，提高工作效率。但是随着近期维护的项目逐渐变多，原来的脚本已不满足需求。因此，猿对自己的脚本进行了一定的改造，贡献出来供大家参考。

### 执行脚本(build.sh)
-  脚本说明
脚本中会以source方式引入自己定义的一个build_util.sh脚本，此脚本为执行脚本提供输出提示信息方法和过滤用户输入的方法，最后用户输入的文件目录选项，模块选项，操作的文件夹选项都会通过build_util.sh脚本进行处理和分发。

- 脚本源码
      #!bin/sh                                                                    
      . /home/script/build_util.sh
      # open target project #
      funOperationHint 1
      read chooseProject
      # switch operation moudle
      funOperationHint 2
      read chooseModule
      # open folder
      funOperationHint 3
      read chooseFolder
      # operation target folder
      funOperationTarget ${chooseProject} ${chooseModule} ${chooseFolder}

### 工具脚本(build_util.sh)
-  脚本说明
脚本中会同样以source方式引入每个分支的脚本，此脚本除了输出提示信息外，最后会通过funOperationTarget()方法选择不同的脚本方法去处理。

**这里需要注意的是，以source方式引入的脚本中，最好不要有重名方法。**因为猿最初是每个分支脚本中定义相同的方法然后去执行发现执行结果错误。后来想想原因也简单，脚本source相当于include, 本身直接调用一个同名方法，脚本也不确认究竟要执行哪个脚本中的方法，肯定执行不正确。**

- 脚本源码
      #!bin/sh
      . /home/script/build_branch1.sh
      . /home/script/build_branch2.sh
      . /home/script/build_branch3.sh
      . /home/script/build_branch4.sh
      # hint message about operation project name
      funOperationHint() {
          case $1 in
              1)
                  echo -e "\033[1;33m
                  ## Choose Operation Project ##
                  |     1. Device A           |
                  |     2. Device B           |
                  |     3. Device C           |
                  |     4. Device D         |
                  ##############################
                  \033[0m"
                  ;;
              2)
                echo -e "\033[1;33m
                #### Choose Operation Project  ####
                |   1. Module ModuleA    |
                |   2. Module ModuleB         |
                ###################################
                \033[0m"
                ;;
            3)
                echo -e "\033[1;33m
                ##  Choose Operation Type  ##
                |     1. Folder package     |
                |     2. Folder out         |
                ############################
                \033[0m"
                ;;
                esac
                echo -e "\033[1;35m*****input the switch key*****\033[0m"
        }
        # target to handle the command
        funOperationTarget() {
            case $1 in
               1)
                  handleOperationOnA $2 $3
                  ;;
              2)
                  handleOperationOnB $2 $3
                  ;;
              3)
                  handleOperationOnC $2 $3
                  ;;
              4)
                  handleOperationOnD $2 $3
                ;;
           esac
      }

### 分支脚本(build_branch1.sh)
-  脚本说明
分支脚本中主要实现了handleOperationOnC方法的实现，主要用于判断模块与操作目录，根据输入的模块选项进行编译或者打开目录。由于实现部分代码比较类似，主要列出一项用于参考。

- 脚本源码
      #!bin/sh
      # handle operation
      handleOperationOnA() {
          case $1 in
              1)
                  handleFolderA $2
                  ;;
              2)
                  handleFolderB $2
                  ;;
          esac
      }

      # handle FolderA
      handleFolderA() {
           case $1 in
               1)
                  cd folderA/folderC
                   ;;
               2)
                  cd /folderD/folderE
                  ;;
         esac
      }

      # handleFolderB
      handleT1FileManager() {
           case $1 in
              1)
                sudo sed -ri 's#^.*JAVA_HOME=.*$#export JAVA_HOME=/usr/lib/java/jdk1.6.0_41#' /etc/profile
                . /etc/profile
                . build/envsetup.sh
                lunch branch-userdebug
                cd /home/folderF
                ;;
            2)
                cd FolderG
            ;;
        esac
      }
