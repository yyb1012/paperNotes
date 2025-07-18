
## 1 概述
### 基于溯源的入侵监测系统(PIDSes)的四个维度
1.范围：能否检测跨越应用程序边界的现代攻击？  
2.攻击无关性：能否在没有攻击特征先验知识的情况下检测新型攻击？  
3.时效性：能否在主机系统运行时高效地对其进行监控？  
4.攻击重构：能否从大型溯源图中提取攻击活动，以便系统管理员轻松理解并快速响应系统入侵？
## 2 KAIROS框架
![[Pasted image 20250709154043.png]]
### 四大组件
* [[#2.1 图构建与表示]]：以流方式分析图，按时间顺序构建图中出现的边
* [[#2.2 图学习]]：出现新边时，KAIROS根据邻域结构和节点状态对边进行重构
* [[#2.3 异常检测]]：通过构建时间窗口队列来监测异常
* [[#2.4 异常调查]]：KAIROS从异常时间窗口队列中自动生成紧凑的攻击摘要图

### 2.1 图构建与表示
![[Pasted image 20250709155626.png]]
#### 分层哈希
**1. 含义**：将系统实体的高维属性（如路径、IP 地址）转化为可被图神经网络处理的低维向量，并保留它们之间的层次语义。例如：  
`/var/log/wdev`和`/var/log/xdev`在特征空间中的映射比位于不同目录的文件`/home/admin/profile`更接近。  
**2. 作用**：节点属性变化时，无须全量重编码；层次哈希保留了这种 $结构相近 \rightarrow 向量相近$的特性 
**3. 执行过程**：
（1）首先对节点属性进行多次编码，每次编码对应不同的层级
$$ 路径名： /home/admin/clean=\left\{
\begin{matrix}
 /home \\
 /home/admin \\
 /home/admin/clean 
\end{matrix}
\right.
$$
$$ip地址： 161.116.88.72=\left\{
\begin{matrix}
 161 \\
 161.116 \\
 161.116.88 \\
 161.116.88.72
\end{matrix}
\right.
$$
（2）每个字串投影至特征空间： 
- 使用两个哈希函数： $h(s_j)$：将字符 $s_j$ 映射到特征向量的某一维。$\mathcal H(s_j)$:将字符映射+1或-1，防止哈希偏移。
某个字串s的特征向量第 $i$ 维: $\phi_i(s)=\sum_{j:h(s_j)}{\mathcal H(s_j)}$
整体特征向量是所有字串特征向量之和：$\phi(a)=\sum_{j}{\phi(s_j)}$
**4.局限性** ：分层特征哈希假设具有相似语义的两个内核实体具有相似的分层特征。虽然这在多数情况下成立，但攻击者可能会伪造路径/ip，因此不能完全信任$语义相近 \rightarrow 层次相近$这一假设
**5.解决方法**：[[#2.2 图学习]]会利用时间上下文和节点间的连接结构更新这些初始特征向量，从而尽量克服此类问题
### 2.2 图学习

|   模块    | 作用                      | **技术实现**                                                | 效果                      |
| :-----: | ----------------------- | ------------------------------------------------------- | ----------------------- |
| Encoder | 将边及其周围的图结构+时间信息转换为一个向量z | 基于 **Temporal Graph Network (TGN，2020)**，输入节点状态、邻居边和时间戳 | 捕捉图的局部行为模式，用于理解系统调用的上下文 |
| Decoder | 根据解码器的给出的z，预测边的类型       | MLP                                                     | 模拟系统在“正常”情况下应该会发生什么操作   |
|  状态更新   | 更新与边相连的两端节点的状态向量        | 使用GRU更新节点状态                                             | 记录节点的长期行为，捕捉攻击行为的演变过程   |
#### 图学习过程
[[#2.1 图构建与表示]]节点特征化仅捕获了系统实体的属性，而未考虑单个实体与溯源图中其他实体之间的任何结构关系(即一个实体与其他实体之间的交互)或时间关系(即涉及某个实体的事件序列)   
**嵌入过程**：设新边$e_t$在时间t出现在图$G_t$时，编码器基于t之前的状态($t^-$)将$e_t$嵌入到图中。即边嵌入过程中首先考虑了$G_{t^{-}}=G_t-e_t$的图特征，然后解码器再将来自于编码器的z作为输入，并预测边的类型。
**训练及测试**：训练的目标是最小化边类型与解码器预测的类型之间的差异。测试时，如果边嵌入编码的图结构类似于在相似时间上下文中从良性系统活动中观察到的结构上下文，则解码器为该边分配较小的重构误差，否则分配较大的重构误差。   
**例子**：假设训练时经常看到以下模式$$python \rightarrow read \rightarrow config.txt$$
那么在测试时，看到类似的结构比如访问其他配置文件：$$python \rightarrow read \rightarrow /etc/resolv.conf$$
二者的重构误差较小，因此认为是正常行为。
但如果出现：$$python \rightarrow exac \rightarrow /bin/sh$$
结构与之前严重不符，解码器无法准确预测，则判断为异常边。
- [I] 时序建模：TGN(2020)仅对时间戳进行拼接，对于时序建模能力较弱，可以替换为Temporal Positional Encoding、Time-aware Attention
- [I] 解码器目标设计单一：仅预测边的类型。可以拓展为多任务学习，在预测边的基础上预测节点属性、或者访问的目标是否合规
### 2.3 异常检测
#### 2.3.1 识别可疑节点
**异常性**：[[#2.2 图学习]]一节中的重构误差大于阈值
**稀有性**：如果一个节点对应的系统实体在良性执行中不频繁出现，则为稀有节点。使用逆文档频率(IDF)来计算。
#### 2.3.2 构建时间窗口队列
不能仅判断某一时刻是否异常，而是观察“**一段时间的行为演化轨迹**”
随着溯源图中出现新的时间窗口，KAIROS迭代地构建时间窗口队列。具体做法为：   
1. 对于每个窗口计算可疑节点集合
2. 如果当前窗口和已有队列的某个窗口共享可以节点，则加入该队列
3. 否则新建一个队列
#### 2.3.3 检测异常队列
- 一个时间窗口 $T$ 的异常分数是该窗口内重构误差高于阈值 $\sigma_{T}$ 的边的均值
- 一个队列 $q$ 的异常分数是队列中所有时间窗口 $T_i$ 的异常分数的乘积
- 队列异常分数高于阈值时，则被视为异常队列   

- [?] 为什么队列异常分数选择乘积而不是求和  
- 乘积可以使得***连续一致性***的重要性提高，更贴合APT等持续性攻击的行为特征
### 2.4 异常调查
从检测出的异常边中，自动生成一个结构紧凑、可读性强的攻击路径子图
- 筛选高异常边：选出具有高重构误差的边
- 构建异常图：将高异常边构成一个图 $G^*$,只保留攻击相关的节点和边
- 社区检测：使用**Louvain社区发现算法**（2008，遍历所有邻居的社区标签，并选择**最大化模块度增量**的社区标签）,对异常图进行划分，形成多个攻击阶段(社区),每个社区可能对应一次横向移动、特权提升或数据窃取。
- 形成精简攻击摘要图

##  数据集
DARPA  E3  E5

