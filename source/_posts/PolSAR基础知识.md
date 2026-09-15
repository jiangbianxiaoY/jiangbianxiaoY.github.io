---
title: PolSAR基础知识：极化SAR数据表示、矩阵体系与转换关系详解
date: 2026-09-12 10:00:00
updated: 2026-09-12 10:00:00
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
|------|------|------|------|
| HH | 水平 | 水平 | 同极化 |
| HV | 水平 | 垂直 | 交叉极化 |
| VH | 垂直 | 水平 | 交叉极化 |
| VV | 垂直 | 垂直 | 同极化 |

对于满足**互易定理**（Reciprocity Theorem）的单站雷达系统，有 $S_{HV} = S_{VH}$，因此实际上只需要测量三个独立的复散射系数。

## 二、PolSAR数据的数学表示

### 2.1 Sinclair散射矩阵（S矩阵）

PolSAR数据最基本的表示形式是 **Sinclair散射矩阵**，也称为**极化散射矩阵**。它描述了目标对入射电磁波的完整散射响应。

#### 2.1.1 定义

当入射波的极化状态用Jones矢量 $\mathbf{E}^i$ 表示时，散射波的极化状态为：

$$\mathbf{E}^s = \frac{e^{jkr}}{r} \mathbf{S} \mathbf{E}^i$$

其中 $\mathbf{S}$ 就是 Sinclair 散射矩阵：

$$\mathbf{S} = \begin{bmatrix} S_{HH} & S_{HV} \\ S_{VH} & S_{VV} \end{bmatrix}$$

每个元素 $S_{PQ}$ 是一个**复数**，包含幅度和相位信息：

$$S_{PQ} = |S_{PQ}| e^{j\phi_{PQ}}$$

其中 $|S_{PQ}|$ 表示散射幅度，$\phi_{PQ}$ 表示散射相位。

#### 2.1.2 物理含义

- **$S_{HH}$**：发射水平极化、接收水平极化 —— 水平同极化通道
- **$S_{HV}$**：发射水平极化、接收垂直极化 —— 交叉极化通道
- **$S_{VH}$**：发射垂直极化、接收水平极化 —— 交叉极化通道
- **$S_{VV}$**：发射垂直极化、接收垂直极化 —— 垂直同极化通道

#### 2.1.3 互易性

在单站（monostatic）雷达系统中，根据电磁互易定理，满足：

$$S_{HV} = S_{VH}$$

因此散射矩阵简化为仅有**三个独立复数参数**（6个实参数）。

#### 2.1.4 向量化表示——目标矢量

将散射矩阵按列（或行）排列成向量形式，称为**目标矢量**（Target Vector）。常用的排列方式有**Lexicographic基**和**Pauli基**两种。

**Lexicographic基排列**（按行展开）：

$$\mathbf{k}_L = \begin{bmatrix} S_{HH} \\ S_{HV} \\ S_{VH} \\ S_{VV} \end{bmatrix}$$

利用互易性后简化为：

$$\mathbf{k}_L = \begin{bmatrix} S_{HH} \\ \sqrt{2}S_{HV} \\ S_{VV} \end{bmatrix}$$

**Pauli基排列**（按Pauli矩阵展开）：

$$\mathbf{k}_P = \frac{1}{\sqrt{2}} \begin{bmatrix} S_{HH} + S_{VV} \\ S_{HH} - S_{VV} \\ 2S_{HV} \end{bmatrix}$$

Pauli 基的物理含义更加直观：

- 第一个分量 $S_{HH} + S_{VV}$：对称散射（同极化贡献）
- 第二个分量 $S_{HH} - S_{VV}$：反对称散射（交叉极化贡献）
- 第三个分量 $2S_{HV}$：**体散射**（交叉极化散射）

### 2.2 相干矩阵（T矩阵）

#### 2.2.1 定义

**相干矩阵**（Coherence Matrix）$\mathbf{T}$ 定义为目标矢量与其共轭转置的外积的**集合平均**（期望值）：

$$\mathbf{T} = \langle \mathbf{k}_P \mathbf{k}_P^H \rangle$$

展开后：

$$\mathbf{T} = \frac{1}{2} \begin{bmatrix} \langle |S_{HH}+S_{VV}|^2 \rangle & \langle (S_{HH}+S_{VV})(S_{HH}-S_{VV})^* \rangle & \langle (S_{HH}+S_{VV})2S_{HV}^* \rangle \\ \langle (S_{HH}-S_{VV})(S_{HH}+S_{VV})^* \rangle & \langle |S_{HH}-S_{VV}|^2 \rangle & \langle (S_{HH}-S_{VV})2S_{HV}^* \rangle \\ \langle 2S_{HV}(S_{HH}+S_{VV})^* \rangle & \langle 2S_{HV}(S_{HH}-S_{VV})^* \rangle & \langle |2S_{HV}|^2 \rangle \end{bmatrix}$$

#### 2.2.2 性质

1. **Hermitian矩阵**：$\mathbf{T} = \mathbf{T}^H$，因此对角线元素为实数，非对角线元素满足 $T_{ij} = T_{ji}^*$
2. **半正定**：$\mathbf{T} \succeq 0$，即所有特征值非负
3. **独立参数个数**：3×3 Hermitian矩阵有 **9个独立实参数**（3个对角线实数 + 3对非对角线复数各2个实部虚部）

#### 2.2.3 物理含义

对角线元素具有明确的物理含义：

$$T_{11} = \frac{1}{2}\langle|S_{HH}+S_{VV}|^2\rangle$$

