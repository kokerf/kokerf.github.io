---
title: On-Manifold Preintegration
date: 2020-02-04 15:50:47
categories: "VIO"
tags: ["VIO", "IMU", "SLAM"]
mathjax: true
top: true
---

这是一篇流形上预积分的笔记。预积分用多次的IMU测量来构造这段时间运动的相对变化量，在优化的框架中，这个构造的量可以代替一帧一帧的IMU测量加入到pose的求解。IMU中的加速度计对时间进行两次积分可以得到位移，而陀螺仪一次积分可以得到角度。在已知$T$时刻的PVR和到$T+1$时刻之间的IMU测量，可以通过积分得到$T+1$时刻的PVR。预积分的方法就是用来表示这两个时刻的相对PVR，且使得这个相对量与$T$时刻的PVR无关。其中旋转有多种表示方式，包括欧拉角、旋转向量、四元数和$SO(3)$。这篇笔记主要是针对16年的`On-Manifold Preintegration for Real-Time Visual-Inertial Odometry`论文的推导，在旋转表示上采用了$SO(3)$。

<!--more-->

### **1. 数学**
#### **1.1 李群李代数**
在预积分中，使用了李群$SO(3)$来表示旋转。定义为$SO(3)\doteq \{\mathbf R \in \mathbb R^{3\times3}: \mathbf R^T\mathbf R = \mathbf I, \text{det}(\mathbf R) = 1\}$，其切空间（tangent space）表示为${\frak so}(3)$，为李代数。${\frak so}(3)$可以用$3\times3$的反对称矩阵表示，在$\mathbb R^3$上的一个向量使用`hat`操作得到反对称矩阵：

$$
\begin{equation}
\mathbf\omega^{\wedge} = \begin{bmatrix}\omega_1\\\omega_2\\\omega_3\end{bmatrix}
= \begin{bmatrix}0 & -\omega_3 & \omega_2\\ \omega_3 & 0 & -\omega_1\\ -\omega_2 & \omega_1 & 0\end{bmatrix} \in {\frak so}(3)
\end{equation}
$$

这里给出两个比较常用的公式：
$$
\begin{eqnarray}
\text{Log}(\text{Exp}(\mathbf\phi)\text{Exp}(\delta\mathbf\phi)) \approx \mathbf\phi + \mathbf J_r^{-1}(\mathbf\phi)\delta\mathbf\phi\label{so3_log_jr}\\
\text{Log}(\text{Exp}(\delta\mathbf\phi)\text{Exp}(\mathbf\phi)) \approx \mathbf\phi + \mathbf J_l^{-1}(\mathbf\phi)\delta\mathbf\phi\label{so3_log_jl}
\end{eqnarray}
$$

$$
\begin{eqnarray}
\text{Exp}(\mathbf\phi + \delta\mathbf\phi) \approx \text{Exp}(\mathbf\phi)\text{Exp}(\mathbf J_r(\mathbf\phi) \delta\mathbf\phi)\label{so3_exp_jr}\\
\text{Exp}(\mathbf\phi + \delta\mathbf\phi) \approx \text{Exp}(\mathbf J_l(\mathbf\phi) \delta\mathbf\phi)\text{Exp}(\mathbf\phi)\label{so3_exp_jl}
\end{eqnarray}
$$

左乘雅可比和右乘雅可比的关系为：
$$
\begin{eqnarray}
\mathbf J_l(-\mathbf\phi) = \mathbf J_r(\mathbf\phi)\label{so3_jac_lr1}\\
\mathbf J_l(-\mathbf\phi) = \text{Exp}(\mathbf\phi)\mathbf J_r(\mathbf\phi)\label{so3_jac_lr2}
\end{eqnarray}
$$

$\text{SO}(3)$指数映射的性质：
$$
\begin{eqnarray}
\mathbf R\text{Exp}(\mathbf\phi)\mathbf R^T = \text{exp}(\mathbf R\mathbf\phi^\land\mathbf R^T) = \text{Exp}(\mathbf R\mathbf\phi) \label{so3_exp_prop1}\\
\iff \text{Exp}(\mathbf\phi)\mathbf R = \mathbf R \text{Exp}(\mathbf R^T\mathbf\phi)\label{so3_exp_prop2}
\end{eqnarray}
$$

并且有：
$$
\begin{equation}
(\mathbf R\mathbf\phi)^\land = \mathbf R\mathbf\phi^\land\mathbf R^T\label{so3_hat}
\end{equation}
$$


### **2. IMU模型与运动积分**

#### **2.1 IMU模型与运动积分**

### **3. 流形上的预积分**
在预积分过程中，为了防止不必要的重新积分，我们希望通过IMU计算得到两个关键帧间运动关系的相对的变化量，与当前关键帧在世界坐标下的运动状态无关。另外，这里的相对量是在IMU刚刚获取的时候就计算的，而IMU的bias会随着时间变化。当然bias可以作为一个变量加入到之后的优化中去求解，但是考虑到把预积分的值是与IMU的bias有关的，当bias变化，预积分的值必然会变化。而重新计算预积分的值就违背了我们使用预积分的初衷，因此我们希望新的预积分值不用重新计算，最好只通过一个简单的更新步骤就可以得到。因此预积分需要找到一种表达形式满足以下两个条件：

> - 可以表示两个关键帧间运动的相对变化量
> - 可以在bias变化后，预积分的值可以通过简单的方式更新（而不用重新计算）

考虑到在优化中，我们把预积分的值作为**IMU的观测**以“边”的形式加入到优化框架中，而边的权重（**协方差矩阵**）则通过分析IMU的噪声模型来获得，另外其中的bias是我们**需要估计的自变量**。之后，将分别从预积分的表示形式、bias的更新方式、误差的传播进行讨论。

#### **3.1 流形上的预积分**
在两个关键帧之间对IMU的数据进行积分，关键帧的时刻分别为$k=i$和$k=j$，则给出如下：
$$
\begin{eqnarray}
\mathbf R_j &=& \mathbf R_i \prod_{k=i}^{j-1}\text{Exp}\big((\tilde{\mathbf\omega}_k-\mathbf b_k^g-\mathbf\eta_k^{gd})\Delta t\big)\label{model_r}\\
\mathbf v_j &=& \mathbf v_i + \mathbf g\Delta t_{ij} + \sum_{k=i}^{j-1}\mathbf R_k(\tilde{\mathbf a}_k-\mathbf b_k^a-\mathbf\eta_k^{ad})\Delta t\label{model_v}\\
\mathbf p_j &=& \mathbf p_i + \sum_{k=i}^{j-1}\big[\mathbf v_k\Delta t + \frac{1}{2}\mathbf g\Delta t^2 + \frac{1}{2}\mathbf R_k(\tilde{\mathbf a}_k-\mathbf b_k^a-\mathbf\eta_k^{ad})\Delta t^2\big]\label{model_p}
\end{eqnarray}
$$

进一步，定义为相对的增量：
$$
\begin{eqnarray}
\Delta\mathbf R_{ij} &\doteq& \mathbf R_i^T\mathbf R_j = \prod_{k=i}^{j-1}\text{Exp}\big((\tilde{\mathbf\omega}_k-\mathbf b_k^g-\mathbf\eta_k^{gd})\Delta t\big)\label{model_dr}\\
\Delta\mathbf v_{ij} &\doteq& \mathbf R_i^T(\mathbf v_j - \mathbf v_i - \mathbf g\Delta t_{ij}) = \sum_{k=i}^{j-1}\Delta\mathbf R_{ik}(\tilde{\mathbf a}_k-\mathbf b_k^a-\mathbf\eta_k^{ad})\Delta t\label{model_dv}\\
\Delta\mathbf p_{ij} &\doteq& \mathbf R_i^T(\mathbf p_j - \mathbf p_i - \mathbf v_i \Delta t_{ij} - \frac{1}{2}\mathbf g\Delta t_{ij}^2 ) = \sum_{k=i}^{j-1}\big[\Delta \mathbf v_{ik}\Delta t + \frac{1}{2}\Delta \mathbf R_{ik}(\tilde{\mathbf a}_k-\mathbf b_k^a-\mathbf\eta_k^{ad})\Delta t^2\big]\label{model_dp}
\end{eqnarray}
$$

这里$\Delta \mathbf R_{ik} \doteq \mathbf R_i^T\mathbf R_j$，$\Delta\mathbf v_{ik}\doteq \mathbf R_i^T(\mathbf v_k - \mathbf c_i - \mathbf g \Delta t_{ik})$。

忽略噪声这里写为增量的形式为:
$$
\begin{eqnarray}
\color{green}{\Delta\mathbf R_{ij}} &=& \color{teal}{\Delta\mathbf R_{ij-1}} \text{Exp}\big((\tilde{\mathbf\omega}_{j-1}-\mathbf b_{j-1}^g)\Delta t\big)\\
\color{green}{\Delta\mathbf v_{ij}} &=& \color{teal}{\Delta\mathbf v_{ij-1}} + \color{teal}{\Delta\mathbf R_{ij-1}}(\tilde{\mathbf a}_{j-1}-\mathbf b_{j-1}^a)\Delta t\\
\color{green}{\Delta\mathbf p_{ij}} &=& \color{teal}{\Delta\mathbf p_{ij-1}} + \color{teal}{\Delta\mathbf v_{ij-1}}\Delta t + \frac{1}{2}\color{teal}{\Delta\mathbf R_{ij-1}}(\tilde{\mathbf a}_{j-1}-\mathbf b_{j-1}^a)\Delta t^2
\end{eqnarray}
$$
在实际编程实现时，如果在更新过程中预积分的变量都是同一个，那么，尽可能把被其他预积分增量用到的变量放在最后进行更新。因此，最好的更新顺序是先$\Delta\mathbf p$、再$\Delta\mathbf v$，最后$\Delta\mathbf R$。

#### **3.2 IMU预积分对Bias的一阶泰勒近似**

当bias变化之后，如何更新预积分呢？`On-manifold`算法中使用对bias进行一阶泰勒展开的方法。这里给出的预积分对Bias的更新方程：

