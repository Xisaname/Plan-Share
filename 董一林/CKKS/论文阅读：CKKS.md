## 论文阅读：Homomorphic Encryption for Arithmetic of Approximate Numbers

### Abstract

- 构建近似算术的同态加密方案，支持加密信息的近似加法和乘法。

  构建一个管理明文大小的重定程序，将密码文本截断成较小的模数，导致明文四舍五入。

  在包含主要信息的重要数字之后添加噪音，解密结构以预先确定的精度输出一个明文的近似值。

- 基于RLWE提出一种新的批处理技术

- 在设计的结构中，因为重新缩放程序，密码文本模数的位数与被评估电路的深度呈线性增长。评估过程中的精度损失受电路深度的限制。

### 1.Introduction

- 同态加密Homomorphic encryption(HE)技术可以在不解密的情况下对加密数据进行同态操作，HE可用于对加密的金融、医疗或基因组数据的算法评估。

- 大多数现实世界的数据都含有一些与真实值不同的误差。如测量值与真实值的观察误差。在实践中，应该将数据离散化（量化）为一个近似值，如浮点数，以便在计算机系统中用有限的比特数表示。

  因此在一定情况下，一个近似值可以代替原始数据，小的舍入误差对计算结果没有太大影响。为了提高近似算术的效率，存储一些有意义的数字（如最有意义的比特MSBs）并在它们之间进行算术运算，得到的数值通过去除一些不准确的最小有效位再进行四舍五入，以保持尾数的位数大小。

  这种舍入运算被认为很难在HE上执行，因为它不是简单地表示为小度数多项式。

