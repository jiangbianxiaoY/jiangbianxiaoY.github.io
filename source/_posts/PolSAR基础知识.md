---
title: PolSAR基础知识：极化SAR数据表示、矩阵体系与转换关系详解
date: 2026-09-12 10:00:00
updated: 2026-09-15 21:50:00
categories:
  - 深度学习
  - 遥感
tags:
  - PolSAR
  - 极化SAR
  - 散射矩阵
  - 协方差矩阵
  - 相干矩阵
  - Mueller矩阵
---

## 一、极化SAR概述


### 1.1 什么是PolSAR

极化合成孔径雷达（Polarimetric Synthetic Aperture Radar，简称 PolSAR）是 SAR 技术的高级形式。传统单极化 SAR 仅使用一种极化组合（如 HH）发射和接收电磁波，而 PolSAR 能够同时获取多种极化组合的回波信息，从而完整描述地物目标的电磁散射特性。


PolSAR 的核心优势在于：它不仅记录回波的**幅度**信息，还记录**相位**信息，并且能够同时获取四种极化组合（HH、HV、VH、VV）的完整散射响应。这使得 PolSAR 成为遥感领域中信息量最丰富的传感器之一。


### 1.2 极化的物理含义

电磁波是横波，其电场矢量的振动方向定义了极化方向。对于单色平面波，极化描述了电场矢量端点在传播过程中所描绘的轨迹：


- **线极化**：电场矢量始终在一条直线上振动（H 水平极化、V 垂直极化）

- **圆极化**：电场矢量端点描绘圆形轨迹（左旋、右旋）

- **椭圆极化**：一般情况下，电场矢量端点描绘椭圆轨迹


在 PolSAR 系统中，通常采用**正交线极化基**（H/V）作为基准：


- **H（Horizontal）**：水平极化，电场矢量平行于地面

- **V（Vertical）**：垂直极化，电场矢量垂直于地面


### 1.3 PolSAR系统的极化组合

一个完整的 PolSAR 系统可以发射 H 或 V 极化波，并分别接收 H 和 V 极化的回波，因此产生四种极化组合：


| 组合 | 发射 | 接收 | 含义 |
| --- | --- | --- | --- |
| HH | 水平 | 水平 | 同极化 |
| HV | 水平 | 垂直 | 交叉极化 |
| VH | 垂直 | 水平 | 交叉极化 |
| VV | 垂直 | 垂直 | 同极化 |


对于满足**互易定理**（Reciprocity Theorem）的单站雷达系统，有 $S_{HV} = S_{VH}$SHV​=SVH​，因此实际上只需要测量三个独立的复散射系数。


## 二、PolSAR数据的数学表示


### 2.1 Sinclair散射矩阵（S矩阵）

PolSAR数据最基本的表示形式是 **Sinclair散射矩阵**，也称为**极化散射矩阵**。它描述了目标对入射电磁波的完整散射响应。


#### 2.1.1 定义

当入射波的极化状态用Jones矢量 $\mathbf{E}^i$Ei 表示时，散射波的极化状态为：


$$\mathbf{E}^s = \frac{e^{jkr}}{r} \mathbf{S} \mathbf{E}^i$$Es=rejkr​SEi其中 $\mathbf{S}$S 就是 Sinclair 散射矩阵：


$$\mathbf{S} = \begin{bmatrix} S_{HH} & S_{HV} \\ S_{VH} & S_{VV} \end{bmatrix}$$S=[SHH​SVH​​SHV​SVV​​]每个元素 $S_{PQ}$SPQ​ 是一个**复数**，包含幅度和相位信息：


$$S_{PQ} = |S_{PQ}| e^{j\phi_{PQ}}$$SPQ​=∣SPQ​∣ejϕPQ​其中 $|S_{PQ}|$∣SPQ​∣ 表示散射幅度，$\phi_{PQ}$ϕPQ​ 表示散射相位。


#### 2.1.2 物理含义


- **$S_{HH}$SHH​**：发射水平极化、接收水平极化 —— 水平同极化通道

- **$S_{HV}$SHV​**：发射水平极化、接收垂直极化 —— 交叉极化通道