$$
\begin{eqnarray}
\Delta\tilde{\mathbf R}_{ij}(\mathbf b_i^g) &\simeq& \Delta \tilde{\mathbf R}_{ij}(\bar{\mathbf b}_i^g)\text{Exp}\bigg(\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\delta\mathbf b_i^g\bigg)\label{bias_correct_r}\\
\Delta\tilde{\mathbf v}_{ij}(\mathbf b_i^g, \mathbf b_i^g) &\simeq& \Delta\tilde{\mathbf v}_{ij}(\bar{\mathbf b}_i^g, \bar{\mathbf b}_i^g) + \frac{\partial\Delta\bar{\mathbf v}_{ij}}{\partial\mathbf b_i^g} \delta\mathbf b_i^g + \frac{\partial\Delta\bar{\mathbf v}_{ij}}{\partial\mathbf b_i^a} \delta\mathbf b_i^a\label{bias_correct_v}\\
\Delta\tilde{\mathbf p}_{ij}(\mathbf b_i^g, \mathbf b_i^g) &\simeq& \Delta\tilde{\mathbf p}_{ij}(\bar{\mathbf b}_i^g, \bar{\mathbf b}_i^g) + \frac{\partial\Delta\bar{\mathbf p}_{ij}}{\partial\mathbf b_i^g} \delta\mathbf b_i^g + \frac{\partial\Delta\bar{\mathbf p}_{ij}}{\partial\mathbf b_i^a} \delta\mathbf b_i^a\label{bias_correct_p}
\end{eqnarray}
$$

最终给出对应的雅可比：
$$
\begin{eqnarray}
\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g} &=& - \sum_{k=i}^{j-1} \Delta\tilde{\mathbf R}_{k+1j}(\bar{\mathbf b}_i^g)^T\mathbf J_r^k \Delta t\\
\frac{\partial\Delta\bar{\mathbf v}_{ij}}{\partial\mathbf b_i^a} &=& - \sum_{k=i}^{j-1}\Delta\tilde{\mathbf R}_{ik}\Delta t\\
\frac{\partial\Delta\bar{\mathbf v}_{ij}}{\partial\mathbf b_i^g} &=& - \sum_{k=i}^{j-1} \Delta\tilde{\mathbf R}_{ik}(\tilde{\mathbf a}_k-\bar{\mathbf b}_i^a)^{\wedge} \frac{\partial\Delta\bar{\mathbf R}_{ik}}{\partial\mathbf b_i^g} \Delta t\\
\frac{\partial\Delta\bar{\mathbf p}_{ij}}{\partial\mathbf b_i^a} &=& \sum_{k=i}^{j-1}\big[\frac{\partial\Delta\bar{\mathbf v}_{ik}}{\partial\mathbf b_i^a}\Delta t - \frac{1}{2}\Delta\bar{\mathbf R}_{ik}\Delta t^2\big]\\
\frac{\partial\Delta\bar{\mathbf p}_{ij}}{\partial\mathbf b_i^g} &=& \sum_{k=i}^{j-1}\big[\frac{\partial\Delta\bar{\mathbf v}_{ik}}{\partial\mathbf b_i^g}\Delta t - \frac{1}{2}\Delta\tilde{\mathbf R}_{ik}(\tilde{\mathbf a}_k-\bar{\mathbf b}_i^a)^{\wedge} \frac{\partial\Delta\bar{\mathbf R}_{ik}}{\partial\mathbf b_i^g} \Delta t^2\big]
\end{eqnarray}
$$

这里注意到，由于第一个式子中的$\Delta\tilde{\mathbf R}_{k+1j}$ 是从$k+1$ 时刻到$j$ 时刻，因此只有当所有数据都有之后，才可以计算，也就是说没有办法通过迭代的形式求解。为了能够在形式上统一，也就是在每一帧IMU数据到来的时候，都可以进行更新，因此做如下处理：
$$
\begin{equation}
\begin{split}
\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g} &= - \sum_{k=i}^{j-1} \Delta\tilde{\mathbf R}_{k+1j}(\bar{\mathbf b}_i^g)^T\mathbf J_r^k \Delta t\\
&=  - \Delta\tilde{\mathbf R}_{jj}^T\mathbf J_r^{j-1} \Delta t - \sum_{k=i}^{j-2} \Delta\tilde{\mathbf R}_{k+1j}^T\mathbf J_r^k \Delta t\\
&= - \Delta\tilde{\mathbf R}_{jj}^T\mathbf J_r^{j-1} \Delta t - \Delta\tilde{\mathbf R}_{jj-1}\sum_{k=i}^{j-2}(\Delta\tilde{\mathbf R}_{k+1j}\Delta\tilde{\mathbf R}_{jj-1})^T\mathbf J_r^k \Delta t\\
&= - \mathbf J_r^{j-1} \Delta t - \Delta\tilde{\mathbf R}_{j-1j}^T\sum_{k=i}^{j-2} \Delta\tilde{\mathbf R}_{k+1j-1}^T\mathbf J_r^k \Delta t\\
&= - \mathbf J_r^{j-1} \Delta t + \Delta\tilde{\mathbf R}_{j-1j}^T\frac{\partial\Delta\bar{\mathbf R}_{ij-1}}{\partial\mathbf b_i^g}
\end{split}
\end{equation}
$$

这样我们就化成了两个时刻间的迭代形式。其他式子的迭代形式可以很容易写出：
$$
\begin{eqnarray}
\color{green}{\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}} &=& \Delta\tilde{\mathbf R}_{j-1j}^T\color{teal}{\frac{\partial\Delta\bar{\mathbf R}_{ij-1}}{\partial\mathbf b_i^g}} - \mathbf J_r^{j-1} \Delta t\\
\color{green}{\frac{\partial\Delta\bar{\mathbf v}_{ij}}{\partial\mathbf b_i^a}} &=& \color{teal}{\frac{\partial\Delta\bar{\mathbf v}_{ij-1}}{\partial\mathbf b_i^a}} - \Delta\tilde{\mathbf R}_{ij-1}\Delta t\\
\color{green}{\frac{\partial\Delta\bar{\mathbf v}_{ij}}{\partial\mathbf b_i^g}} &=& \color{teal}{\frac{\partial\Delta\bar{\mathbf v}_{ij-1}}{\partial\mathbf b_i^g}} - \Delta\tilde{\mathbf R}_{ij-1}(\tilde{\mathbf a}_{j-1}-\bar{\mathbf b}_i^a)^{\wedge} \color{teal}{\frac{\partial\Delta\bar{\mathbf R}_{ij-1}}{\partial\mathbf b_i^g}} \Delta t\\
\color{green}{\frac{\partial\Delta\bar{\mathbf p}_{ij}}{\partial\mathbf b_i^a}} &=& \color{teal}{\frac{\partial\Delta\bar{\mathbf p}_{ij-1}}{\partial\mathbf b_i^a}} + \color{teal}{\frac{\partial\Delta\bar{\mathbf v}_{ij-1}}{\partial\mathbf b_i^a}}\Delta t - \frac{1}{2}\Delta\bar{\mathbf R}_{ij-1}\Delta t^2\\
\color{green}{\frac{\partial\Delta\bar{\mathbf p}_{ij}}{\partial\mathbf b_i^g}} &=& \color{teal}{\frac{\partial\Delta\bar{\mathbf p}_{ij-1}}{\partial\mathbf b_i^g}} +  \color{teal}{\frac{\partial\Delta\bar{\mathbf v}_{ij-1}}{\partial\mathbf b_i^g}}\Delta t - \frac{1}{2}\Delta\tilde{\mathbf R}_{ij-1}(\tilde{\mathbf a}_{j-1}-\bar{\mathbf b}_i^a)^{\wedge} \color{teal}{\frac{\partial\Delta\bar{\mathbf R}_{ij-1}}{\partial\mathbf b_i^g}} \Delta t^2
\end{eqnarray}
$$
考虑效率，上面存在的相同成分，可以提前用局部变量计算好。

#### **3.3 噪声传播模型**

$$
\begin{equation}
\begin{split}
\delta\mathbf\phi_{ij} &\simeq \sum_{k=i}^{j-1}\Delta \tilde{\mathbf R}_{k+1j}^T\mathbf J_r^k \mathbf\eta_k^{gd}\Delta t\\
&= \sum_{k=i}^{j-2}\Delta \tilde{\mathbf R}_{k+1j}^T\mathbf J_r^k \mathbf\eta_k^{gd}\Delta t + \Delta \tilde {\mathbf R}_{jj}^T\mathbf J_r^{j-1} \mathbf\eta_{j-1}^{gd}\Delta t\\
&= \sum_{k=i}^{j-2}(\Delta \tilde{\mathbf R}_{k+1j-1} \Delta \tilde{\mathbf R}_{j-1j})^T\mathbf J_r^k \mathbf\eta_k^{gd}\Delta t + \mathbf J_r^{j-1} \mathbf\eta_{j-1}^{gd}\Delta t\\
&=  \Delta \tilde{\mathbf R}_{j-1j}^T\sum_{k=i}^{j-2}\Delta \tilde{\mathbf R}_{k+1j-1}^T\mathbf J_r^k \mathbf\eta_k^{gd}\Delta t + \mathbf J_r^{j-1} \mathbf\eta_{j-1}^{gd}\Delta t\\
&= \Delta\tilde{\mathbf R}_{j-1j}^T\delta\phi_{ij-1} + \mathbf J_r^{j-1} \mathbf\eta_{j-1}^{gd}\Delta t
\end{split}
\end{equation}
$$

---