- 本文指出现有的HE方案的解密结构并不适合离散空间的算术。

  证明BGV型HE方案、FV型HE方案中，$m_1+m_2$和$m_1m_2$的MSB在同态操作中被插入的错误$e_i$所破坏。

  [<img src="https://s1.ax1x.com/2022/12/06/zcmMJf.png" alt="zcmMJf.png" style="zoom:80%;" />](https://imgse.com/i/zcmMJf)

#### 1.1Homomorphic Encryption for Approximate Arithmetic

- 本文的目的：提出一种在HE上进行高效近似计算的方法。

- 主要思想：将加密噪声作为近似计算期间发生的错误的一部分。

  用密钥$sk$对信息$m$的加密$c$将有一个形式为$\langle c, s k\rangle=m+e(\bmod q)$的解密结构，其中$e$是插入的一个小误差，以保证硬性假设的安全性，如learning with errors(LWE)问题、the ring-LWE(RLWE)问题和NTRU问题。

  如果$e$与信息相比足够小，这个噪声就不会破坏$m$中的有效数字，整个数值$m^{\prime}=m+e$可以在近似算术中取代原始信息。可以在加密钱对信息乘一个比例系数，以减少加密噪声带来的精度损失。

- 对于同态操作，总是保持解密结构与密文模数相比足够小，以便计算结果仍然小于$q$。但是消息的比特大小随着电路深度的增加呈指数级增长，而没有四舍五入。

  为解决这个问题，提出重缩放技术操作密码文本的信息。

  对于一个m的加密c，使$\langle c, s k\rangle=m+e(\bmod q)$，重缩放程序输出一个密码文本$\left.\mid p^{-1} \cdot c\right\rceil \quad(\bmod q / p)$，这是一个有效的$m / p$的加密，噪声约为$e / p$，减少了密文模数的大小，从而消除了位于信息LSB中的错误，类似于定点/浮点运算的舍入步骤，同时几乎保留了明文的精度。

  同态运算和重新缩放的组合模仿了普通的近似算术（如图2）,所以密码文本模数的比特大小与电路的深度呈线性增长，而不是指数增长。而且精度上最优，与未加密的浮点运算相比，产生的信息精度损失最多只有一个比特。

  [<img src="https://s1.ax1x.com/2022/12/06/zcmaF0.png" alt="zcmaF0.png" style="zoom:80%;" />](https://imgse.com/i/zcmaF0)

#### 1.2Encoding Technique for Packing Messages

- 设$\mathbb{H}=\left\{\left(z_j\right)_{j \in \mathbb{Z}_M^*}: z_{-j}=\overline{z_j}, \forall j \in \mathbb{Z}_M^*\right\} \subseteq \mathbb{C}^{\Phi(M)}$

  令T是乘法组$\mathbb{Z}_M^*$ 的一个子组，满足$\mathbb{Z}_M^* / T=\{\pm 1\}$

  方案的原生明文空间是环形环$\mathcal{R}=\mathbb{Z}[X] /\left(\Phi_M(X)\right)$中的多项式集合，其大小由密码文模数决定。

  解码程序首先通过典型的嵌入映射$\sigma$将明文多项式$m(X) \in \mathcal{R}$转化为复数向量$\left(z_j\right)_{j \in \mathbb{Z}_M^*} \in \mathbb{H}$，然后使用自然投影$\pi: \mathbb{H} \rightarrow \mathbb{C}^{\phi(M) / 2}$将其发送至向量$\left(z_j\right)_{j \in T}$。

  编码方法几乎是解码程序的倒数，但需要一个离散化的舍入算法，以便其输出成为一个积分多项式。

  编码函数总结如下：

$$
\begin{aligned}
& \mathbb{C}^{\phi(M) / 2} \stackrel{\pi^{-1}}{\longrightarrow} \quad \mathbb{H} \stackrel{\lfloor\cdot\rceil_{\sigma(\mathcal{R})}^{\longrightarrow}}{\longrightarrow} \sigma(\mathcal{R}) \quad \stackrel{\sigma^{-1}}{\longrightarrow} \quad \mathcal{R} \\
& \boldsymbol{z}=\left(z_i\right)_{i \in T} \longmapsto \pi^{-1}(\boldsymbol{z}) \longmapsto\left\lfloor\pi ^ { - 1 } ( \boldsymbol { z } ) | _ { \sigma ( \mathcal { R } ) } \longmapsto \sigma ^ { - 1 } \left(\left\lfloor\left.\pi^{-1}(\boldsymbol{z})\right|_{\sigma(\mathcal{R})}\right)\right.\right. \\
&
\end{aligned}
$$

​		其中$\lfloor\cdot\rceil_{\sigma(\mathcal{R})}$表示对$\sigma(\mathcal{R})$中接近的元素进行四舍五入

#### 1.3Homomorphic Evaluation of Approximate Arithmetic

本方案的一个重要特点是同态评估中的精度损失受电路深度的约束，与未加密的近似算术相比，最多只多一个比特。

### 2.Preliminaries

2.1Basic Notation

2.2The Cyclotomic Ring and Canonical Embedding

2.3Gaussian Distributions and RLWE Problem

### 3.Homomorphic Encryption for Approximate Arithmetic

- 提出一种构建HE方案的方法，用于加密数据的近似算术。给定$m_1$和$m_2$的加密，这个方案允许我们安全地计算$m_1+m_2$和$m_1m_2$的近似值的加密，并有预定的精度。
- 主要思想是把RLWE问题的插入噪声作为近似计算中发生的错误的一部分。最重要的特点是明文的舍入操作，就像使用浮点数的普通近似计算一样，舍入操作删除了信息的一些LSB，并在数字的大小和精度损失之间进行了权衡
- 具体构造方案基于BGV

#### 3.1Decryption Structure of Homomorphic Encryption for Approximate Arithmetic

- 大多数现有的HE方案是在一个模数空间上进行操作，如$\mathbb{Z}_t$和$\mathbb{Z}_t[X] /\left(\Phi_M(X)\right)$。他们的目的是计算一个密码文本，该文本在同态计算后对所产生的信息的一些LSB进行加密。

  例如，在BGV型方案中，明文被置于密码文模的最低位，也就是说，一个消息的m相对于一个秘密sk的加密c有一个解密结构的形式$\langle c, s k\rangle=m+t e(\bmod q)$。$m_1、m_2$的加密乘法保留了$m_1m_2$的一些LSB（即$\left[m_1 m_2\right]_t$），而其MSB（即$\left\lfloor m_1 m_2 / t\right\rceil$）则被错误破坏。

- CKKS的目标是对加密数据进行近似算术，或者说在同态运算后计算出结果信息的MSBs。

  主要思想是在输入信息的重要数字后面添加一个加密噪声。

  CKKS方案有一个形式为$\langle\boldsymbol{c}, s k\rangle=m+e(\bmod q)$的解密结构，针对一些小的误差e。

  插入这个加密误差e以保证方案的安全性，但它将被视为近似计算过程中出现的误差。

  解密算法的输出将被视为具有高精度的原始信息的近似值。

  在同态运算中，明文的大小与模数相比将足够小，因此算术计算的结果仍小于密文模数。

- 有一些问题值得考虑：计算错误的上限并预测结果值的精度；如果计算乘法深度为L的电路而不对信息进行四舍五入，那么输出值的比特将随L呈指数级增长，为此我们将中间值除以一个基数，使信息的大小几乎保持不变，并使所需的密文模数与深度L成线性。

#### 3.2Plaintext Encoding for Packing

- **Intuition**

  在编码过程中，舍入错误可能会破坏信息的有效数字，因此在四舍五入前对明文乘以一个比例系数$\Delta \geq 1$，以保持精度。

  编码/解码算法如下：

  [<img src="https://s1.ax1x.com/2022/12/06/zcJ2W9.png" alt="zcJ2W9.png" style="zoom: 80%;" />](https://imgse.com/i/zcJ2W9)

#### 	3.3Leveled Homomorphic Encryption Scheme for Approximate Arithmetic

​		固定基数$p>0$和模数$q_0$，并令$q_{\ell}=p^{\ell} \cdot q_0$，$0<\ell \leq L$。对于安全参数$\lambda$，选择安全参数$M=M\left(\lambda, q_L\right)$作为环状多项式。对于级别$0<\ell<L$，一个级别$\ell$的密码文本是$\mathcal{R}_{q_{\ell}}^k$中的一个参数固定为$k$的向量。

​		方案由$(KeyGen, Enc, Dec, Add, Mult)$五种算法组成，在环$\mathcal{R}=\mathbb{Z}[X] /\left(\Phi_M(X)\right)$上描述HE方案：

- $\operatorname{KeyGen}\left(1^\lambda\right)$.

  ​		生成一个私钥$sk$，一个用于加密的公钥$pk$，一个辅助计算密钥$evk$用于同态乘法.

- $\mathrm{Enc}_{p k}(m)$.

  ​		对于给定的多项式$m \in \mathcal{R}$，输出一个密码文本$c \in \mathcal{R}_{q_L}^k$。对一些小的$e$，$m$的加密$c$满足:
  $$
  \langle c, s k\rangle=m+e\left(\bmod q_L\right)
  $$

- $\operatorname{Dec}_{s k}(\boldsymbol{c})$.

  ​		对处于级别$\ell$ 的密码文，输出一个密钥$sk$的多项式$m^{\prime} \leftarrow\langle c, s k\rangle\left(\bmod q_{\ell}\right)$.

- $\operatorname{Add}\left(c_1, c_2\right)$.

  ​		对于给定$m_1$和$m_2$的加密，输出$m_1+m_2$的加密。

- $Mult _{e v k}\left(c_1, c_2\right)$.

  ​		对一些密码文本$\left(c_1, c_2\right)$，输出一个密码文本$c_{\text {mult }} \in \mathcal{R}_{q_{\ell}}^k$满足:
  $$
  \left\langle\boldsymbol{c}_{\text {mult }}, s k\right\rangle=\left\langle\boldsymbol{c}_1, s k\right\rangle \cdot\left\langle\boldsymbol{c}_2, s k\right\rangle+e_{\text {mult }}\left(\bmod q_{\ell}\right)
  $$

- $\mathrm{RS}_{\ell \rightarrow \ell^{\prime}}(\mathbf{c})$.

  可以调整现有的环上的HE方案，以构建近似算术的HE方案。例如基于环的BGV档案，通过重缩放程序来代表：

  对密文$c$进行重缩放$\mathbf{c}^{\prime}=\left\lfloor\frac{\mathrm{q}_1^{\prime}}{\mathrm{q}_{\mathrm{l}}} \mathbf{c}\right\rceil$

  对于一个信息$m$的输入密码文本$c$，使得$\langle c, s k\rangle=m+e\left(\bmod q_{\ell}\right)$，重缩放程序的输出密码文本$c^{\prime}$满足$\left\langle c^{\prime}, s k\right\rangle=\frac{q_{\ell^{\prime}}}{q_{\ell}} m+\left(\frac{q_{\ell^{\prime}}}{q_{\ell}} e+e_{\text {scale }}\right)\left(\bmod q_{\ell^{\prime}}\right)$，输出密码文本便是$\frac{q_{\ell^{\prime}}}{q_{\ell}} m$的加密。

​		重缩放程序类似于模数转换算法，将明文除以一个整数，以去除一些不准确的LSB，作为近似计算中的一个舍入步骤。

​		在同态评估过程中，信息的大小几乎可以保持不变，因此所需的最大密码文本模数大小与被评估电路的深度呈线性增长。

**Tagged Informations：**

[<img src="https://s1.ax1x.com/2022/12/07/zgtrfe.png" alt="zgtrfe.png" style="zoom:80%;" />](https://imgse.com/i/zgtrfe)

**Homomorphic Operations of Ciphertexts at Different Levels:**

对于不同级别的密文，通过模数缩减，把较高级别的密文带到较低级别。

### CKKS算法描述

设CKKS方案的安全系数为 $\lambda$ ，明文空间 $\mathcal{M}=\mathbb{C}^{\mathrm{N} / 2}$ ，映射 $\tau$ 为:
$$
\tau: \mathbb{Z}_q[\mathbf{x}] /\left\langle\mathbf{x}^{\mathrm{N}}+1\right\rangle \rightarrow \mathbb{C}^{\mathrm{N} / 2}
$$
即 $\mathrm{m}(\mathrm{x}) \mapsto \mathbf{m}$ ，具体关系为 $\mathrm{z}_{\mathrm{j}}=\mathrm{m}\left(\zeta_{\mathrm{M}}^{\mathrm{j}}\right)$ ，其中 $\zeta_{\mathrm{M}}=\exp (-2 \pi \mathrm{i} / \mathrm{M})$ 。方案详细描述如下:

**1.CKKS.KeyGen$(\left.1^\lambda\right)$**

- 选择一个2的方幂的整数 $\mathrm{M}=\mathrm{M}\left(\lambda, \mathrm{q}_{\mathrm{L}}\right)$ ，一个整数 $\mathrm{h}=\mathrm{h}\left(\lambda, \mathrm{q}_{\mathrm{L}}\right)$ ，一个大整数 $\mathrm{P}=\mathrm{P}\left(\lambda, \mathrm{q}_{\mathrm{L}}\right)$ ，和一个实数 $\sigma=\sigma\left(\lambda, \mathrm{q}_{\mathrm{L}}\right)$ ；
- 采样$s \leftarrow \mathrm{HWT}(\mathrm{h})$ ， $\mathrm{a} \leftarrow \mathrm{R}_{\mathrm{qL}}$ ， $\mathrm{e} \leftarrow \mathrm{DG}\left(\sigma^2\right)$ ，设私钥 $s k=(1, \mathrm{~s})$ ，公钥为 $\mathrm{pk}=(\mathrm{b}, \mathrm{a}) \in \mathrm{R}_{\mathrm{qL}}^2$ ，其中 $b\leftarrow-a s+ e \left(\bmod \mathrm{q}_{\mathrm{L}}\right)$;
- 采样 $\mathrm{a}^{\prime} \leftarrow \mathrm{R}_{\mathrm{qL}}$ ， $\mathrm{e}^{\prime} \leftarrow \mathrm{DG}\left(\sigma^2\right)$ ，设同态乘密钥$evk \leftarrow\left(\mathrm{b}^{\prime}, \mathrm{a}^{\prime}\right) \in \mathrm{R}_{\mathrm{P} \cdot \mathrm{qL}}^2$ ，其中 $\mathrm{b}^{\prime}=-\mathrm{a}^{\prime} \mathrm{s}+\mathrm{e}^{\prime}+\mathrm{Ps}^2\left(\bmod \mathrm{P} \cdot \mathrm{q}_{\mathrm{L}}\right)$

**2.CKKS.Encode $(\mathbf{m}, \Delta)$**

- 对于明文向量 $\mathbf{m}$ ，首先对其等比放大 $\lfloor\Delta \cdot \mathbf{m}\rceil$;
- 接着通过 $\tau$ 映射的逆 $\tau^{-1}$ 将 $[\Delta \cdot \mathbf{m}\rceil$ 转化为多项式

$$
\mathrm{m}=\left\lfloor\tau^{-1}(\lfloor\Delta \cdot \mathbf{m}\rceil)\right\rceil
$$

**3.CKKS.Decode $(\mathrm{m}, \Delta)$**

- 对于明文多项式m，使用 $\tau$ 映射还原为向量 $\mathbf{m}$ 并等比缩小

$$
\mathbf{m}=\left\lfloor\Delta^{-1}(\lfloor\tau(\mathrm{m})\rceil)\right\rceil
$$

**4.CKKS.Enc $_{p k}(\mathrm{~m})$**

- 采样 $\mathrm{v} \leftarrow \mathrm{ZO}(0.5)$, $\mathrm{e}_0, \mathrm{e}_1 \leftarrow \mathrm{DG}\left(\sigma^2\right)$, 输出密文:

$$
\mathbf{c}=\mathrm{v} \cdot \mathrm{pk}+\left(\mathrm{m}+\mathrm{e}_0, \mathrm{e}_1\right)\left(\operatorname{modq}_{\mathrm{L}}\right)
$$

**5.CKKS.Dec $_{\text {sk }}(\mathbf{c})$**

- 对于 $\mathbf{c}=(\mathrm{b}, \mathrm{a})$ ，解密算法为 $[\langle\mathbf{c}, \mathbf{s k}\rangle] \mathbf{q}_1$ ， 即

$$
\mathrm{m}=\mathrm{b}+\mathrm{as}\left(\mathrm{modq}_{\mathrm{l}}\right)
$$

由于CKKS方案在加密时引入了噪声，所以其解密函数生成的明文与原始明文是不同的，但误差的数量级是远远小于明文的，所以该误差是完全可以忽略的。

**6.CKKS.Add$\left(\mathbf{c}_1, \mathbf{c}_2\right)$**

- 对于 $\mathbf{c}_1=\left(\mathrm{b}_1, \mathrm{a}_1\right) ， \mathbf{c}_{\mathbf{2}}=\left(\mathrm{b}_2, \mathrm{a}_2\right)$ ，密文的同态加法为相应位直接相加

$$
\mathbf{c}_{\text {Add }}=\left[\left(\mathrm{b}_1+\mathrm{b}_2, \mathrm{a}_1+\mathrm{a}_2\right)\right] \mathrm{q}_{\mathrm{l}}
$$

由于密文的同态乘法会导致密文规模扩大，这将影响到解密算法的执行，从而使得密钥规模随乘法深度的增加而增大。故Cheon等人在方案设计时引入同态乘密钥$evk$实现对乘法密文的缩放，使其规模保持在$\mathrm{R}_{\mathrm{qL}}^2$中。

**7.CKKS.Mult $_{\text {evk }}\left(\mathbf{c}_1, \mathbf{c}_2\right)$**

- 对于 $\mathbf{c}_{\mathbf{1}}=\left(\mathrm{b}_1, \mathrm{a}_1\right), \mathbf{c}_{\mathbf{2}}=\left(\mathrm{b}_2, \mathrm{a}_2\right)$ ，记
$$
\left(\mathrm{d}_0, \mathrm{~d}_1, \mathrm{~d}_2\right)=\left[\left(\mathrm{b}_1 \mathrm{~b}_2, \mathrm{a}_1 \mathrm{~b}_2+\mathrm{a}_2 \mathrm{~b}_1, \mathrm{a}_1 \mathrm{a}_2\right)\right]_{\mathrm{ql}}
$$
- 则 $\mathbf{c}_1 、 \mathbf{c}_2$ 的同态乘法表示为

$$
\mathbf{c}_{\text {Mult }}=\left(\mathrm{d}_0, \mathrm{~d}_1\right)+\left\lfloor\mathrm{P}^{-1} \cdot \mathrm{d}_2 \cdot \text { evk }\right\rceil
$$

乘法操作会导致噪声增大，出现无法正确解密的情况，因此需要在每次乘法之后进行一个“重缩放”的操作。

**8.CKKS.RS $_{1 \rightarrow 1^{\prime}}(\mathbf{c})$**

- 对于密文 $\mathbf{c}$ ，我们对其进行“重缩放”
$$
\mathbf{c}^{\prime}=\left\lfloor\frac{\mathrm{q}_1^{\prime}}{\mathrm{q}_1} \mathbf{c}\right\rceil
$$
经“重缩放”后，其密文模数降低为 $\mathrm{q}_1^{\prime}$ 。

### Conclusion

提出同态加密方案CKKS，支持近似算数的加密。

引入一种新的批处理技术，将许多信息打包在一个密码文本中，可以通过并行性实现实际的性能优势。

设计重缩放程序，使得近似计算后保留信息的精度，使密码文本的大小大大减少。

CKKS可以成为大整数计算的一个合理方案。