- **$S_{VH}$SVH​**：发射垂直极化、接收水平极化 —— 交叉极化通道

- **$S_{VV}$SVV​**：发射垂直极化、接收垂直极化 —— 垂直同极化通道


#### 2.1.3 互易性

在单站（monostatic）雷达系统中，根据电磁互易定理，满足：


$$S_{HV} = S_{VH}$$SHV​=SVH​因此散射矩阵简化为仅有**三个独立复数参数**（6个实参数）。


#### 2.1.4 向量化表示——目标矢量

将散射矩阵按列（或行）排列成向量形式，称为**目标矢量**（Target Vector）。常用的排列方式有**Lexicographic基**和**Pauli基**两种。


**Lexicographic基排列**（按行展开）：


$$\mathbf{k}_L = \begin{bmatrix} S_{HH} \\ S_{HV} \\ S_{VH} \\ S_{VV} \end{bmatrix} = \begin{bmatrix} S_{HH} \\ S_{HV} \\ S_{HV} \\ S_{VV} \end{bmatrix}$$kL​=​SHH​SHV​SVH​SVV​​​=​SHH​SHV​SHV​SVV​​​利用互易性后简化为：


$$\mathbf{k}_L = \begin{bmatrix} S_{HH} \\ \sqrt{2}S_{HV} \\ S_{VV} \end{bmatrix}$$kL​=​SHH​2​SHV​SVV​​​**Pauli基排列**（按Pauli矩阵展开）：


$$\mathbf{k}_P = \frac{1}{\sqrt{2}} \begin{bmatrix} S_{HH} + S_{VV} \\ S_{HH} - S_{VV} \\ 2S_{HV} \end{bmatrix}$$kP​=2​1​​SHH​+SVV​SHH​−SVV​2SHV​​​Pauli 基的物理含义更加直观：


- 第一个分量 $S_{HH} + S_{VV}$SHH​+SVV​：**奇次散射**（表面散射）

- 第二个分量 $S_{HH} - S_{VV}$SHH​−SVV​：**偶次散射**（二面角散射）

- 第三个分量 $2S_{HV}$2SHV​：**体散射**（交叉极化散射）


### 2.2 相干矩阵（T矩阵）


#### 2.2.1 定义

**相干矩阵**（Coherence Matrix）$\mathbf{T}$T 定义为目标矢量与其共轭转置的外积的**集合平均**（期望值）：


$$\mathbf{T} = \langle \mathbf{k}_P \mathbf{k}_P^H \rangle$$T=⟨kP​kPH​⟩其中 $\langle \cdot \rangle$⟨⋅⟩ 表示空间平均（多视处理），$(\cdot)^H$(⋅)H 表示共轭转置。


展开后：


$$\mathbf{T} = \frac{1}{2} \begin{bmatrix} \langle |S_{HH}+S_{VV}|^2 \rangle & \langle (S_{HH}+S_{VV})(S_{HH}-S_{VV})^* \rangle & \langle (S_{HH}+S_{VV})2S_{HV}^* \rangle \\ \langle (S_{HH}-S_{VV})(S_{HH}+S_{VV})^* \rangle & \langle |S_{HH}-S_{VV}|^2 \rangle & \langle (S_{HH}-S_{VV})2S_{HV}^* \rangle \\ \langle 2S_{HV}(S_{HH}+S_{VV})^* \rangle & \langle 2S_{HV}(S_{HH}-S_{VV})^* \rangle & \langle |2S_{HV}|^2 \rangle \end{bmatrix}$$T=21​​⟨∣SHH​+SVV​∣2⟩⟨(SHH​−SVV​)(SHH​+SVV​)∗⟩⟨2SHV​(SHH​+SVV​)∗⟩​⟨(SHH​+SVV​)(SHH​−SVV​)∗⟩⟨∣SHH​−SVV​∣2⟩⟨2SHV​(SHH​−SVV​)∗⟩​⟨(SHH​+SVV​)2SHV∗​⟩⟨(SHH​−SVV​)2SHV∗​⟩⟨∣2SHV​∣2⟩​​

#### 2.2.2 性质