$$
\begin{equation}
\begin{split}
\delta\mathbf v_{ij} =& \sum_{k=i}^{j-1}\begin{bmatrix}-\Delta\tilde{\mathbf R}_{ik}(\tilde{\mathbf a}_k - \mathbf b_i^a)^{\land} \delta\phi_{ik}\Delta t + \Delta\tilde{\mathbf R}_{ik}\mathbf\eta_k^{ad}\Delta t\end{bmatrix}\\
=& \sum_{k=i}^{j-2}\begin{bmatrix}-\Delta\tilde{\mathbf R}_{ik}(\tilde{\mathbf a}_k - \mathbf b_i^a)^{\land} \delta\phi_{ik}\Delta t + \Delta\tilde{\mathbf R}_{ik}\mathbf\eta_k^{ad}\Delta t\end{bmatrix}\\ &-\Delta\tilde{\mathbf R}_{ij-1}(\tilde{\mathbf a}_{j-1} - \mathbf b_i^a)^{\land} \delta\phi_{ij-1}\Delta t + \Delta\tilde{\mathbf R}_{ij-1}\mathbf\eta_{j-1}^{ad}\Delta t\\
=& \delta\mathbf v_{ij-1} - \Delta\tilde{\mathbf R}_{ij-1}(\tilde{\mathbf a}_{j-1} - \mathbf b_i^a)^{\land} \delta\phi_{ij-1}\Delta t + \Delta\tilde{\mathbf R}_{ij-1}\mathbf\eta_{j-1}^{ad}\Delta t
\end{split}
\end{equation}
$$

---

$$
\begin{equation}
\begin{split}
\delta \mathbf p_{ij} =& \sum_{k=i}^{j-1} \begin{bmatrix}\delta\mathbf v_{ik}\Delta t - \frac{1}{2}\Delta\tilde{\mathbf R}_{ik}(\tilde{\mathbf a}_k - \mathbf b_i^a)^{\land} \delta\phi_{ik}\Delta t^2 + \frac{1}{2} \Delta\tilde{\mathbf R}_{ik}\mathbf \eta_k^{ad}\Delta t^2\end{bmatrix}\\
=& \sum_{k=i}^{j-2} \begin{bmatrix}\delta\mathbf v_{ik}\Delta t - \frac{1}{2}\Delta\tilde{\mathbf R}_{ik}(\tilde{\mathbf a}_k - \mathbf b_i^a)^{\land} \delta\phi_{ik}\Delta t^2 + \frac{1}{2} \Delta\tilde{\mathbf R}_{ik}\mathbf \eta_k^{ad}\Delta t^2\end{bmatrix}\\ &+ \delta\mathbf v_{ij-1}\Delta t - \frac{1}{2}\Delta\tilde{\mathbf R}_{ij-1}(\tilde{\mathbf a}_{j-1} - \mathbf b_i^a)^{\land} \delta\phi_{ij-1}\Delta t^2 + \frac{1}{2} \Delta\tilde{\mathbf R}_{ij-1}\mathbf \eta_{j-1}^{ad}\Delta t^2\\
=& \delta\mathbf p_{ij-1}+ \delta\mathbf v_{ij-1}\Delta t - \frac{1}{2}\Delta\tilde{\mathbf R}_{ij-1}(\tilde{\mathbf a}_{j-1} - \mathbf b_i^a)^{\land} \delta\phi_{ij-1}\Delta t^2 + \frac{1}{2} \Delta\tilde{\mathbf R}_{ij-1}\mathbf \eta_{j-1}^{ad}\Delta t^2
\end{split}
\end{equation}
$$

有$\mathbf \eta_{ik}^{\Delta} \doteq [\delta\mathbf\phi_{ik}, \delta\mathbf v_{ik}, \delta\mathbf p_{ik}]$，并且测量噪声为$\mathbf\eta_k^d \doteq [\mathbf\eta_k^{gd}, \mathbf\eta_k^{ad}]$，我们有如下的迭代形式：

$$
\begin{equation}
\mathbf\eta_{ij}^\Delta = \mathbf A_{j-1}\mathbf\eta_{ij-1}^\Delta + \mathbf B_{j-1}\mathbf\eta_{j-1}^d
\end{equation}
$$

这里的$\mathbf A_{j-1}$和$\mathbf B_{j-1}$为：

$$
\begin{equation}
\mathbf A_{j-1} = \begin{bmatrix}
\Delta\tilde{\mathbf R}_{j-1j}^T & 0 & 0\\
\Delta\tilde{\mathbf R}_{ij-1}(\tilde{\mathbf a}_{j-1} - \mathbf b_i^a)^{\land}\Delta t & \mathbf I_{3\times3} & 0\\
- \frac{1}{2}\Delta\tilde{\mathbf R}_{ij-1}(\tilde{\mathbf a}_{j-1} - \mathbf b_i^a)^{\land}\Delta t^2& \mathbf I_{3\times3}\Delta t & \mathbf I_{3\times3}
\end{bmatrix}
\end{equation}
$$

且
$$
\begin{equation}
\mathbf B_{j-1} = \begin{bmatrix}\mathbf B_{j-1}^{gd} & \mathbf B_{j-1}^{ad}\end{bmatrix}
= \begin{bmatrix}
\mathbf J_r^{j-1}\Delta t & \mathbf 0_{3\times3}\\
\mathbf 0_{3\times3} & \Delta\tilde{\mathbf R}_{ij-1}\Delta t\\
\mathbf 0_{3\times3} & \frac{1}{2} \Delta\tilde{\mathbf R}_{ij-1}\Delta t^2
\end{bmatrix}
\end{equation}
$$

从而，给定IMU的协方差$\mathbf\Sigma_{\mathbf\eta}\in\mathbb R^{6\times6}$，协方差的传播可以表示为：
$$
\begin{equation}
\mathbf \Sigma_{ij} = \mathbf A_{j-1}\mathbf\Sigma_{ij-1}\mathbf A_{j-1}^T + \mathbf B_{j-1}\mathbf\Sigma_{\mathbf\eta}\mathbf B_{j-1}^T
\end{equation}
$$
这里需要注意的是，为了减少矩阵运算的规模，可以把$\mathbf B_{j-1}$分开，分别对加速度计和陀螺仪的传播进行计算：
$$
\begin{equation}
\mathbf \Sigma_{ij} = \underbrace{\mathbf A_{j-1}}_{9\times9}\mathbf\Sigma_{ij-1}\mathbf A_{j-1}^T + \underbrace{\mathbf B_{j-1}^{ad}}_{9\times3}\underbrace{\mathbf\Sigma_{\mathbf\eta^{ad}}}_{3\times3}\mathbf B_{j-1}^{ad T} + \underbrace{\mathbf B_{j-1}^{gd}}_{9\times3}\underbrace{\mathbf\Sigma_{\mathbf\eta^{gd}}}_{3\times3}\mathbf B_{j-1}^{gd T}
\end{equation}
$$
这里需要说明的是，离散时间的协方差和连续时间协方差的关系为$\text{Cov}(\mathbf\eta^{d}(t)) = \frac{1}{\Delta t}\text{Cov}(\mathbf\eta(t))$

### **4. IMU预积分残差模型**
#### **4.1 残差模型**
在预积分误差模型中，除了相机的pose之外，IMU的bias同样是`自变量`。通过$\eqref{model_dr}\eqref{model_dv}\eqref{model_dp}$中预积分量和世界坐标系速度、位置、旋转的关系，以及公式$\eqref{bias_correct_r}\eqref{bias_correct_v}\eqref{bias_correct_p}$中预积分对bias的一阶近似，我们可以给出残差的定义：
$$
\begin{eqnarray}
\mathbf r_{\Delta\mathbf R_{ij}} &\doteq& \text{Log}\Big(\Big(\Delta\tilde{\mathbf R}_{ij}(\bar{\mathbf b}_i^g)\text{Exp}\Big(\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\delta\mathbf b_i^g\Big)\Big)^T\mathbf R_i^T\mathbf R_j\Big)\label{res_dr}\\
\mathbf r_{\Delta\mathbf v_{ij}} &\doteq& \mathbf R_i^T(\mathbf v_j - \mathbf v_i - \mathbf g\Delta t_{ij}) - \Big[\Delta\tilde{\mathbf v}_{ij}(\bar{\mathbf b}_i^g, \bar{\mathbf b}_i^g) + \frac{\partial\Delta\bar{\mathbf v}_{ij}}{\partial\mathbf b_i^g} \delta\mathbf b_i^g + \frac{\partial\Delta\bar{\mathbf v}_{ij}}{\partial\mathbf b_i^a} \delta\mathbf b_i^a\Big]\label{res_dv}\\
\mathbf r_{\Delta\mathbf p_{ij}} &\doteq& \mathbf R_i^T(\mathbf p_j - \mathbf p_i - \mathbf v_i \Delta t_{ij} - \frac{1}{2}\mathbf g\Delta t_{ij}^2) - \Big[\Delta\tilde{\mathbf p}_{ij}(\bar{\mathbf b}_i^g, \bar{\mathbf b}_i^g) + \frac{\partial\Delta\bar{\mathbf p}_{ij}}{\partial\mathbf b_i^g} \delta\mathbf b_i^g + \frac{\partial\Delta\bar{\mathbf p}_{ij}}{\partial\mathbf b_i^a} \delta\mathbf b_i^a\Big]\label{res_dp}
\end{eqnarray}
$$

总的IMU误差可以记作$\mathbf r_{\mathcal I_{ij}}\doteq [\mathbf r_{\Delta\mathbf R_{ij}}^T, \mathbf r_{\Delta\mathbf v_{ij}}^T, \mathbf r_{\Delta\mathbf p_{ij}}^T]^T\in \mathbb R^9$ 。接下来我们讨论各个残差的雅可比。

这里给出变量的更新关系：
$$
\mathbf R_i \leftarrow \mathbf R_i\text{Exp}(\delta\mathbf\phi_i) \quad\quad \mathbf R_j \leftarrow \mathbf R_j\text{Exp}(\delta\mathbf\phi_j)\\
\mathbf p_i \leftarrow \mathbf p_i + \delta\mathbf p_i \quad\quad \mathbf p_j \leftarrow \mathbf p_j + \delta\mathbf p_j\\
\mathbf v_i \leftarrow \mathbf v_i + \delta\mathbf v_i \quad\quad \mathbf v_j \leftarrow \mathbf v_j + \delta\mathbf v_j\\
\delta\mathbf b_i^g \leftarrow \delta\mathbf b_i^g + \tilde\delta\mathbf b_i^g \quad\quad  \delta\mathbf b_i^a \leftarrow \delta\mathbf b_i^a + \tilde\delta\mathbf b_i^a
$$

**注意：**其中$\mathbf p$的更新和论文中不同，采用增量进行直接的更新。这里和王京的代码更新方式是一样的。

