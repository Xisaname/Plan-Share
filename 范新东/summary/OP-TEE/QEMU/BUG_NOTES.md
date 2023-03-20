目录
- [TEE\_ERROR\_ITEM\_NOT\_FOUND](#tee_error_item_not_found)
  - [bug描述](#bug描述)
  - [进展](#进展)
    - [1](#1)
    - [2](#2)

# TEE_ERROR_ITEM_NOT_FOUND
## bug描述
当使用optee自带的benchmark程序进行程序测试时，出现了以下问题：

client端：
```
[Benchmark] ERROR: TEEC_OpenSession: 0xffff0008
```
tee端：
```
tee_ta_invoke_command:773 Error: ffff0008 of 4
```
## 进展

### 1
目前将问题定位到`optee_project/optee_os/out/arm/export-ta_arm64/include/tee_api_defines.h`中的`TEE_ERROR_ITEM_NOT_FOUND`，该问题对应的代码就是`0xffff0008`，初步分析就是缺少某些资源的问题。

通过查找[资料](http://github.com/OP-TEE/optee_os/pull/2847),发现出现上述报错代码的问题很多，并且很多是跟硬件配置息息相关的。

### 2
通过secure端调试窗口，发现报错的原因是UUID匹配不上。
如下图所示：

可以看到，我们传入的UUID和系统从session中查找的UUID并不匹配，从而报错。具体的代码可看`optee_os/core/kernel/tee_ta_manager.c`中的`tee_ta_context_find`函数。

因此可以查看两者不同的原因。