- **Hermitian矩阵**：$\mathbf{T} = \mathbf{T}^H$T=TH，因此对角线元素为实数，非对角线元素满足 $T_{ij} = T_{ji}^*$Tij​=Tji∗​

- **半正定**：$\mathbf{T} \succeq 0$T⪰0，即所有特征值非负

- **对角线元素为实数**：$T_{11}, T_{22}, T_{33} \geq 0$T11​,T22​,T33​≥0

- **独立参数个数**：3×3 Hermitian矩阵有 **9个独立实参数**（3个对角线实数 + 3对非对角线复数各2个实部虚部）


#### 2.2.3 物理含义

对角线元素具有明确的物理含义：


- $T_{11} = \frac{1}{2}\langle|S_{HH}+S_{VV}|^2\rangle$T11​=21​⟨∣SHH​+SVV​∣2⟩：**奇次散射**能量（表面散射）

- $T_{22} = \frac{1}{2}\langle|S_{HH}-S_{VV}|^2\rangle$T22​=21​⟨∣SHH​−SVV​∣2⟩：**偶次散射**能量（二面角散射）

- $T_{33} = \frac{1}{2}\langle|2S_{HV}|^2\rangle = 2\langle|S_{HV}|^2\rangle$T33​=21​⟨∣2SHV​∣2⟩=2⟨∣SHV​∣2⟩：**体散射**能量（交叉极化散射）


非对角线元素反映不同散射机制之间的**相干性**（相位关系）。


### 2.3 协方差矩阵（C矩阵）


#### 2.3.1 定义

**协方差矩阵**（Covariance Matrix）$\mathbf{C}$C 定义为 Lexicographic 基目标矢量与其共轭转置的外积的集合平均：


$$\mathbf{C} = \langle \mathbf{k}_L \mathbf{k}_L^H \rangle$$C=⟨kL​kLH​⟩展开后（利用互易性 $S_{HV}=S_{VH}$SHV​=SVH​）：


$$\mathbf{C} = \begin{bmatrix} \langle |S_{HH}|^2 \rangle & \langle S_{HH}S_{HV}^* \rangle & \langle S_{HH}S_{VV}^* \rangle \\ \langle S_{HV}S_{HH}^* \rangle & \langle |S_{HV}|^2 \rangle & \langle S_{HV}S_{VV}^* \rangle \\ \langle S_{VV}S_{HH}^* \rangle & \langle S_{VV}S_{HV}^* \rangle & \langle |S_{VV}|^2 \rangle \end{bmatrix}$$C=​⟨∣SHH​∣2⟩⟨SHV​SHH∗​⟩⟨SVV​SHH∗​⟩​⟨SHH​SHV∗​⟩⟨∣SHV​∣2⟩⟨SVV​SHV∗​⟩​⟨SHH​SVV∗​⟩⟨SHV​SVV∗​⟩⟨∣SVV​∣2⟩​​

#### 2.3.2 性质


- **Hermitian矩阵**：$\mathbf{C} = \mathbf{C}^H$C=CH

- **半正定**

- **独立参数个数**：9个实参数

- **对角线元素为实数**：$\langle|S_{HH}|^2\rangle, \langle|S_{HV}|^2\rangle, \langle|S_{VV}|^2\rangle$⟨∣SHH​∣2⟩,⟨∣SHV​∣2⟩,⟨∣SVV​∣2⟩


#### 2.3.3 物理含义


- $C_{11} = \langle|S_{HH}|^2\rangle$C11​=⟨∣SHH​∣2⟩：HH通道平均功率

- $C_{22} = \langle|S_{HV}|^2\rangle$C22​=⟨∣SHV​∣2⟩：HV通道平均功率（交叉极化功率）

- $C_{33} = \langle|S_{VV}|^2\rangle$C33​=⟨∣SVV​∣2⟩：VV通道平均功率

- 非对角线元素反映不同极化通道之间的**相关性**


### 2.4 Mueller矩阵（M矩阵）/ Stokes矩阵


#### 2.4.1 定义