另外，需要考虑IMU的bias会随时间的变化。**关于布朗运动模型之后再讨论**，这里直接先给出时间连续的关键帧之间bias的约束。
$$
\begin{equation}
\|\mathbf r_{\mathbf b_{ij}}\|^2 = \|\mathbf b_j^g - \mathbf b_i^g\|^2_{\Sigma^{bgd}} + \|\mathbf b_j^a - \mathbf b_i^a\|^2_{\Sigma^{bad}}\label{res_bias}
\end{equation}
$$
这里的协方差$\Sigma^{bgd}\doteq\Delta t_{ij}\text{Cov}(\mathbf\eta^{bg})$，$\Sigma^{bad}\doteq\Delta t_{ij}\text{Cov}(\mathbf\eta^{ba})$



#### **4.2 $\mathbf r_{\Delta\mathbf R_{ij}}$的雅可比**

首先看$\delta\mathbf b_i^g$:
$$
\begin{equation}
\begin{split}
&\mathbf r_{\Delta\mathbf R_{ij}}(\delta\mathbf b_i^g + \tilde\delta\mathbf b_i^g)\\
&= \text{Log}\Big(\Big(\Delta\tilde{\mathbf R}_{ij}(\bar{\mathbf b}_i^g)\text{Exp}\big(\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}(\delta\mathbf b_i^g+\tilde\delta\mathbf b_i^g)\big)\Big)^T\mathbf R_i^T\mathbf R_j\Big)\\
&\stackrel{\eqref{so3_exp_jr}}\approx \text{Log}\Big(\Big(\Delta\tilde{\mathbf R}_{ij}(\bar{\mathbf b}_i^g)\text{Exp}(\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\delta\mathbf b_i^g)\text{Exp}(\mathbf J_r^b\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\tilde\delta\mathbf b_i^g)\Big)^T\mathbf R_i^T\mathbf R_j\Big)\\
&= \text{Log}\Big(\text{Exp}(-\mathbf J_r^b\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\tilde\delta\mathbf b_i^g)\big(\Delta\tilde{\mathbf R}_{ij}(\bar{\mathbf b}_i^g)\text{Exp}(\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\delta\mathbf b_i^g)\big)^T\mathbf R_i^T\mathbf R_j\Big)\\
&= \text{Log}\Big(\text{Exp}(-\mathbf J_r^b\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\tilde\delta\mathbf b_i^g)\text{Exp}\big(\mathbf r_{\Delta\mathbf R_{ij}}(\delta\mathbf b_i^g)\big)\Big)\\
&\stackrel{\eqref{so3_exp_prop2}}= \text{Log}\Big(\text{Exp}\big(\mathbf r_{\Delta\mathbf R_{ij}}(\delta\mathbf b_i^g)\big)\text{Exp}\Big(-\text{Exp}\big(\mathbf r_{\Delta\mathbf R_{ij}}(\delta\mathbf b_i^g)\big)^T\mathbf J_r^b\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\tilde\delta\mathbf b_i^g\Big)\Big)\\
&\stackrel{\eqref{so3_log_jr}}\approx \mathbf r_{\Delta\mathbf R_{ij}}(\delta\mathbf b_i^g) - \mathbf J_r^{-1}\big(\mathbf r_{\Delta\mathbf R_{ij}}(\delta\mathbf b_i^g)\big)\text{Exp}\big(\mathbf r_{\Delta\mathbf R_{ij}}(\delta\mathbf b_i^g)\big)^T\mathbf J_r^b\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\tilde\delta\mathbf b_i^g
\end{split}
\end{equation}
$$
这里$\mathbf J_r^b = \mathbf J_r(\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\delta\mathbf b_i^g)$

尝试使用左乘雅可比
$$
\begin{equation}
\begin{split}
&\mathbf r_{\Delta\mathbf R_{ij}}(\delta\mathbf b_i^g + \tilde\delta\mathbf b_i^g)\\
&= \text{Log}\Big(\Big(\Delta\tilde{\mathbf R}_{ij}(\bar{\mathbf b}_i^g)\text{Exp}\big(\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}(\delta\mathbf b_i^g+\tilde\delta\mathbf b_i^g)\big)\Big)^T\mathbf R_i^T\mathbf R_j\Big)\\
&= \text{Log}\Big(\text{Exp}\big(-\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\delta\mathbf b_i^g - \frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\tilde\delta\mathbf b_i^g\big)\Delta\tilde{\mathbf R}_{ij}(\bar{\mathbf b}_i^g)^T\mathbf R_i^T\mathbf R_j\Big)\\
&\stackrel{\eqref{so3_exp_jl}}\approx \text{Log}\Big(\text{Exp}(-\mathbf J_l(-\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\delta\mathbf b_i^g)\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\tilde\delta\mathbf b_i^g)\text{Exp}(\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\delta\mathbf b_i^g\big)^T\Delta\tilde{\mathbf R}_{ij}(\bar{\mathbf b}_i^g)^T\mathbf R_i^T\mathbf R_j\Big)\\
&\stackrel{\eqref{so3_jac_lr1}\eqref{res_dr}}= \text{Log}\Big(\text{Exp}(-\mathbf J_r(\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\delta\mathbf b_i^g)\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\tilde\delta\mathbf b_i^g)\text{Exp}\big(\mathbf r_{\Delta\mathbf R_{ij}}(\delta\mathbf b_i^g)\big)\Big)\\
&\stackrel{\eqref{so3_log_jl}}= \mathbf r_{\Delta\mathbf R_{ij}}(\delta\mathbf b_i^g) -\mathbf J_l^{-1}\big(\mathbf r_{\Delta\mathbf R_{ij}}(\delta\mathbf b_i^g)\big)\mathbf J_r(\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\delta\mathbf b_i^g)\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\tilde\delta\mathbf b_i^g
\end{split}
\end{equation}
$$
通过$\eqref{so3_jac_lr2}$我们有$\mathbf J_r^{-1}\big(\mathbf r_{\Delta\mathbf R_{ij}}(\delta\mathbf b_i^g)\big)\text{Exp}\big(\mathbf r_{\Delta\mathbf R_{ij}}(\delta\mathbf b_i^g)\big)^T = \mathbf J_l^{-1}\big(\mathbf r_{\Delta\mathbf R_{ij}}(\delta\mathbf b_i^g)\big)$，上述两式是等价的。单纯看来，计算左乘雅可比要比右乘雅可比少去乘以一个旋转矩阵的计算量。

接下来看对旋转$\mathbf R_i$上增量的雅可比
$$
\begin{equation}
\begin{split}
&\mathbf r_{\Delta\mathbf R_{ij}}(\mathbf R_i\text{Exp}(\delta\mathbf\phi_i))\\
&=\text{Log}\Big(\Big(\Delta\tilde{\mathbf R}_{ij}(\bar{\mathbf b}_i^g)\text{Exp}\Big(\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\delta\mathbf b_i^g\Big)\Big)^T\big(\mathbf R_i\text{Exp}(\delta\mathbf\phi_i)\big)^T\mathbf R_j\Big)\\
&= \text{Log}\Big(\Big(\Delta\tilde{\mathbf R}_{ij}(\bar{\mathbf b}_i^g)\text{Exp}\Big(\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\delta\mathbf b_i^g\Big)\Big)^T\text{Exp}(-\delta\mathbf\phi_i)\mathbf R_i^T\mathbf R_j\Big)\\
&\stackrel{\eqref{so3_exp_prop2}}= \text{Log}\Big(\Big(\Delta\tilde{\mathbf R}_{ij}(\bar{\mathbf b}_i^g)\text{Exp}\Big(\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\delta\mathbf b_i^g\Big)\Big)^T\mathbf R_i^T\mathbf R_j\text{Exp}(-\mathbf R_j^T\mathbf R_i\delta\mathbf\phi_i)\Big)\\
&\stackrel{\eqref{res_dr}}= \text{Log}\Big(\text{Exp}(\mathbf r_{\Delta\mathbf R_{ij}})\text{Exp}(-\mathbf R_j^T\mathbf R_i\delta\mathbf\phi_i)\Big)\\
&\stackrel{\eqref{so3_log_jr}} \approx \mathbf r_{\Delta\mathbf R_{ij}} - \mathbf J_r^{-1}(\mathbf r_{\Delta\mathbf R_{ij}})\mathbf R_j^T\mathbf R_i\delta\mathbf\phi_i
\end{split}
\end{equation}
$$

同样对与$\mathbf R_j$:
$$
\begin{equation}
\begin{split}
&\mathbf r_{\Delta\mathbf R_{ij}}(\mathbf R_j\text{Exp}(\delta\mathbf\phi_j))\\
&=\text{Log}\Big(\Big(\Delta\tilde{\mathbf R}_{ij}(\bar{\mathbf b}_i^g)\text{Exp}\Big(\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\delta\mathbf b_i^g\Big)\Big)^T\mathbf R_i^T\mathbf R_j\text{Exp}(\delta\mathbf\phi_j)\Big)\\
&\stackrel{\eqref{so3_log_jr}}\approx \mathbf r_{\Delta\mathbf R_{ij}} + \mathbf J_r^{-1}(\mathbf r_{\Delta\mathbf R_{ij}})\delta\mathbf\phi_j
\end{split}
\end{equation}
$$

最终这里给出的雅可比为：
$$
\begin{equation}
\begin{split}
\color{green}{\frac{\partial \mathbf r_{\Delta\mathbf R_{ij}}}{\partial\delta\phi_i}} &= - \mathbf J_r^{-1}(\mathbf r_{\Delta\mathbf R_{ij}}(\mathbf R_i))\mathbf R_j^T\mathbf R_i\\
\color{green}{\frac{\partial \mathbf r_{\Delta\mathbf R_{ij}}}{\partial\delta\phi_j}} &= \mathbf J_r^{-1}(\mathbf r_{\Delta\mathbf R_{ij}}(\mathbf R_j))\\
\color{green}{\frac{\partial \mathbf r_{\Delta\mathbf R_{ij}}}{\partial\tilde\delta\mathbf b_i^g}} &= -\mathbf J_l^{-1}\big(\mathbf r_{\Delta\mathbf R_{ij}}(\delta\mathbf b_i^g)\big)\mathbf J_r(\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}\delta\mathbf b_i^g)\frac{\partial\Delta\bar{\mathbf R}_{ij}}{\partial\mathbf b_i^g}
\end{split}
\end{equation}
$$

