# OPTEEOS安装教程(Raspberry 3B+)
本教程旨在树莓派中安装OPTEEOS，如需在其他平台安装，请参考[OPTEEOS官方安装教程](https://optee.readthedocs.io/en/latest/building/index.html)。

安装平台：

+ Raspberry 3B+
+ 

# 安装环境配置
TrustZone将整个内存分为REE部分和TEE部分，因此需要在TF卡中安装两个操作系统。这就需要主机在TF卡上预先安装，之后将TF卡放入树莓派。在主机上需要的环境配置有(Ubuntu20.04)：

  sudo apt install \
  
  android-tools-adb \

  android-tools-fastboot \

  autoconf \

  automake \
  
  bc \
  
  bison \
  
  build-essential \
  
  ccache \
  
  cscope \
  
  curl \
  
  device-tree-compiler \
  
  expect \
  
  flex \
  
  ftp-upload \
  
  gdisk \
  
  iasl \
  
  libattr1-dev \
  
  libcap-dev \
  
  libfdt-dev \
  
  libftdi-dev \
  
  libglib2.0-dev \
  
  libgmp3-dev \
  
  libhidapi-dev \
  
  libmpc-dev \
  
  libncurses5-dev \
  
  libpixman-1-dev \
  
  libssl-dev \
  
  libtool \
  
  make \
  
  mtools \
  
  netcat \
  
  ninja-build \
  
  python3-crypto \
  
  python3-cryptography \
  
  python3-pip \
  
  python3-pyelftools \
  
  python3-serial \
  
  rsync \
  
  unzip \
  
  uuid-dev \
  
  xdg-utils \
  
  xterm \
  
  xz-utils \
  
  zlib1g-dev

# 安装流程
由于REE和TEE本质上是属于同一层面的。并不存在包含与被包含的关系，为了较好的隔绝，需要将TF卡划为两个分区，分别存放REE和TEE。
## 安装REE
本文在购买树莓派用TF卡时已经预安装Ubuntu20.04，若没有预安装，则只需去Ubuntu官网下载对应版本的镜像，执行以下命令即可烧录到SD卡：

`wget Ubuntu对应版本镜像网址`

`xzcat 镜像所在位置 | sudo dd of=/dev/disk2  bs=4M`

当然也可安装其他REE，本文没有进行测试，不再赘述。

## 安装TEE
如题目所述，本文选用的TEE为OPTEEOS。树莓派要想使用OPTEEOS，需要先在主机中完成编译，之后运行其自带的移植脚本移植到TF卡。流程为：

下载OPTEEOS：

`git clone https://github.com/xuanxuanblingbling/RPi3_OPTEE_3.1.git`

进入编译目录：

`cd RPi3_OPTEE_3.1/build`

编译工具链：

`make -j2 toolchains`

开始编译。注意，nproc是为编译分配的处理器核数，越大编译速度越快。但不要超过自己cpu最大核数，一般分配8个即可：

`make -j nproc`

将编译好的OPTEEOS复制到TF卡中。其中，需要修改脚本文件中TF卡的路径，之后再运行脚本文件：

```
cd ..
vi ./copy_to_sdcard.sh 
./copy_to_sdcard.sh
```