**Mueller矩阵**（也称为**Stokes矩阵**）描述了目标对完全极化或部分极化电磁波的散射作用，它将入射波的 Stokes 矢量映射到散射波的 Stokes 矢量：


$$\mathbf{J}^s = \mathbf{M} \mathbf{J}^i$$Js=MJi其中 Stokes 矢量定义为：


$$\mathbf{J} = \begin{bmatrix} g_0 \\ g_1 \\ g_2 \\ g_3 \end{bmatrix} = \begin{bmatrix} \langle|E_H|^2 + |E_V|^2\rangle \\ \langle|E_H|^2 - |E_V|^2\rangle \\ 2\text{Re}\langle E_H E_V^*\rangle \\ 2\text{Im}\langle E_H E_V^*\rangle \end{bmatrix}$$J=​g0​g1​g2​g3​​​=​⟨∣EH​∣2+∣EV​∣2⟩⟨∣EH​∣2−∣EV​∣2⟩2Re⟨EH​EV∗​⟩2Im⟨EH​EV∗​⟩​​

#### 2.4.2 Mueller矩阵与散射矩阵的关系

对于完全相干的单散射目标，Mueller矩阵可以通过散射矩阵元素计算：


$$\mathbf{M} = \mathbf{A} (\mathbf{S} \otimes \mathbf{S}^*) \mathbf{A}^{-1}$$M=A(S⊗S∗)A−1更直接地，Mueller矩阵的16个元素可以用散射矩阵元素表示为：


$$\mathbf{M} = \begin{bmatrix} M_{00} & M_{01} & M_{02} & M_{03} \\ M_{10} & M_{11} & M_{12} & M_{13} \\ M_{20} & M_{21} & M_{22} & M_{23} \\ M_{30} & M_{31} & M_{32} & M_{33} \end{bmatrix}$$M=​M00​M10​M20​M30​​M01​M11​M21​M31​​M02​M12​M22​M32​​M03​M13​M23​M33​​​其中各元素为散射矩阵元素的二次组合（具体公式较长，此处从略）。


#### 2.4.3 性质


- **实数矩阵**：所有元素都是实数（这是相对于T矩阵和C矩阵的重要区别）

- **不是Hermitian矩阵**

- **独立参数个数**：最多16个实参数（但实际受物理约束少于16个独立参数）

- **对完全相干目标**：Mueller矩阵的秩为1（退化为纯态）

- **对部分极化散射**：Mueller矩阵描述了极化状态的完整变化


#### 2.4.4 特殊情况


- 对于**完全相干目标**（单个散射体），Mueller矩阵可由散射矩阵唯一确定

- 对于**分布式目标**（空间平均），Mueller矩阵描述了散射的统计特性


## 三、各矩阵之间的转换关系


### 3.1 T矩阵与C矩阵的转换

T矩阵和C矩阵之间存在直接的**酉变换**关系，这是最重要的转换关系之一。


#### 3.1.1 从C矩阵到T矩阵

$$\mathbf{T} = \mathbf{U} \mathbf{C} \mathbf{U}^H$$T=UCUH其中转换矩阵 $\mathbf{U}$U 为：


$$\mathbf{U} = \frac{1}{\sqrt{2}} \begin{bmatrix} 1 & 0 & 1 \\ 1 & 0 & -1 \\ 0 & \sqrt{2} & 0 \end{bmatrix}$$U=2​1​​110​002​​1−10​​展开后各元素的关系为：


$$\begin{aligned}
T_{11} &= \frac{1}{2}(C_{11} + C_{33} + C_{13} + C_{31}) \\
T_{12} &= \frac{1}{2}(C_{11} - C_{33} + C_{13} - C_{31}) \\
T_{13} &= \frac{1}{\sqrt{2}}(C_{12} + C_{32}) \\
T_{22} &= \frac{1}{2}(C_{11} + C_{33} - C_{13} - C_{31}) \\
T_{23} &= \frac{1}{\sqrt{2}}(C_{12} - C_{32}) \\
T_{33} &= 2C_{22}
\end{aligned}$$T11​T12​T13​T22​T23​T33​​=21​(C11​+C33​+C13​+C31​)=21​(C11​−C33​+C13​−C31​)=2​1​(C12​+C32​)=21​(C11​+C33​−C13​−C31​)=2​1​(C12​−C32​)=2C22​​

