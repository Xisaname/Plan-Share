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
      - [ldelf/ldelf.mk](#ldelfldelfmk)
      - [mk/lib.mk](#mklibmk)
      - [mk/compile.mk](#mkcompilemk)

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
sm: sub moudle是子模块的名称

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

最后，该文件还包含了一些编译规则，例如compile.mk和link.mk，用于编译和链接TEE核心模块，编译规则包括：
+ compile.mk：编译规则文件，用于编译源文件。
+ link.mk：链接规则文件，用于链接目标文件。

**详细描述**

该文件包含了许多include语句，用于包含其他Makefile文件和配置文件。这些文件包括：
+ cleanvars.mk：用于清除变量的Makefile文件。
+ conf.mk：平台配置文件。
+ config.mk：TEE核心模块配置文件。
+ arch/(ARCH)/(ARCH).mk：架构配置文件。
+ crypto.mk：加密库配置文件。
+ lib.mk：库编译规则文件。
+ subdir.mk：子目录编译规则文件。
+ compile.mk：编译规则文件。
+ link.mk：链接规则文件。

这个Makefile文件定义了许多变量和宏，如`sm`、`arch-dir`、`platform-dir`、`cppflags`、`cflags`、`aflags`等。这些变量和宏用于设置编译选项和路径等。

+ `sm`: 当前子模块，用于设置编译选项和路径等。
+ `arch-dir`: 架构目录，用于设置架构相关的路径。
+ `platform-dir`: 平台目录，用于设置平台相关的路径。
+ `cppflags`、`cflags`、`aflags`: 编译选项，用于设置编译器的选项，如预编译宏、头文件路径、优化级别等。

这个Makefile文件包含了许多条件语句，如ifeq和ifdef，用于根据配置文件中的选项设置编译选项。这些选项包括：

+ CFG_PAGED_USER_TA：是否启用用户空间分页。
+ CFG_WITH_PAGER、CFG_WITH_USER_TA：是否支持页表管理、用户态应用程序等。
+ CFG_SCMI_SCPFW：是否支持系统管理互连协议（SCMI）。
+ CFG_CORE_STACK_PROTECTOR、CFG_CORE_STACK_PROTECTOR_STRONG、CFG_CORE_STACK_PROTECTOR_ALL：是否启用堆栈保护。
+ CFG_CORE_SANITIZE_UNDEFINED、CFG_CORE_SANITIZE_KADDRESS：是否启用未定义行为、内核地址污点分析等。
+ CFG_CORE_DEBUG_CHECK_STACKS：是否启用堆栈检查。
+ CFG_SYSCALL_FTRACE：是否启用系统调用跟踪。
+ CFG_TEE_CORE_LOG_LEVEL：日志级别。
+ CFG_TEE_CORE_MALLOC_DEBUG：是否启用内存调试。
+ CFG_TEE_CORE_DEBUG：是否启用调试模式。
+ CFG_CORE_DYN_SHM、CFG_CORE_RESERVED_SHM：是否启用动态共享内存、保留共享内存等。
+ CFG_CRYPTOLIB_NAME：加密库名称。
+ CFG_CRYPTOLIB_DIR：加密库目录。
+ CFG_CRYPTO_RSASSA_NA1：RSASSA-PKCS1-v1_5签名算法是否支持NA1补丁。

#### ldelf/ldelf.mk
该文件的作用是编译和链接一个名为ldelf的子模块，同时定义了该子模块所依赖的库和编译选项。

具体来说，该makefile文件完成以下操作：

+ 引入清除变量的makefile文件mk/cleanvars.mk；
+ 定义当前子模块的名称和输出目录；
+ 定义当前子模块的编译选项，包括cppflags、cflags和aflags三个选项；
+ 根据核心架构的不同，为当前子模块设置不同的编译选项；
+ 为当前子模块添加额外的编译选项，包括包含conf-file文件、定义TRACE_LEVEL和__LDELF__宏；
+ 定义当前子模块使用的交叉编译器和编译器类型，并引入对应的makefile文件；
+ 定义当前子模块中的多个库，包括库名称、库目录和makefile文件；
+ 定义当前子模块的子目录，并引入对应的makefile文件；
+ 引入编译相关和链接相关的makefile文件，完成编译和链接操作。

#### mk/lib.mk
该文件是编译的通用文件。

该Makefile文件的作用是编译和链接库文件，生成静态库文件和共享库文件，并设置生成的库文件的路径、依赖关系和清理过程。它还包含了对子目录和源文件的编译规则。该Makefile文件的目的是为了方便管理和构建库文件，使得编译和链接库文件的过程更加自动化和简单化。

该文件定义了下列一系列变量：
+ subdirs：设置为libdir的值，用于在后面的子目录中进行编译。
+ lib-libfile：将输出库文件的路径设置为/(out−dir)/(base-prefix)/(libdir)/lib(libname).a。
+ lib-shlibfile：如果CFG_ULIBS_SHARED为y，则设置共享库文件的路径为/(out−dir)/(base-prefix)/(libdir)/lib(libname).so。
+ lib-shlibstrippedfile：设置剥离后的共享库文件的路径为/(out−dir)/(base-prefix)/(libdir)/lib(libname).stripped.so。
+ lib-shlibtafile：设置共享库文件的TA文件路径为/(out−dir)/(base-prefix)/(libdir)/(libuuid).ta。
+ lib-libuuidln：设置链接到共享库文件的符号链接路径为/(out−dir)/(base-prefix)/(libdir)/(libuuid).elf。
+ lib-needed-so-files：设置依赖的共享库文件的路径。
+ cleanfiles：设置要清除的文件列表。
+ libfiles：设置要生成的库文件列表。
+ libdirs：设置库文件目录列表。
+ libnames：设置库文件名称列表。
+ libdeps：设置库文件依赖列表。

#### mk/compile.mk
该文件定义了一个函数process_srcs，该函数用于处理源文件并生成相应的编译规则。该函数接受两个参数，分别是源文件列表和编译规则列表。该函数首先将源文件列表中的每个文件添加到objs变量中。然后，它为每个源文件生成一个依赖文件和一个编译命令文件，并将它们添加到cleanfiles变量中。最后，它根据源文件的类型为每个文件生成相应的编译规则。

之后，该文件定义了一个生成汇编代码的函数_gen-asm-defines-file，该函数接受四个参数，分别是.c文件名、.h文件名、输出的.S文件名和附加的依赖项。该函数将为.c文件生成一个.S文件，并将其添加到cleanfiles变量中。它还为.S文件生成一个依赖文件和一个编译命令文件，并将它们添加到cleanfiles变量中。

在生成编译规则时，_gen-asm-defines-file函数使用CC编译器，并使用相应的编译选项。它还使用-fverbose-asm选项生成详细的汇编代码。

该Makefile使用foreach循环调用process_srcs函数来处理所有的源文件，包括srcs、gen-srcs和spec-srcs。这些源文件将编译成objs变量中列出的目标文件。它还指定了conf-file和additional-compile-deps作为所有目标文件的依赖项。

