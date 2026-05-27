FedID: Enhancing Federated Learning Security through Dynamic Identification `TPAMI-2025`



**核心问题**
联邦学习中，恶意客户端可以上传带后门的模型更新：正常样本预测不受明显影响，但遇到攻击者指定触发器时输出目标标签。难点是很多后门攻击会让恶意梯度看起来很像正常梯度，尤其在 non-IID 数据下，正常客户端之间本来差异也很大，传统单一统计指标很容易误判。

**FedID 的基本原理**

论文认为防御检测模型中常用的**欧式距离**在高维模型参数空间里区分力会下降，所以加入**曼哈顿距离**和**余弦相似度**，使用多个指标检测更新梯度，即 $x=(x_{Man}, x_{Eul}, x_{Cosine})$。

$x_{Man}$ 表示曼哈顿距离，$x_{Eul}$ 表示欧氏距离，$x_{Cosine}$ 表示余弦相似度。

按照以下方式计算每一轮第 $i$ 个客户端的梯度特征 $x_{Man}$、$x_{Eul}$ 和 $x_{Cosine}$：

$x_{Man}^{(i)}=\|\boldsymbol{w}_{i}-\boldsymbol{w}_{0}\|_{1}$，曼哈顿距离：捕捉高维参数中的累积偏移；
$x_{Eul}^{(i)}=\|\boldsymbol{w}_{i}-\boldsymbol{w}_{0}\|_{2}$，欧氏距离：衡量整体欧氏幅度差异；
$x_{Cosine}^{(i)}=\frac{\langle\boldsymbol{w}_{0}, \boldsymbol{w}_{i}\rangle}{\|\boldsymbol{w}_{0}\|\cdot\|\boldsymbol{w}_{i}\|}$，余弦相似度：衡量更新方向异常

其中 $\boldsymbol{w}_{0}$ 表示联邦训练前的全局模型，$\boldsymbol{w}_{i}$ 表示训练后第 $i$ 个客户端的本地模型，$\|\cdot\|_1$ 表示向量的 L1 范数，$\|\cdot\|_2$ 表示向量的 L2 范数，$\langle\cdot, \cdot\rangle$ 表示内积。

使用每个梯度与其他所有梯度之间的距离之和作为指标。

$x^{\prime{(i)}}=(\sum_{j,j\neq i}^{K}|x_{Man}^{(i)}-x_{Man}^{(j)}|, \sum_{j,j\neq i}^{K}|x_{Eul}^{(i)}-x_{Eul}^{(j)}|, \sum_{j,j\neq i}^{K}|x_{Cosine}^{(i)}-x_{Cosine}^{(j)}|) \quad (3)$

由于每个指标都是相互关联的，数值大的指标会占主导，相关性强的指标也可能被重复计算，因此应用白化过程，平衡每个指标的权重

$\delta^{(i)}=\sqrt{x^{\prime{(i)\top}}\Sigma^{-1}x^{\prime{(i)}}}.(4)$

其中 $\Sigma$ 是 $X = [x^{\prime(1)}, x^{\prime(2)}, \cdot \cdot \cdot , x^{\prime(K)}]^\top$ 的协方差矩阵。

在获得每个梯度的评分后，使用z-score来进行动态阈值设定，

$\gamma^{(i)}=\frac{\delta^{(i)}-\min_i\delta^{(i)}}{\mathrm{MAD}(\{\delta^{(i)}\}_{i=1}^K)}.\quad(7)$

在该方法中，最小的 $\delta$ 被假定为最能代表全局梯度的值，

使用绝对中位差 MAD ，让异常阈值$\gamma^{(i)}$不容易被恶意客户端本身拉偏，

$\mathrm{MAD}(\{\delta^{(i)}\}_{i=1}^{K})=\mathrm{median}\left(\left|\delta^{(i)}-\mathrm{median}(\{\delta^{(i)}\}_{i=1}^{K})\right|\right),(6) $

其中 $\gamma^{(i)}$ 代表第 $i$ 个梯度的异常程度。将 $\gamma^{(i)} \ge 1$ 的梯度视为恶意梯度，仅聚合 $\gamma^{(i)} < 1$ 的梯度。

最后对筛选出的“良性”梯度执行标准的 FedAvg 聚合。