#### 3.1.2 从T矩阵到C矩阵

$$\mathbf{C} = \mathbf{U}^H \mathbf{T} \mathbf{U}$$C=UHTU由于 $\mathbf{U}$U 是酉矩阵，$\mathbf{U}^H = \mathbf{U}^{-1}$UH=U−1，因此反向转换矩阵为：


$$\mathbf{U}^H = \frac{1}{\sqrt{2}} \begin{bmatrix} 1 & 1 & 0 \\ 0 & 0 & \sqrt{2} \\ 1 & -1 & 0 \end{bmatrix}$$UH=2​1​​101​10−1​02​0​​

#### 3.1.3 转换的物理意义


- T矩阵基于**Pauli基**，物理含义清晰（奇次/偶次/体散射）

- C矩阵基于**Lexicographic基**，直接对应各极化通道功率和相关性

- 两者包含**完全相同的信息**，只是表示方式不同

- 转换是**可逆的**、**保信息的**


### 3.2 散射矩阵与协方差/相干矩阵的关系


#### 3.2.1 单视情况

对于单视（single-look）数据，每个像素只有一个散射矩阵，不存在平均。此时：


$$\mathbf{C}_{single} = \mathbf{k}_L \mathbf{k}_L^H$$Csingle​=kL​kLH​$$\mathbf{T}_{single} = \mathbf{k}_P \mathbf{k}_P^H$$Tsingle​=kP​kPH​它们都是**秩1矩阵**（rank-1），因为没有进行空间平均。


#### 3.2.2 多视情况

实际应用中，为了抑制 Speckle 噪声，通常进行多视处理（spatial averaging）：


$$\mathbf{C} = \frac{1}{N} \sum_{i=1}^{N} \mathbf{k}_{L,i} \mathbf{k}_{L,i}^H$$C=N1​i=1∑N​kL,i​kL,iH​$$\mathbf{T} = \frac{1}{N} \sum_{i=1}^{N} \mathbf{k}_{P,i} \mathbf{k}_{P,i}^H$$T=N1​i=1∑N​kP,i​kP,iH​其中 $N$N 是视数（number of looks）。


多视后矩阵的秩可以大于1，最大秩为 $\min(N, 3)$min(N,3)。


### 3.3 Mueller矩阵与T/C矩阵的关系


#### 3.3.1 完全相干目标

对于完全相干目标，Mueller矩阵和散射矩阵之间有确定的关系。通过Kennaugh矩阵（实数表示的散射矩阵）可以建立联系：


$$\mathbf{M} = \mathbf{R} \cdot \text{vec}(\mathbf{S}\mathbf{S}^H)$$M=R⋅vec(SSH)更具体地，使用Kronecker积表示：


$$\mathbf{M} = \mathbf{A}(\mathbf{S} \otimes \mathbf{S}^*) \mathbf{A}^{-1}$$M=A(S⊗S∗)A−1其中 $\mathbf{A}$A 是一个固定的4×4实矩阵。


#### 3.3.2 部分极化/分布式目标

对于分布式目标（需要空间平均），Mueller矩阵和T矩阵之间的关系为：


$$\mathbf{M} = \mathbf{R} \mathbf{T}_L \mathbf{R}^{-1}$$M=RTL​R−1其中 $\mathbf{T}_L$TL​ 是将T矩阵按行展开后重新排列的实向量形式，$\mathbf{R}$R 是一个4×4的实矩阵。


**重要结论**：T矩阵和Mueller矩阵包含的信息量不同：


- T矩阵（或C矩阵）有 **9个独立实参数**

- Mueller矩阵最多有 **16个实参数**，但对完全相干目标只有9个独立参数


### 3.4 转换关系总结图


