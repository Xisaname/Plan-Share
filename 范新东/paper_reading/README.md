# 论文导读
可以根据本README来快速了解论文，本文是按照论文上传的时间顺序进行快读导读。
## 第一篇 A Hardware-Software Co-design for Efficient Intra-Enclave Isolation
**FROM**: USENIX Security 22: 31st USENIX Security Symposium.

本文是在SGX enclave基础上再建立了一层TEE，名为Light-Enclave。

+ **问题**：SGX enclave TCB面临膨胀问题，需要减小攻击面；同时跨enclave通信十分耗时，计算效率低下。
+ **目的**：减小攻击面，提高执行效率。
+ **工作**：实现了enclave内部的隔离，从逻辑上减小了攻击面；同时实现了enclave内部进程之间的通信，不再需要跨enclave进行进程通信。

## 第二篇 Denial-of-Service on FPGA-based Cloud Infrastructures
**FROM**: CHES 22: International Conference on Cryptographic Hardware and Embedded Systems.

本文提出了针对FPGA云端部署实例的一种DoS攻击方法，并且设计了一种在大型数据中心部署的病毒扫描框架。

+ **问题**：针对FPGA云服务的DoS攻击可以使得FPGA板宕机数个乃至几十个小时，大规模此类攻击对云服务将是是一种不小的损失。
+ **目的**：提高云端FPGA的防护能力。
+ **工作**：提出了一种DoS攻击：power-hammering。这种攻击可以将FPGA板载功率最高提升至2.7kw，达到破坏效果。同时提出了一种基于开源FPGADefender开发的FPGA病毒扫描框架，可以部署到大型数据中心进行快速病毒扫描。