$$T_{22} = \frac{1}{2}\langle|S_{HH}-S_{VV}|^2\rangle$$

$$T_{33} = \frac{1}{2}\langle|2S_{HV}|^2\rangle = 2\langle|S_{HV}|^2\rangle$$

非对角线元素反映不同散射机制之间的**相干性**（相位关系）。

### 2.3 协方差矩阵（C矩阵）

#### 2.3.1 定义

**协方差矩阵**（Covariance Matrix）$\mathbf{C}$ 定义为 Lexicographic 基目标矢量与其共轭转置的外积的集合平均：

$$\mathbf{C} = \langle \mathbf{k}_L \mathbf{k}_L^H \rangle$$

展开后（利用互易性）：

$$\mathbf{C} = \begin{bmatrix} \langle |S_{HH}|^2 \rangle & \langle S_{HH}S_{HV}^* \rangle & \langle S_{HH}S_{VV}^* \rangle \\ \langle S_{HV}S_{HH}^* \rangle & \langle |S_{HV}|^2 \rangle & \langle S_{HV}S_{VV}^* \rangle \\ \langle S_{VV}S_{HH}^* \rangle & \langle S_{VV}S_{HV}^* \rangle & \langle |S_{VV}|^2 \rangle \end{bmatrix}$$

#### 2.3.2 性质

1. **Hermitian矩阵**：$\mathbf{C} = \mathbf{C}^H$
2. **半正定**
3. **独立参数个数**：9个实参数
4. **对角线元素为实数**

#### 2.3.3 物理含义

- $C_{11} = \langle|S_{HH}|^2\rangle$：同极化散射功率
- $C_{22} = \langle|S_{HV}|^2\rangle$：交叉极化散射功率
- $C_{33} = \langle|S_{VV}|^2\rangle$：反向同极化散射功率

### 2.4 Mueller矩阵（M矩阵）

#### 2.4.1 定义

**Mueller矩阵**是描述极化散射的一种实数矩阵形式，它将散射矩阵从复数域映射到实数域。

$$\mathbf{J}^s = \mathbf{M} \mathbf{J}^i$$

其中 $\mathbf{J}$ 为 **Stokes矢量**：

$$\mathbf{J} = \begin{bmatrix} g_0 \\ g_1 \\ g_2 \\ g_3 \end{bmatrix} = \begin{bmatrix} \langle|E_H|^2 + |E_V|^2\rangle \\ \langle|E_H|^2 - |E_V|^2\rangle \\ 2\text{Re}\langle E_H E_V^*\rangle \\ 2\text{Im}\langle E_H E_V^*\rangle \end{bmatrix}$$

Mueller矩阵与散射矩阵的关系：

$$\mathbf{M} = \mathbf{A} (\mathbf{S} \otimes \mathbf{S}^*) \mathbf{A}^{-1}$$

#### 2.4.2 Mueller矩阵的形式

$$\mathbf{M} = \begin{bmatrix} M_{00} & M_{01} & M_{02} & M_{03} \\ M_{10} & M_{11} & M_{12} & M_{13} \\ M_{20} & M_{21} & M_{22} & M_{23} \\ M_{30} & M_{31} & M_{32} & M_{33} \end{bmatrix}$$

#### 2.4.3 与协方差矩阵的关系

$$\mathbf{T} = \mathbf{U} \mathbf{C} \mathbf{U}^H$$

$$\mathbf{U} = \frac{1}{\sqrt{2}} \begin{bmatrix} 1 & 0 & 1 \\ 1 & 0 & -1 \\ 0 & \sqrt{2} & 0 \end{bmatrix}$$

$$\mathbf{C} = \mathbf{U}^H \mathbf{T} \mathbf{U}$$

## 三、极化目标分解

### 3.1 目标分解的基本概念

极化目标分解是将观测到的极化散射信息分解为若干基本散射机制的贡献。常见的散射机制包括：

- **表面散射**（Surface scattering）：由平坦地面或水面产生
- **双站散射**（Double-bounce scattering）：由墙-地或树-地结构产生
- **体散射**（Volume scattering）：由植被等体积目标产生

### 3.2 相干矩阵分解

基于相干矩阵的特征值分解：

$$\mathbf{T} = \lambda_1 \mathbf{e}_1 \mathbf{e}_1^H + \lambda_2 \mathbf{e}_2 \mathbf{e}_2^H + \lambda_3 \mathbf{e}_3 \mathbf{e}_3^H$$

特征值 $\lambda_i$ 表示各散射机制的功率占比：

$$p_i = \frac{\lambda_i}{\sum_{j=1}^{3} \lambda_j}$$

熵（Entropy）：

$$H = -\sum_{i=1}^{3} p_i \log_3(p_i)$$

### 3.3 散射功率分解

基于散射功率的分解方法：

$$\mathbf{M} = \mathbf{R} \cdot \text{vec}(\mathbf{S}\mathbf{S}^H)$$

等价地：

$$\mathbf{M} = \mathbf{A}(\mathbf{S} \otimes \mathbf{S}^*) \mathbf{A}^{-1}$$

## 四、总结

极化SAR数据具有丰富的电磁散射信息，通过 Sinclair散射矩阵、相干矩阵、协方差矩阵和Mueller矩阵等多种数学表示方式，可以全面描述地物目标的散射特性。极化目标分解方法（如H/α分解、Freeman分解等）能够将复杂的极化信息分解为基本散射机制，为地物分类、目标检测和参数反演提供强有力的工具。随着深度学习技术的发展，基于极化SAR的智能处理方法正在成为新的研究热点。