```
Pauli基变换 (U)
   ┌────────────────────────────────┐
   │                                │
   ▼                                │
Sinclair矩阵 S ──vec──► 目标矢量 k_L (Lexicographic基)
   │                         │
   │ k_L k_L^H               │ k_P k_P^H
   │                         │
   ▼                         ▼
协方差矩阵 C ◄── U^H·C·U ──► 相干矩阵 T
   │                         │
   │     实数化/重排           │     实数化/重排
   │                         │
   ▼                         ▼
Mueller矩阵 M ◄─────────── Mueller矩阵 M
   │
   │ Stokes空间
   ▼
散射波 Stokes 矢量

```


## 四、各种表示形式的优缺点对比


### 4.1 Sinclair散射矩阵（S矩阵）


| 优点 | 缺点 |
| --- | --- |
| 物理含义最直观 | 仅适用于完全相干目标 |
| 直接描述散射过程 | 无法描述分布式目标的统计特性 |
| 便于理论分析和推导 | 受Speckle噪声影响大 |
| 便于理解极化散射机理 | 不适合直接用于分类/识别 |


### 4.2 协方差矩阵（C矩阵）


| 优点 | 缺点 |
| --- | --- |
| 对角线元素直接对应各通道功率 | 物理含义不如T矩阵直观 |
| 适合极化干涉等应用 | 基于Lexicographic基，物理意义不够清晰 |
| 与散射矩阵转换简单 | 非对角线元素的物理解释较复杂 |
| 已有大量成熟算法 | — |


### 4.3 相干矩阵（T矩阵）


| 优点 | 缺点 |
| --- | --- |
| 物理含义清晰（奇次/偶次/体散射） | 计算需要Pauli基变换 |
| 对角线元素有明确散射机理解释 | — |
| 非对角线元素描述散射机制间相干性 | — |
| 适合极化分解（Cloude-Pottier等） | — |
| 是多数极化分类/分解算法的基础 | — |


### 4.4 Mueller矩阵（M矩阵）


| 优点 | 缺点 |
| --- | --- |
| 所有元素都是实数，便于处理 | 最多16个参数，有冗余 |
| 可描述部分极化散射 | 物理含义不如T矩阵直观 |
| 在Stokes空间有清晰的几何解释 | 不如T/C矩阵在极化分解中常用 |
| 适合描述非相干散射过程 | 从M矩阵恢复极化信息不如T/C矩阵方便 |


## 五、极化SAR数据的基本参数


### 5.1 极化功率参数

基于散射矩阵的各通道幅度，可以定义以下功率参数：


| 参数 | 定义 | 含义 |
| --- | --- | --- |
| $\sigma_{HH}^0$σHH0​ | $\langle | S_{HH} |
| $\sigma_{HV}^0$σHV0​ | $\langle | S_{HV} |
| $\sigma_{VH}^0$σVH0​ | $\langle | S_{VH} |
| $\sigma_{VV}^0$σVV0​ | $\langle | S_{VV} |


### 5.2 极化比

**极化比**（Polarization Ratio）是同极化通道之间的比值：


$$\rho = \frac{\sigma_{VV}^0}{\sigma_{HH}^0}$$ρ=σHH0​σVV0​​极化比可以提供地表粗糙度和介电常数信息。


### 5.3 极化相关系数

不同极化通道之间的**复相关系数**（也称为**相干系数**）：


$$\rho_{HH,VV} = \frac{\langle S_{HH} S_{VV}^* \rangle}{\sqrt{\langle|S_{HH}|^2\rangle \langle|S_{VV}|^2\rangle}}$$ρHH,VV​=⟨∣SHH​∣2⟩⟨∣SVV​∣2⟩​⟨SHH​SVV∗​⟩​其模值 $|\rho_{HH,VV}|$∣ρHH,VV​∣ 在0到1之间：


- 接近1：表示HH和VV高度相关（如平坦表面散射）

- 接近0：表示HH和VV不相关（如体散射）


### 5.4 极化熵（Polarimetric Entropy）

极化熵是衡量散射随机性的重要参数，由Cloude和Pottier于1997年提出。


对相干矩阵 $\mathbf{T}$T 进行特征值分解：