可以发现对旋转量的偏导都使用了$\mathbf r_{\Delta\mathbf R_{ij}}$对应的右乘雅可比，因此对bias的求导虽然计算左乘雅可比少乘一个矩阵，但是需要重新计算左乘雅可比，因此选择左乘可以不还是右乘雅可比，计算量可能差不了多少（可以进一步比较一下）。

由于在对旋转做更新的时候是右乘一个扰动的方式，如果使用直接在切空间上做加法的话，通过公式$\eqref{so3_exp_jr}$，则可以把雅可比变为：
$$
\begin{split}
\frac{\partial \mathbf r_{\Delta\mathbf R_{ij}}}{\partial\delta\phi_i} &= - \mathbf J_r^{-1}(\mathbf r_{\Delta\mathbf R_{ij}}(\mathbf R_i))\mathbf R_j^T\mathbf R_i\mathbf J_r(\mathbf R_i)\\
\frac{\partial \mathbf r_{\Delta\mathbf R_{ij}}}{\partial\delta\phi_j} &= \mathbf J_r^{-1}(\mathbf r_{\Delta\mathbf R_{ij}}(\mathbf R_j))\mathbf J_r(\mathbf R_j)
\end{split}
$$

#### **4.3 $\mathbf r_{\Delta\mathbf v_{ij}}$的雅可比**
先回顾残差公式$\eqref{res_dv}$:
$$
\begin{split}
\mathbf r_{\Delta\mathbf v_{ij}} &= \mathbf R_i^T(\mathbf v_j - \mathbf v_i - \mathbf g\Delta t_{ij}) - \Big[\Delta\tilde{\mathbf v}_{ij}(\bar{\mathbf b}_i^g, \bar{\mathbf b}_i^g) + \frac{\partial\Delta\bar{\mathbf v}_{ij}}{\partial\mathbf b_i^g} \delta\mathbf b_i^g + \frac{\partial\Delta\bar{\mathbf v}_{ij}}{\partial\mathbf b_i^a} \delta\mathbf b_i^a\Big]\\
&\doteq \mathbf R_i^T(\mathbf v_j - \mathbf v_i - \mathbf g\Delta t_{ij}) - D
\end{split}
$$

首先看对速度$\mathbf v_i$和$\mathbf v_j$的雅可比，
$$
\begin{equation}
\begin{split}
\mathbf r_{\Delta\mathbf v_{ij}}(\mathbf v_i + \delta\mathbf v_i) &= \mathbf R_i^T(\mathbf v_j - \mathbf v_i - \delta\mathbf v_i- \mathbf g\Delta t_{ij}) - D\\
&= \mathbf r_{\Delta\mathbf v_{ij}}(\mathbf v_i) - \mathbf R_i^T\delta\mathbf v_i
\end{split}
\end{equation}
$$

$$
\begin{equation}
\begin{split}
\mathbf r_{\Delta\mathbf v_{ij}}(\mathbf v_j + \delta\mathbf v_j) &= \mathbf R_i^T(\mathbf v_j  + \delta\mathbf v_j - \mathbf v_i - \mathbf g\Delta t_{ij}) - D\\
&= \mathbf r_{\Delta\mathbf v_{ij}}(\mathbf v_i) + \mathbf R_i^T\delta\mathbf v_j
\end{split}
\end{equation}
$$

然后看对旋转$\mathbf R_i$的雅可比：
$$
\begin{equation}
\begin{split}
\mathbf r_{\Delta\mathbf v_{ij}}(\mathbf R_i \text{Exp}(\delta\mathbf \phi_i)) &= \text{Exp}(\delta\mathbf \phi_i)^T\mathbf R_i^T(\mathbf v_j - \mathbf v_i - \mathbf g\Delta t_{ij}) - D\\
&\approx (\mathbf I - \delta\mathbf \phi_i^{\land})\mathbf R_i^T(\mathbf v_j - \mathbf v_i - \mathbf g\Delta t_{ij}) - D\\
&= \mathbf r_{\Delta\mathbf v_{ij}}(\mathbf R_i) + \big(\mathbf R_i^T(\mathbf v_j - \mathbf v_i - \mathbf g\Delta t_{ij})\big)^\land \delta\mathbf \phi_i
\end{split}
\end{equation}
$$

剩下的是bias的雅可比，直接可以看出来，汇总一下非零的雅可比：
$$
\begin{equation}
\begin{split}
\color{green}{\frac{\partial \mathbf r_{\Delta\mathbf v_{ij}}}{\partial\delta\phi_i}} &= \big(\mathbf R_i^T(\mathbf v_j - \mathbf v_i - \mathbf g\Delta t_{ij})\big)^\land\\
\color{green}{\frac{\partial \mathbf r_{\Delta\mathbf v_{ij}}}{\partial\tilde\delta\mathbf b_i^a}} &= -\frac{\partial\Delta\bar{\mathbf v}_{ij}}{\partial\mathbf b_i^a}\\
\color{green}{\frac{\partial \mathbf r_{\Delta\mathbf v_{ij}}}{\partial\tilde\delta\mathbf b_i^g}} &= -\frac{\partial\Delta\bar{\mathbf v}_{ij}}{\partial\mathbf b_i^g}\\
\color{green}{\frac{\partial \mathbf r_{\Delta\mathbf v_{ij}}}{\partial\delta\mathbf v_i}} &= -\mathbf R_i^T\\
\color{green}{\frac{\partial \mathbf r_{\Delta\mathbf v_{ij}}}{\partial\delta\mathbf v_j}} &= \mathbf R_i^T
\end{split}
\end{equation}
$$

#### **4.4 $\mathbf r_{\Delta\mathbf p_{ij}}$的雅可比**
回顾残差$\eqref{res_dp}$
$$
\begin{split}
\mathbf r_{\Delta\mathbf p_{ij}} &= \mathbf R_i^T(\mathbf p_j - \mathbf p_i - \mathbf v_i \Delta t_{ij} - \frac{1}{2}\mathbf g\Delta t_{ij}^2) - \Big[\Delta\tilde{\mathbf p}_{ij}(\bar{\mathbf b}_i^g, \bar{\mathbf b}_i^g) + \frac{\partial\Delta\bar{\mathbf p}_{ij}}{\partial\mathbf b_i^g} \delta\mathbf b_i^g + \frac{\partial\Delta\bar{\mathbf p}_{ij}}{\partial\mathbf b_i^a} \delta\mathbf b_i^a\Big]
\end{split}
$$

和上一节的推导相似，我们可以直接写出非零的雅可比
$$
\begin{equation}
\begin{split}
\color{green}{\frac{\partial \mathbf r_{\Delta\mathbf p_{ij}}}{\partial\delta\mathbf p_i}} &= -\mathbf R_i^T\\
\color{green}{\frac{\partial \mathbf r_{\Delta\mathbf p_{ij}}}{\partial\delta\phi_i}} &= \big(\mathbf R_i^T(\mathbf p_j - \mathbf p_i - \mathbf v_i \Delta t_{ij} - \frac{1}{2}\mathbf g\Delta t_{ij}^2)\big)^\land\\
\color{green}{\frac{\partial \mathbf r_{\Delta\mathbf p_{ij}}}{\partial\delta\mathbf p_j}} &= \mathbf R_i^T\\
\color{green}{\frac{\partial \mathbf r_{\Delta\mathbf p_{ij}}}{\partial\tilde\delta\mathbf b_i^a}} &= -\frac{\partial\Delta\bar{\mathbf p}_{ij}}{\partial\mathbf b_i^a}\\
\color{green}{\frac{\partial \mathbf r_{\Delta\mathbf p_{ij}}}{\partial\tilde\delta\mathbf b_i^g}} &= -\frac{\partial\Delta\bar{\mathbf p}_{ij}}{\partial\mathbf b_i^g}\\
\color{green}{\frac{\partial \mathbf r_{\Delta\mathbf p_{ij}}}{\partial\delta\mathbf v_i}} &= -\mathbf R_i^T\Delta t_{ij}
\end{split}
\end{equation}
$$

#### **4.5 整体雅可比**
在优化过程中，由两个关键帧之间的IMU数据可以构建一个15维的残差，定义为：
$$
\begin{equation}
\mathbf r_\text{IMU} = \begin{bmatrix} \mathbf r_{\Delta\mathbf p_{ij}}^T & \mathbf r_{\Delta\mathbf R_{ij}}^T &  \mathbf r_{\Delta\mathbf v_{ij}}^T & \mathbf r_{\mathbf b_{ij}}^T\end{bmatrix}^T
\end{equation}
$$

整体残差的维数为$3+3+3+6=15$。整体的参数变量为：
$$
\begin{equation}
\delta\mathbf p = \begin{bmatrix}\delta{\mathbf p}_i^T & \delta\mathbf\phi_i^T & \delta{\mathbf p}_j^T &  \delta\mathbf\phi_j^T & \delta\mathbf v_i^T  & \delta\mathbf v_j^T & \tilde\delta{\mathbf b_i^g}^T & \tilde\delta{\mathbf b_i^a}^T\end{bmatrix}^T\\
= \begin{bmatrix} \delta\mathbf\xi_i^T & \delta\mathbf\xi_j^T & \delta\mathbf v_i^T & \delta\mathbf v_j^T& \tilde\delta{\mathbf b_i^g}^T & \tilde\delta{\mathbf b_i^a}^T\end{bmatrix}^T
\end{equation}
$$
分别是第$i$、$j$关键帧的旋转、位移、速度的增量以及第$i$帧关键帧的IMU bias。通常位姿都使用$\text{SE}(3)$来表示，因此这里的参数块中$R$、$t$的分量就写在了一起，并且按照$\text{SE}(3)$的存储顺序写了（先平移后旋转）。**需要注意的是，这里的位姿都是IMU的位姿，如果使用相机的位姿，需要做转换。**

