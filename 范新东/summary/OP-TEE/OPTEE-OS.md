目录
- [概述](#概述)
- [C语言项目架构的学习方法（不一定对）](#c语言项目架构的学习方法不一定对)
  - [关于源文件和头部文件](#关于源文件和头部文件)
    - [函数声明和实现](#函数声明和实现)
    - [include机制](#include机制)
  - [关于Makefile分析](#关于makefile分析)
    - [Makefile编写结构](#makefile编写结构)
    - [OPTEE-OS的Makefile结构](#optee-os的makefile结构)
      - [optee\_os/Makefile](#optee_osmakefile)
      - [mk/checkconf.mk](#mkcheckconfmk)
      - [core/core.mk](#corecoremk)

# 概述
本文将会记录学习OP-TEE OS（v3.21.0）的组织架构。

# C语言项目架构的学习方法（不一定对）
## 关于源文件和头部文件

### 函数声明和实现
通常情况下，一个项目的头文件会放到```include```文件夹下，源文件会放到```src(source)```文件夹下。头文件存放函数的声明，源文件存放函数的实现。源文件在实现头文件的某个函数时，需要include该头文件，include机制下述。

需要注意的是，在程序中不同的源文件中定义的同名函数会被链接器认为是不同的函数，因此需要确保在程序中只有一个定义相同的函数。一种常见的做法是将函数的实现放在一个单独的源文件中，然后在需要使用该函数的源文件中包含该函数的声明头文件即可。

### include机制
需要注意的是，include属于预处理部分。

在进行编译时，编译器首先进行预处理部分，即根据搜索路径将include中的内容都搜索到。搜索路径一般为：
+ 当前目录：编译器会先在当前源文件所在的目录中搜索头文件。
+ 系统目录：编译器会在系统默认的头文件目录中搜索头文件。这些目录通常包括操作系统的标准库目录、编译器的默认库目录等。
+ 环境变量：可以通过设置环境变量来指定额外的头文件搜索路径。例如，可以设置环境变量"C_INCLUDE_PATH"来指定额外的头文件搜索路径。
+ 命令行选项：编译器还可以通过命令行选项来指定头文件搜索路径。例如，可以使用"-I"选项来指定额外的头文件搜索路径。

所以在进行我们自己的项目时，要注意源文件和头文件相应的位置关系。

## 关于Makefile分析
Makefile脚本一般用于处理大型项目的统一编译的问题。不像一个小程序gcc一下就行。本文主要分析文件结构及各个关键字的作用。

### Makefile编写结构
对Makefile文件包括.mk文件分析时，可以按照一下步骤进行：
+ 查看makefile的文件名和路径，确定makefile的作用和依赖关系。
+ 查看makefile中定义的变量，包括系统变量和自定义变量，确定变量的用途和取值。
+ 查看makefile中定义的规则，包括目标、依赖关系和命令，确定规则的作用和执行顺序。
+ 查看makefile中定义的伪目标和特殊目标，包括.PHONY、.DEFAULT、.SUFFIXES等，确定它们的作用和执行顺序。
+ 查看makefile中定义的函数和宏，包括(shell)、(foreach)、$(if)等，确定它们的用途和参数。
+ 查看makefile中定义的注释，了解makefile的编写者对makefile的说明和解释。
+ 通过make命令执行makefile，查看makefile的执行过程和结果，验证makefile的正确性。

当我们在编写Makefile时，需要遵循以下原则：
+ 确定要编译的目标，以及它们之间的依赖关系
+ 在编写时，使用变量管理Makefile中的各种路径、编译器选项等

### OPTEE-OS的Makefile结构
#### optee_os/Makefile
该目录是位于optee_os最顶层的编译文件，它是编译opteeos的起始文件。即在要开始编译opteeos时，该文件是最先被调用的。

文件的主要作用是定义一些Global变量并且将各个部分连接起来，具体要编译的功能要看各个组件的作用。

该文件定义了很多变量，并且include所要用到的mk文件:

+ `mk/checkconf.mk` # 用于检查配置是否完整
+ `core/core.mk` # 用于编译内核的文件
+ `ldelf/ldelf.mk` # TODO
+ `ta/ta.mk` # 用于编译ta的文件
+ `ta/mk/build-user-ta.mk` # 用于编译用户端ta的文件
+ `mk/cleandirs.mk` # 用于make clean的文件

#### mk/checkconf.mk
该文件定义了一系列的函数，用于生成TEE的配置文件，这些函数为：
+ `check-conf-h`: 创建一个.h的头文件，其中包含TEE的配置信息，这个函数会从MakeFile中获取所有的TEE配置变量，并将这些变量转换为c语言的宏定义，并输出到h文件中。
+ `check-conf-cmake`: 创建一个cmake文件，作用同上。
+ `check-conf-mk`: 创建一个make文件，作用同上。
+ `mv-if-changed`: 比较两个文件的内容是否相同，相同则删除第一个文件；不同则将第一个文件命名为第二个文件。
+ `cfg-vars-by-prefix`: 根据给定的前缀，获取所有以该文件开头的变量。
+ `cfg-make-define`: 将给定的变量转换为C语言的宏定义。
+ `cfg-cmake-set`: 将给定的变量转换为CMake的变量设置。
+ `cfg-one-enabled`: 检查给定的一组变量中是否至少有一个变量的值为'y'
+ `cfg-all-enabled`: 检查给定的一组变量中是否全部变量的值为'y'
+ `cfg-depends-all`: 如果一个变量依赖的变量中至少有一个变量的值为'n'，则将该变量设置为'n'
+ `cfg-depends-one`: 如果一个变量依赖的变量中所有变量的值为'n'，则将该变量设置为'n'
+ `cfg-enable-all-depends`: 将依赖于一个变量的所有变量设置为'y'
+ `cfg-check-value`: 检查变量是否有给定的值，如果不是则发出错误消息。
+ `force`: 将变量设置为给定的值，如果该变量已经被设置为不同的值，则发出错误消息。

#### core/core.mk
该文件用于编译TEE的核心模块。该文件包含了`cleanvars.mk`文件，然后设置了当前的submodule为core，并包含了相关的配置文件和Makefile文件。接下来，定义了一些变量和标志，例如`cppflags$(sm)`,`cflags$(sm)`,`aflags$(sm)`等。

然后，该文件还定义了一些库，如`utils`、`mbedtls`、`tomcrypt`、`fdt`、`zlib`、`unw`和`scmi-server`等，这些库都是针对TEE的一些特定需求而编写的，例如加密、压缩、调试等。

最后，该文件还包含了一些编译规则，例如compile.mk和link.mk，用于编译和链接TEE核心模块。