$$\mathbf{T} = \sum_{i=1}^{3} \lambda_i \mathbf{e}_i \mathbf{e}_i^H$$T=i=1∑3​λi​ei​eiH​其中 $\lambda_1 \geq \lambda_2 \geq \lambda_3 \geq 0$λ1​≥λ2​≥λ3​≥0 为特征值，$\mathbf{e}_i$ei​ 为对应的特征矢量。


定义归一化概率：


$$p_i = \frac{\lambda_i}{\sum_{j=1}^{3} \lambda_j}$$pi​=∑j=13​λj​λi​​则极化熵为：


$$H = -\sum_{i=1}^{3} p_i \log_3(p_i)$$H=−i=1∑3​pi​log3​(pi​)$H$H 的取值范围为 $[0, 1]$[0,1]：


- $H = 0$H=0：完全极化散射（单一散射机制）

- $H = 1$H=1：完全随机散射（三个等权散射机制）


### 5.5 平均散射角（Alpha角）

平均散射角定义了主要散射机制的类型：


$$\bar{\alpha} = \sum_{i=1}^{3} p_i \alpha_i$$αˉ=i=1∑3​pi​αi​其中 $\alpha_i = \arccos(|\mathbf{u}_{i1}|)$αi​=arccos(∣ui1​∣)，$\mathbf{u}_{i1}$ui1​ 是第 $i$i 个特征矢量的第一个分量。


$\bar{\alpha}$αˉ 的物理含义：


- $\bar{\alpha} \approx 0°$αˉ≈0°：表面散射（Bragg散射）

- $\bar{\alpha} \approx 45°$αˉ≈45°：偶次散射（二面角散射）

- $\bar{\alpha} \approx 90°$αˉ≈90°：体散射（随机散射）


## 六、极化分解理论概述

极化分解是 PolSAR 数据分析的核心技术，目的是将复杂的散射过程分解为若干基本散射机制的组合。


### 6.1 相干分解（Coherent Decomposition）

适用于**单视**或**完全相干**数据（秩1矩阵）。


#### 6.1.1 Pauli分解

将散射矩阵分解为Pauli矩阵的线性组合：


$$\mathbf{S} = a(\mathbf{S}_a) + b(\mathbf{S}_b) + c(\mathbf{S}_c) + d(\mathbf{S}_d)$$S=a(Sa​)+b(Sb​)+c(Sc​)+d(Sd​)其中Pauli矩阵为：


$$\begin{aligned}
\mathbf{S}_a &= \begin{bmatrix} 1 & 0 \\ 0 & 1 \end{bmatrix} \quad \text{(奇次散射)} \\
\mathbf{S}_b &= \begin{bmatrix} 1 & 0 \\ 0 & -1 \end{bmatrix} \quad \text{(偶次散射)} \\
\mathbf{S}_c &= \begin{bmatrix} 0 & 1 \\ 1 & 0 \end{bmatrix} \quad \text{(45°二面角)} \\
\mathbf{S}_d &= \begin{bmatrix} 0 & -j \\ j & 0 \end{bmatrix} \quad \text{(螺旋散射)}
\end{aligned}$$Sa​Sb​Sc​Sd​​=[10​01​](奇次散射)=[10​0−1​](偶次散射)=[01​10​](45°二面角)=[0j​−j0​](螺旋散射)​分解系数就是Pauli基目标矢量的分量。


#### 6.1.2 Freeman-Durden分解

将散射矩阵分解为三种基本散射机制的组合：


$$\mathbf{S} = f_s \mathbf{S}_s + f_d \mathbf{S}_d + f_v \mathbf{S}_v$$S=fs​Ss​+fd​Sd​+fv​Sv​其中：


- $f_s$fs​：表面散射功率

- $f_d$fd​：偶次散射功率

- $f_v$fv​：体散射功率


每种散射机制有对应的散射矩阵模型，通过求解功率方程组得到各机制的权重。


### 6.2 非相干分解（Incoherent Decomposition）

适用于**多视**数据（满秩矩阵）。


#### 6.2.1 Cloude-Pottier分解

基于相干矩阵 $\mathbf{T}$T 的特征值分解，将散射过程分解为三个独立的散射机制：