这里写出完整的雅可比矩阵如下：
$$
\begin{equation}
\begin{split}
\mathbf J_{\mathbf r_\text{IMU}} = \frac{\partial \mathbf r_\text{IMU}}{\partial \delta\mathbf p}
= \begin{pmatrix}
\frac{\partial \mathbf r_{\Delta\mathbf p_{ij}}}{\partial \delta\mathbf p}^T & \frac{\partial \mathbf r_{\Delta\mathbf R_{ij}}}{\partial\delta\mathbf p}^T & \frac{\partial \mathbf r_{\Delta\mathbf v_{ij}}}{\partial \delta\mathbf p}^T & \frac{\partial \mathbf r_{\Delta\mathbf b_{ij}}}{\partial \delta\mathbf p}^T
\end{pmatrix}^T
\end{split}
\end{equation}
$$

可以先写出每一个残差对应的雅可比：
$$
\begin{equation}
\begin{pmatrix}
\frac{\partial \mathbf r_{\Delta\mathbf p_{ij}}}{\partial \delta\mathbf p}\\ \frac{\partial \mathbf r_{\Delta\mathbf R_{ij}}}{\partial\delta\mathbf p}\\ \frac{\partial \mathbf r_{\Delta\mathbf v_{ij}}}{\partial \delta\mathbf p}\\ \frac{\partial \mathbf r_{\Delta\mathbf b_{ij}}}{\partial \delta\mathbf p}
\end{pmatrix} = 
\begin{pmatrix}
\frac{\partial \mathbf r_{\Delta\mathbf p_{ij}}}{\partial\delta\mathbf p_i} & \frac{\partial \mathbf r_{\Delta\mathbf p_{ij}}}{\partial \delta\phi_i} & \frac{\partial \mathbf r_{\Delta\mathbf p_{ij}}}{\partial\delta\mathbf p_j} & \mathbf 0_{3\times3} & \frac{\partial \mathbf r_{\Delta\mathbf p_{ij}}}{\partial\delta\mathbf v_i} & \mathbf 0_{3\times3} & \frac{\partial \mathbf r_{\Delta\mathbf p_{ij}}}{\partial\tilde\delta\mathbf b_i^g} & \frac{\partial \mathbf r_{\Delta\mathbf p_{ij}}}{\partial\tilde\delta\mathbf b_i^a}\\
\mathbf 0_{3\times3} & \frac{\partial \mathbf r_{\Delta\mathbf R_{ij}}}{\partial \delta\phi_i} & \mathbf 0_{3\times3} & \frac{\partial \mathbf r_{\Delta\mathbf R_{ij}}}{\partial \delta\phi_j} & \mathbf 0_{3\times3} & \mathbf 0_{3\times3} & \frac{\partial \mathbf r_{\Delta\mathbf R_{ij}}}{\partial\tilde\delta\mathbf b_i^g} & \mathbf 0_{3\times3}\\
\mathbf 0_{3\times3} & \frac{\partial \mathbf r_{\Delta\mathbf v_{ij}}}{\partial\delta\phi_i} & \mathbf 0_{3\times3} & \mathbf 0_{3\times3} & \frac{\partial \mathbf r_{\Delta\mathbf v_{ij}}}{\partial\delta\mathbf v_i} & \frac{\partial \mathbf r_{\Delta\mathbf v_{ij}}}{\partial\delta\mathbf v_j} & \frac{\partial \mathbf r_{\Delta\mathbf v_{ij}}}{\partial\tilde\delta\mathbf b_i^g} & \frac{\partial \mathbf r_{\Delta\mathbf v_{ij}}}{\partial\tilde\delta\mathbf b_i^a}\\
\mathbf 0_{3\times3} & \mathbf 0_{3\times3} & \mathbf 0_{3\times3} & \mathbf 0_{3\times3} & \mathbf 0_{3\times3} & \mathbf 0_{3\times3} & \frac{\partial \mathbf r_{\Delta\mathbf b_{ij}}}{\partial\tilde\delta\mathbf b_i^g} & \frac{\partial \mathbf r_{\Delta\mathbf b_{ij}}}{\partial\tilde\delta\mathbf b_i^a}
\end{pmatrix}
\end{equation}
$$


### **5. IMU初始化**
#### **5.1 估计陀螺仪Bias**
通过预积分中的旋转分量$\eqref{model_dr}$可以求解出陀螺仪的bias。通过$\eqref{res_dr}$给出如下代价函数：
$$
\begin{equation}
\arg\min_{\mathbf b_g}\sum_{i=1}^{N-1}\begin{Vmatrix}\text{Log}
\big((\Delta\mathbf R_{i,i+1}\text{Exp}(\mathbf J_{\Delta\mathbf R}^g\mathbf b_g))^T\mathbf R_{\tt BW}^{i+1}\mathbf R_{\tt WB}^i\big)
\end{Vmatrix}^2
\end{equation}
$$
这里$N$为关键帧的个数。旋转$\mathbf R_{\tt WB}^{(\cdot)} = \mathbf R_{\tt WC}^{(\cdot)} \mathbf R_{\tt CB}$，$\mathbf R_{\tt WC}^{(\cdot)}$则是由SLAM中的关键帧得到的。通过Gauss-Newton就可以解得陀螺仪的bias。

#### **5.2 估计重力和尺度**
考虑到相机camera坐标系与IMU的body坐标系的关系，有$\mathbf T_{\tt CB} = [\mathbf R_{\tt CB}| \mathbf p_{\tt CB}]$，考虑尺度$s$后两坐标系间转换关系有：
$$
\begin{equation}
\mathbf p_{\tt WB} = s\mathbf p_{\tt WC} + \mathbf R_{\tt WC}\, \mathbf p_{\tt CB}
\end{equation}
$$

这里考虑每两个关键帧之间的预积分量，$\eqref{model_dp}$有：
$$
\begin{equation}
\begin{split}
\Delta\mathbf p_{ij} &= \mathbf R_{ {\tt WB}_i}^T(\mathbf p_{ {\tt WB}_j} - \mathbf p_{ {\tt WB}_i} - \mathbf v_{ {\tt WB}_i} \Delta t_{ij} - \frac{1}{2}\mathbf g_{\tt W}\Delta t_{ij}^2 )\\
&= \mathbf R_{ {\tt WB}_i}^T((s\mathbf p_{ {\tt WC}_j} + \mathbf R_{ {\tt WC}_j}\, \mathbf p_{\tt CB}) - (s\mathbf p_{ {\tt WC}_i} + \mathbf R_{ {\tt WC}_i}\, \mathbf p_{\tt CB}) - \mathbf v_{ {\tt WB}_i} \Delta t_{ij} - \frac{1}{2}\mathbf g_{\tt W}\Delta t_{ij}^2)\\
&= \mathbf R_{ {\tt WB}_i}^T(s(\mathbf p_{ {\tt WC}_j} - \mathbf p_{ {\tt WC}_i}) + (\mathbf R_{ {\tt WC}_j} - \mathbf R_{ {\tt WC}_i})\mathbf p_{\tt CB} - \mathbf v_{ {\tt WB}_i} \Delta t_{ij} - \frac{1}{2}\mathbf g_{\tt W}\Delta t_{ij}^2)
\end{split}
\end{equation}\label{dpij_scale}
$$

在论文中这里用$\mathbf p_{ {\tt WC}_i}$和$\mathbf p_{ {\tt WC}_j}$的关系表示，把上式整理一下可以得到论文中的形式：
$$
\begin{equation}
s\mathbf p_{ {\tt WC}_j}  =  s\mathbf p_{ {\tt WC}_i} + \mathbf R_{ {\tt WB}_i}\Delta\mathbf p_{ij} - (\mathbf R_{ {\tt WC}_j} - \mathbf R_{ {\tt WC}_i})\mathbf p_{\tt CB} + \mathbf v_{ {\tt WB}_i} \Delta t_{ij} + \frac{1}{2}\mathbf g_{\tt W}\Delta t_{ij}^2
\end{equation}
$$

我们不希望求解速度，因此我们可以通过相邻两个预积分的速度来消去。使用3个关键帧的数据分别记作$1,2,3$，通过式子$\eqref{model_dp}$和$\eqref{model_dv}$则有：
$$
\begin{equation}
\begin{split}
\Delta\mathbf p_{12} &= \mathbf R_{\tt WB_1}^T(s(\mathbf p_{\tt WC_2} - \mathbf p_{\tt WC_1}) + (\mathbf R_{\tt WC_2}-\mathbf R_{\tt WC_1})\mathbf p_{\tt CB} - \mathbf v_{\tt WB_1} \Delta t_{12} - \frac{1}{2}\mathbf g_{\tt W}\Delta t_{12}^2 )\\
\Delta\mathbf p_{23} &= \mathbf R_{\tt WB_2}^T(s(\mathbf p_{\tt WC_3} - \mathbf p_{\tt WC_2}) + (\mathbf R_{\tt WC_3}-\mathbf R_{\tt WC_2})\mathbf p_{\tt CB} - \mathbf v_{\tt WB_2} \Delta t_{23} - \frac{1}{2}\mathbf g_{\tt W}\Delta t_{23}^2 )\\
\Delta\mathbf v_{12} &= \mathbf R_{\tt WB_1}^T(\mathbf v_{\tt WB_2} - \mathbf v_{\tt WB_1} - \mathbf g_{\tt W}\Delta t_{12})
\end{split}
\end{equation}
$$
前两式代入第三式可得
$$
\begin{equation}
\begin{split}
(\mathbf R_{\tt WB_1}\Delta\mathbf v_{12} + \mathbf g_{\tt W}\Delta t_{12})\Delta t_{12}\Delta t_{23} = \mathbf v_{\tt WB_2}\Delta t_{23}\Delta t_{12} - \mathbf v_{\tt WB_1}\Delta t_{12}\Delta t_{23} \\
= (-\mathbf R_{\tt WB_2}\Delta\mathbf p_{23} + s(\mathbf p_{\tt WC_3} - \mathbf p_{\tt WC_2}) + (\mathbf R_{\tt WC_3}-\mathbf R_{\tt WC_2})\mathbf p_{\tt CB} - \frac{1}{2}\mathbf g_{\tt W}\Delta t_{23}^2 )\Delta t_{12}
\\- (-\mathbf R_{\tt WB_1}\Delta\mathbf p_{12} + s(\mathbf p_{\tt WC_2} - \mathbf p_{\tt WC_1}) + (\mathbf R_{\tt WC_2}-\mathbf R_{\tt WC_1})\mathbf p_{\tt CB} - \frac{1}{2}\mathbf g_{\tt W}\Delta t_{12}^2 )\Delta t_{23}
\end{split}
\end{equation}
$$