$$\mathbf{T} = \lambda_1 \mathbf{e}_1 \mathbf{e}_1^H + \lambda_2 \mathbf{e}_2 \mathbf{e}_2^H + \lambda_3 \mathbf{e}_3 \mathbf{e}_3^H$$T=λ1​e1​e1H​+λ2​e2​e2H​+λ3​e3​e3H​每个 $\mathbf{e}_i \mathbf{e}_i^H$ei​eiH​ 对应一种散射机制，$\lambda_i$λi​ 为该机制的权重。


主要参数：


- **极化熵 $H$H**：散射随机性

- **各向异性度 $A$A**：$A = \frac{\lambda_2 - \lambda_3}{\lambda_2 + \lambda_3}$A=λ2​+λ3​λ2​−λ3​​，描述第二和第三散射机制的差异

- **平均散射角 $\bar{\alpha}$αˉ**：主要散射机制类型


#### 6.2.2 H/A/α分类

基于 $H$H、$A$A、$\bar{\alpha}$αˉ 三个参数，可以将散射机制分为8种基本类别（H-α平面的8个区域）：


| 区域 | H | α | 散射类型 |
| --- | --- | --- | --- |
| 1 | 低 | 低 | 表面散射 |
| 2 | 低 | 中 | 偶次散射 |
| 3 | 低 | 高 | 体散射（偶极子） |
| 4 | 中 | 低 | 表面散射+体散射混合 |
| 5 | 中 | 中 | 偶次散射+体散射混合 |
| 6 | 中 | 高 | 体散射 |
| 7 | 高 | 低 | 随机散射（表面主导） |
| 8 | 高 | 高 | 随机散射（体散射主导） |


## 七、PolSAR数据处理流程


### 7.1 数据采集与预处理


```
原始回波数据
    │
    ▼
┌─────────────────┐
│   SAR成像处理    │  距离压缩、方位压缩、运动补偿
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  辐射定标        │  将数字值转换为散射系数
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  极化定标        │  串扰校正、幅度/相位均衡
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  散射矩阵 S      │  单视复数数据
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  多视处理        │  空间平均，抑制Speckle
└────────┬────────┘
         │
         ▼
  T矩阵 或 C矩阵

```


### 7.2 常用处理算法


| 步骤 | 常用方法 |
| --- | --- |
| Speckle滤波 | Lee滤波、Refined Lee、IDAN、NL-SAR |
| 极化分解 | Freeman-Durden、Cloude-Pottier、Yamaguchi |
| 地物分类 | Wishart分类、SVM、随机森林、深度学习 |
| 目标检测 | CFAR、极化恒虚警率检测 |
| 参数反演 | 土壤湿度反演、植被高度估计 |


## 八、总结

本文详细介绍了 PolSAR 数据的基础知识，包括：


- **四种极化组合**（HH、HV、VH、VV）及其物理含义

- **Sinclair散射矩阵**：最基本的极化数据表示

- **协方差矩阵**（C矩阵）：基于Lexicographic基，适合极化干涉等应用

- **相干矩阵**（T矩阵）：基于Pauli基，物理含义清晰，是极化分解的基础

- **Mueller矩阵**（M矩阵）：实数矩阵，适合描述部分极化散射

- **矩阵间的转换关系**：T矩阵和C矩阵之间可通过酉变换互转

- **极化分解理论**：相干分解和非相干分解的基本框架


理解这些基本概念是进行 PolSAR 数据处理和应用的基础。在实际应用中，根据具体需求选择合适的数据表示形式和处理方法至关重要。


## 参考文献


- Cloude, S. R., & Pottier, E. (1997). An entropy based classification scheme for land applications of polarimetric SAR. *IEEE Transactions on Geoscience and Remote Sensing*, 35(1), 68-78.

- Freeman, A., & Durden, S. L. (1998). A three-component scattering model for polarimetric SAR data. *IEEE Transactions on Geoscience and Remote Sensing*, 36(3), 963-973.

- Lee, J. S., & Pottier, E. (2009). *Polarimetric Radar Imaging: From Basics to Applications*. CRC Press.

- Boerner, W. M., et al. (1993). *Polarimetry in Radar Remote Sensing: Basic and Applied Concepts*. Springer.