化简得：
$$
\begin{equation}
\begin{split}
&(-(\mathbf p_{\tt WC_3} - \mathbf p_{\tt WC_2})\Delta t_{12} + (\mathbf p_{\tt WC_2} - \mathbf p_{\tt WC_1})\Delta t_{23})s\\
&+ \frac{1}{2}\mathbf g_{\tt W}(\Delta t_{12}^2\Delta t_{23}+\Delta t_{23}^2\Delta t_{12})\\
&= -\mathbf R_{\tt WB_1}\Delta\mathbf v_{12}\Delta t_{12}\Delta t_{23} - \mathbf R_{\tt WB_2}\Delta\mathbf p_{23}\Delta t_{12} + (\mathbf R_{\tt WC_3}-\mathbf R_{\tt WC_2})\mathbf p_{\tt CB}\Delta t_{12}\\
& + \mathbf R_{\tt WB_1}\Delta\mathbf p_{12}\Delta t_{23} - (\mathbf R_{\tt WC_2}-\mathbf R_{\tt WC_1})\mathbf p_{\tt CB}\Delta t_{23}
\end{split}
\end{equation}\label{estimate_s_g_detail}
$$

这里就有$\mathbf A\mathbf x=\mathbf b$的形式：
$$
\begin{equation}
\begin{bmatrix}\lambda(i) & \beta(i)\end{bmatrix}\begin{bmatrix}s \\ \mathbf g_{\tt W}\end{bmatrix} = \gamma(i)\label{estimate_s_g}
\end{equation}
$$

对于关键帧$i,i+1,i+2$，记作$1,2,3$。则有：
$$
\begin{equation}
\begin{split}
\lambda(i) &= (\mathbf p_{\tt WC_2} - \mathbf p_{\tt WC_3})\Delta t_{12} + (\mathbf p_{\tt WC_2} - \mathbf p_{\tt WC_1})\Delta t_{23}\\
\beta(i) &= \frac{1}{2}\mathbf I_{3\times3}(\Delta t_{12}^2\Delta t_{23}+\Delta t_{23}^2\Delta t_{12})\\
\gamma(i) &= -\mathbf R_{\tt WB_1}\Delta\mathbf v_{12}\Delta t_{12}\Delta t_{23} - \mathbf R_{\tt WB_2}\Delta\mathbf p_{23}\Delta t_{12} + (\mathbf R_{\tt WC_3}-\mathbf R_{\tt WC_2})\mathbf p_{\tt CB}\Delta t_{12}\\
& + \mathbf R_{\tt WB_1}\Delta\mathbf p_{12}\Delta t_{23} - (\mathbf R_{\tt WC_2}-\mathbf R_{\tt WC_1})\mathbf p_{\tt CB}\Delta t_{23}
\end{split}
\end{equation}
$$

注意，这里$\mathbf R_{\tt WBi} = \mathbf R_{\tt WCi}\mathbf R_{\tt CB}$。

对于一系列的数据，我们就可以通过SVD解最小二乘得到尺度$s$和重力向量$_w\mathbf g$。我们用3个关键帧，得到$\eqref{estimate_s_g}$式中等式个数为3。每增加一个IMU预积分观测，就可以增加一个$\eqref{estimate_s_g}$式。最终最小二乘的规模为：$\mathbf A_{3(N-2)\times4}\mathbf x_{4\times1} = \mathbf b_{3(N-2)\times1}$。为了解出4个未知数，我们至少需要4个等式，也就是最少$N=4$时就可以解出尺度和重力。

需要注意的是，这里的$\gamma(i)$和论文中的不太一样，整体差了一个负号。这里的公式和王京的代码是对应的，看样子王京也是从头把公式推导了一边。论文中的形式会使得最终解差一个负号。

#### **5.3 估计加速度Bias并优化尺度和重力**

给定一个惯性参考系$I$，在该坐标系下重力加速度的方向记为$\hat{\mathbf g}_{\tt I}=(0,0,-1)^T$。通过上述的求解，已知重力的方向为$\hat{\mathbf g}_{\tt W} = \mathbf g_{\tt W}^*/\|\mathbf g_{\tt W}^*\|$，可以求解出两个向量间的旋转：
$$
\begin{equation}
\mathbf R_{\tt WI} = \text{Exp}(\hat{\mathbf v}\theta)\\
\hat{\mathbf v} = \frac{\hat{\mathbf g}_{\tt I} \times \hat{\mathbf g}_{\tt W}}{\|\hat{\mathbf g}_{\tt I} \times \hat{\mathbf g}_{\tt W}\|}, \quad \theta = \text{atan2}(\|\hat{\mathbf g}_{\tt I} \times \hat{\mathbf g}_{\tt W}\|, \hat{\mathbf g}_{\tt I} \cdot \hat{\mathbf g}_{\tt W})
\end{equation}
$$
设重力大小为$G$，则有：
$$
\begin{equation}
\mathbf g_{\tt W} = \mathbf R_{\tt WI}\hat{\mathbf g}_{\tt I} G
\end{equation}
$$
从旋转向量的角度来考虑，在$I$坐标系下，由于$\hat{\mathbf g}_{\tt I}$是固定在$z$轴上，只需要在$x$和$y$轴上进行旋转即可。这里的旋转可以通过给定局部的扰动$\delta\mathbf\theta$进行优化：
$$
\begin{equation}
\mathbf g_{\tt W} = \mathbf R_{\tt WI}\text{Exp}(\delta\mathbf\theta)\hat{\mathbf g}_{\tt I} G
\end{equation}
$$
这里$\delta\mathbf\theta=[\delta\mathbf\theta_{xy}^T \  0]^T = [\delta\theta_x \ \delta\theta_y \  0]^T$。使用一阶近似可得：
$$
\begin{equation}
\mathbf g_{\tt W} = \mathbf R_{\tt WI}(\mathbf I_{3\times3} + [\delta\mathbf\theta]_\times)\hat{\mathbf g}_{\tt I} G = \mathbf R_{\tt WI}\hat{\mathbf g}_{\tt I} G - \mathbf R_{\tt WI}[\hat{\mathbf g}_{\tt I}]_\times G\delta\mathbf\theta
\end{equation}
$$

和上一部分相同，把上式代入到$\eqref{model_dp}$和$\eqref{model_dv}$中，并且考虑$\eqref{bias_correct_p}$和$\eqref{bias_correct_v}$中加速度的bias，与上一节比较有如下变化：
$$
\begin{equation}
\begin{split}
\Delta\mathbf p_{ij} &:= \Delta\mathbf p_{ij} + \mathbf J^a_{\Delta\mathbf p_{ij}}\mathbf b_a\\
\Delta\mathbf v_{ij} &:= \Delta\mathbf v_{ij} + \mathbf J^a_{\Delta\mathbf v_{ij}}\mathbf b_a\\
\mathbf g_{\tt W} &:= \mathbf R_{\tt WI}\hat{\mathbf g}_{\tt I} G - \mathbf R_{\tt WI}[\hat{\mathbf g}_{\tt I}]_\times G\delta\mathbf\theta
\end{split}
\end{equation}
$$

对照$\eqref{estimate_s_g_detail}$我们有：
$$
\begin{equation}
\begin{split}
&((\mathbf p_{\tt WC_2} - \mathbf p_{\tt WC_3})\Delta t_{12} + (\mathbf p_{\tt WC_2} - \mathbf p_{\tt WC_1})\Delta t_{23})s\\
&+ \frac{1}{2}\mathbf R_{\tt WI}\hat{\mathbf g}_{\tt I} G(\Delta t_{12}^2\Delta t_{23}+\Delta t_{23}^2\Delta t_{12})\\
&- \frac{1}{2}\mathbf R_{\tt WI}[\hat{\mathbf g}_{\tt I}]_\times G(\Delta t_{12}^2\Delta t_{23}+\Delta t_{23}^2\Delta t_{12})\delta\mathbf\theta\\
&= -\mathbf R_{\tt WB_1}\Delta\mathbf v_{12}\Delta t_{12}\Delta t_{23} - \mathbf R_{\tt WB_2}\Delta\mathbf p_{23}\Delta t_{12} + (\mathbf R_{\tt WC_3}-\mathbf R_{\tt WC_2})\mathbf p_{\tt CB}\Delta t_{12}\\
&+ \mathbf R_{\tt WB_1}\Delta\mathbf p_{12}\Delta t_{23} - (\mathbf R_{\tt WC_2}-\mathbf R_{\tt WC_1})\mathbf p_{\tt CB}\Delta t_{23}\\
&- \mathbf R_{\tt WB_1}\mathbf J^a_{\Delta\mathbf v_{12}}\mathbf b_a\Delta t_{12}\Delta t_{23} - \mathbf R_{\tt WB_2}\mathbf J^a_{\Delta\mathbf p_{23}}\mathbf b_a\Delta t_{12} + \mathbf R_{\tt WB_1}\mathbf J^a_{\Delta\mathbf p_{12}}\mathbf b_a\Delta t_{23}
\end{split}
\end{equation}
$$

这里我们可以写成如下形式：
$$
\begin{equation}
\begin{bmatrix}\lambda(i) & \phi(i) & \zeta(i)\end{bmatrix} \begin{bmatrix}s \\ \delta\mathbf\theta_{xy} \\ \mathbf b_a\end{bmatrix} = \psi(i)
\end{equation}
$$
这里的$\lambda(i)$和上一节相同，其他部分对应有：
$$
\begin{equation}
\begin{split}
\phi(i) &= \bigg[-\frac{1}{2}\mathbf R_{\tt WI}[\hat{\mathbf g}_{\tt I}]_\times G(\Delta t_{12}^2\Delta t_{23}+\Delta t_{23}^2\Delta t_{12})\bigg]_{(:,1:2)}\\
\zeta(i) &= \mathbf R_{\tt WB_1}\mathbf J^a_{\Delta\mathbf v_{12}}\Delta t_{12}\Delta t_{23} + \mathbf R_{\tt WB_2}\mathbf J^a_{\Delta\mathbf p_{23}}\Delta t_{12} - \mathbf R_{\tt WB_1}\mathbf J^a_{\Delta\mathbf p_{12}}\Delta t_{23}\\
\psi(i) &= \gamma(i) - \frac{1}{2}\mathbf R_{\tt WI}\hat{\mathbf g}_{\tt I} G(\Delta t_{12}^2\Delta t_{23}+\Delta t_{23}^2\Delta t_{12})
\end{split}
\end{equation}
$$
这里的$[\cdot]_{(:,1:2)}$代表只取前两列。最终我们得到的形式为：$\mathbf A_{3(N-2)\times6}\mathbf x_{6\times1} = \mathbf b_{3(N-2)\times1}$，这里我们需要求解$6$个未知数，因此至少需要$4$个关键帧。

#### **5.4 计算速度**

通过式子$\eqref{dpij_scale}$，我们可以计算得到关键帧的速度
$$
\begin{equation}
\mathbf v_{ {\tt WB}_i} \Delta t_{ij}
= - \mathbf R_{ {\tt WB}_i}\Delta\mathbf p_{ij} + s(\mathbf p_{ {\tt WC}_j} - \mathbf p_{ {\tt WC}_i}) + (\mathbf R_{ {\tt WC}_j} - \mathbf R_{ {\tt WC}_i})\mathbf p_{\tt CB} - \frac{1}{2}\mathbf g_{\tt W}\Delta t_{ij}^2
\end{equation}
$$

### **6. 边缘化与滑窗优化**
#### **6.1 边缘化**
对于整个优化方程，根据需要边缘化和需要保留的变量，写成分块矩阵的形式：
$$
\begin{equation}
\begin{pmatrix}
\mathbf H_{rr} & \mathbf H_{rm}\\
\mathbf H_{rm}^T & \mathbf H_{mm}
\end{pmatrix}
\begin{pmatrix}\delta\mathbf x_r\\\delta\mathbf x_m\end{pmatrix}
= \begin{pmatrix}\mathbf b_r\\\mathbf b_m\end{pmatrix}\label{sys_func}
\end{equation}
$$
其中脚标$r$代表需要保留的部分，$m$代表需要被被边缘化的部分。使用舒尔补得
$$
\begin{equation}
\begin{pmatrix}
\mathbf H_{rr}- \mathbf H_{rm}\mathbf H_{mm}^{-1}\mathbf H_{rm}^T & \mathbf 0\\
\mathbf H_{rm}^T & \mathbf H_{mm}
\end{pmatrix}
\begin{pmatrix}
\delta\mathbf x_r\\
\delta\mathbf x_m
\end{pmatrix}
= \begin{pmatrix}
\mathbf b_r - \mathbf H_{rm}\mathbf H_{mm}^{-1}\mathbf b_m\\
\mathbf b_m
\end{pmatrix}
\end{equation}
$$

这里就可以得到只有参数$\delta\mathbf x_r$的方程：
$$
\begin{equation}
(\mathbf H_{rr}- \mathbf H_{rm}\mathbf H_{mm}^{-1}\mathbf H_{rm}^T)\delta\mathbf x_r = \mathbf b_r - \mathbf H_{rm}\mathbf H_{mm}^{-1}\mathbf b_m\label{marg_remian}
\end{equation}
$$

在使用**FEJ**的时候，这里的雅可比一直不会变化，也就是我们构成的边缘化先验部分的误差项，在计算边缘化之后，在后续优化中，雅可比保持不变。但是$\mathbf b$向量和残差相关，我们有$\mathbf b = -\mathbf J^T\mathbf\Omega\mathbf e$，在后续优化中，一旦估计的状态量发生改变，这里的$\mathbf b$向量就需要重新计算。如果这样处理，会导致每优化一步，之前计算边缘化的系统方程就改变了，边缘化也就要重新计算。

为了避免这样的操作，对$\mathbf b$做进一步处理。设初始化线性化点为$\mathbf x_0$，当前状态量为$\mathbf x = \mathbf x_0 \boxplus \Delta\mathbf x$我们对$\mathbf b$做一阶泰勒展开：
$$
\begin{equation}
\begin{split}
\mathbf b &= \mathbf b_0 + \frac{\partial\mathbf b}{\partial\mathbf x}\bigg|_{\mathbf x = \mathbf x_0}\Delta\mathbf x\\
&= \mathbf b_0 -\mathbf J^T\mathbf\Omega\frac{\partial\mathbf e}{\partial\mathbf x}\bigg|_{\mathbf x = \mathbf x_0}\Delta\mathbf x\\
&= \mathbf b_0 -\mathbf H\Delta\mathbf x
\end{split}
\end{equation}
$$

然后式子$\eqref{sys_func}$的右侧就改变为：
$$
\begin{equation}
\begin{pmatrix}\mathbf b_r\\\mathbf b_m\end{pmatrix} =
\begin{pmatrix}\mathbf b_{r,0}\\\mathbf b_{m,0}\end{pmatrix} -
\begin{pmatrix}
\mathbf H_{rr} & \mathbf H_{rm}\\
\mathbf H_{rm}^T & \mathbf H_{mm}
\end{pmatrix}
\begin{pmatrix}\Delta\mathbf x_r\\ \Delta\mathbf x_m\end{pmatrix}
\end{equation}
$$

同样通过舒尔补的操作，得到$\mathbf b_r$的表达式：
$$
\begin{equation}
-(\mathbf H_{rr}- \mathbf H_{rm}\mathbf H_{mm}^{-1}\mathbf H_{rm}^T) \Delta\mathbf x_r = \mathbf b_r - \mathbf b_{r,0} - \mathbf H_{rm}\mathbf H_{mm}^{-1}(\mathbf b_m - \mathbf b_{m,0})
\end{equation}
$$

转换一下
$$
\begin{equation}
\mathbf b_r = \mathbf b_{r,0} + \mathbf H_{rm}\mathbf H_{mm}^{-1}(\mathbf b_m - \mathbf b_{m,0}) - (\mathbf H_{rr}- \mathbf H_{rm}\mathbf H_{mm}^{-1}\mathbf H_{rm}^T) \Delta\mathbf x_r
\end{equation}
$$

代入$\eqref{marg_remian}$最终得到
$$
\begin{equation}
(\mathbf H_{rr}- \mathbf H_{rm}\mathbf H_{mm}^{-1}\mathbf H_{rm}^T)\delta\mathbf x_r = \mathbf b_{r,0} -\mathbf H_{rm}\mathbf H_{mm}^{-1} \mathbf b_{m,0} - (\mathbf H_{rr}- \mathbf H_{rm}\mathbf H_{mm}^{-1}\mathbf H_{rm}^T) \Delta\mathbf x_r
\end{equation}
$$

记作
$$
\begin{equation}
\bar{\mathbf H} \delta\mathbf x_r = \bar{\mathbf b} - \bar{\mathbf H} \Delta\mathbf x_r
\end{equation}
$$

这里的$\bar{\mathbf H}$可以分解为$\bar{\mathbf H} = \bar{\mathbf J}^T\bar{\mathbf J} = \mathbf U\mathbf S\mathbf U^T$，把上式转换为$\mathbf J^T\mathbf J\mathbf x=-\mathbf J^T\mathbf e$的形式，有：

$$
\begin{equation}
\bar{\mathbf J}^T\bar{\mathbf J} \delta\mathbf x_r = -\bar{\mathbf J}^T(-\bar{\mathbf J}^{T+}\bar{\mathbf b} + \bar{\mathbf J} \Delta\mathbf x_r)
\end{equation}
$$

得到上式之后，我们就可以把边缘化后的状态构成一个代价方程，也就可以作为先验约束加入到之后的优化当中。需要注意，在看VINS-Mono中的实现，可能会觉得是不是少了一个负号？仔细对照后，会发现在VINS代码中计算边缘化factor时使用$\mathbf b = \mathbf J^T\mathbf e$这样的表达式，因此在边缘化中计算初始残差的时候，负号就抵消了。

在上式中，我们需要知道雅可比$\mathbf J$和对应的残差$\mathbf e$，然后才可以计算我们的未知量$\mathbf x$(我们在优化中，这个都是我们所求参数上的增量)。

我们可以通过对$\bar{\mathbf H}$矩阵进行分解得到雅可比矩阵。考虑到$\bar{\mathbf H} = \bar{\mathbf J}^T\bar{\mathbf J} = \mathbf U\mathbf S\mathbf U^T$，我们有
$$
\begin{equation}
\bar{\mathbf J}^T = \mathbf U \mathbf S^{\frac{1}{2}}\\
\bar{\mathbf J} = \mathbf S^{\frac{1}{2}}\mathbf U^T
\end{equation}
$$

在得到雅可比矩阵后，接下来在求残差中涉及到一个求伪逆的操作。参考MVG V2的A5.2节，对于$m\times n$的矩阵$\mathbf A$，$m\geq n$，有SVD分解$\mathbf A=\mathbf U\mathbf D\mathbf V^T$，$\mathbf A$的伪逆可这样计算：
$$
\begin{equation}
\mathbf A^+ = \mathbf V\mathbf D^+\mathbf U^T
\end{equation}
$$

这里的$\mathbf D^+=\text{diag}(\sigma_1^{-1},...\sigma_r^{-1},0,...,0)$是$n\times m$阶对称矩阵。

对应于求伪逆的方法，我们可以得到
$$
\begin{equation}
\bar{\mathbf J}^{T+} = \mathbf S^{-\frac{1}{2}}\mathbf U^T
\end{equation}
$$

----------

  1. [On-Manifold Preintegration for Real-Time Visual-Inertial Odometry](http://rpg.ifi.uzh.ch/docs/TRO16_forster.pdf)
  2. [Visual-Inertial Monocular SLAM with Map Reuse](http://webdiis.unizar.es/~raulmur/MurTardosRAL17.pdf)
  3. [OKVIS笔记](https://fzheng.me/2018/03/22/okvis-marginalization-base/)
