告别 GPU 依赖的 SLAM? 在 CPU 上实现动态场景下超速跟踪定位的 NGD-SLAM！
==========================================

Original 唐僧洗头用飘柔 [深蓝 AI]

**深蓝 AI** 

Weixin ID shenlanxueyuan

About Feature 专注于人工智能、机器人与自动驾驶的学习平台。

_2024-07-03 11:48_ _北京_

前言🪐

本项工作的作者为来自曼彻斯特大学的 Yuhao Zhang（章昱昊）同学，【深蓝 AI】对这篇优秀 paper 进行了编译首发，相关 demo 可移步 bilibili 章同学的账号@-不会修电脑 - 查看。

论文标题：

NGD-SLAM: Towards Real-Time SLAM for Dynamic Environments without GPU

论文作者：

Yuhao Zhang

项目地址：

https://github.com/yuhaozhang7/NGD-SLAM

编译：唐僧洗头用飘柔

审核：_Los_

> **导读：**
> 
> 本研究提出了一种无需 GPU 的新型视觉 SLAM 系统——NGD-SLAM，专为动态环境设计，实现了在 CPU 上的实时性能。通过创新的掩码预测机制，系统允许深度学习识别动态对象与相机跟踪并行运行，显著提升了效率。结合双阶段跟踪策略，NGD-SLAM 在保持高定位精度的同时，达到了 56fps 的跟踪速率。实验结果证明了该系统在动态环境中的有效性，展示了即使在硬件资源受限的情况下，深度学习在 SLAM 应用中仍然具潜力。©️【深蓝 AI】编译

## Abstract🌟

动态环境下摄像机定位跟踪的精确性和鲁棒性是视觉 SLAM 的重大难题。当前解决该问题的一大方向是通过深度学习技术为动态对象生成掩码，这通常需要 GPU 实时运行（30 fps）。基于此，本文提出了一种新的动态环境下的视觉 SLAM 系统，该系统通过结合掩模预测机制在 CPU 上获得实时性能，使深度学习方法和摄像机跟踪在不同频率下完全并行运行，无需等待对方的结果。并且本研究进一步引入了双级光流跟踪方法，将光流和 ORB 特征混合使用，显著提高了系统的效率和鲁棒性。与目前最先进的方法相比，该系统在动态环境中保持了较高的定位精度，同时在单个笔记本电脑 CPU 上实现了 56 fps 的跟踪帧率，无需任何硬件加速，从而证明了深度学习方法在没有 GPU 支持的情况下仍然可以实现动态 SLAM。据现有信息，这是第一个实现这一目标的 SLAM 系统。

**本文的主要贡献如下：**

> ●NGD-SLAM 方法：提出了一种名为 NGD-SLAM 的新型视觉 SLAM 系统，专为动态环境设计，能够在没有 GPU 支持的情况下，在 CPU 上实现实时性能；
> 
>   ●掩码预测机制：开发了一个框架独立的掩码预测机制，允许深度学习模型和相机跟踪并行运行，提高了系统的运行效率；
> 
> ●双阶段跟踪方法：引入了一种双阶段跟踪方法用于分别跟踪静态与动态关键点；
> 
>   ●混合策略：结合了光流跟踪与 ORB 特征跟踪的优势，探索其在动态环境下的有效性。

## NGD-SLAM

**■2.1 NGD-SLAM 基本框架**  

![Image](https://mmbiz.qpic.cn/mmbiz_png/Nabxc8rdYrjkFrnPibaTsFCbp3fjwz6uPB2I9cKYEwoBeibrHpM1LZuF6FlEuMibQFBKZGATakUmGb6iaolIep8Fhw/640?wx_fmt=png&from=appmsg&random=0.9689914801215049&random=0.5432325523511388&random=0.7070475917768391&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)▲图 1｜NGD-SLAM 流程图©️【深蓝 AI】编译

图 1 展示了 NGD-SLAM 的运作流程。首先，将第一帧的图像数据转发给语义线程，并将该帧设置为关键帧。图像的 ORB 特征是通过一个掩码从语义线程中提取出来的，以过滤掉动态特征；并且只有第一帧会等待语义线程给出的掩码；随后系统初始化地图作为关键帧跟踪的一部分。对于后续帧，系统采用双阶段跟踪，跟踪潜在的动态对象，根据掩码预测机制为其生成掩码，跟踪最后一帧剩余的静态点，估计相机姿态。然后决定设置一个新的关键帧。如果需要一个新的关键帧，系统通过过滤掉动态跟踪过程生成的掩码中的特征，从当前帧中提取静态 ORB 特征，并使用 ORB 特征继续关键帧跟踪过程。

**■2.2 语义线程**  

NGD-SLAM 的语义线程基于 YOLO-fastest 模型构建，用于检测动态对象。这个线程使用一个输入和输出缓冲区，每个缓冲区保存一个帧，其中一个新的输入或输出帧将替换缓冲区中现有的帧。从跟踪线程传递到输入缓冲区的帧由网络处理，然后移动到输出缓冲区供跟踪线程访问。这种设置确保语义线程处理最近的输入帧，而跟踪线程不必等待语义线程的结果，而是直接从缓冲区中获取最新的输出。虽然两个线程因运行频率不同而存在时间不匹配的问题，既从输出缓冲区中获取的结果对应于以前的一个帧，NGD-SLAM 通过使用过去的结果来预测当前帧中动态对象的掩码来补偿这一点。

**■2.3 掩码预测**  

NGD-SLAM 的一个重要模块是掩码预测机制，这也是动态跟踪的基础组成部分。与深度神经网络的前向过程相比，该预测方式效率更高、可以直接适应任何可视化 SLAM 框架，并保持其实时性能。图 2 显示了掩码预测的流程，它由六大部分组成：检测、分割、抽样、跟踪、聚类及预测，其中检测和分割在语义线程内部进行，跟踪线程利用检测和分割结果预测当前帧中的掩码。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Nabxc8rdYrjkFrnPibaTsFCbp3fjwz6uPIZZpd1l807V03zXkZmabrwmVKL3iaic5oGrr8QVasA49lbe0urXsCicJg/640?wx_fmt=png&from=appmsg&random=0.9131718022035089&random=0.7103938066433275&random=0.995035599363254&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)▲图 2｜掩码预测流程图©️【深蓝 AI】编译

_**◆检测：**_  

在语义线程中使用 YOLO-fastest 模型检测帧中的多个对象，并根据语义信息识别潜在的动态对象。NGD-SLAM 采用此模型以提高效率，但也可以根据需求与其他模型轻松替换以进行对象检测或分割。

_**◆分割：**_  

使用目标检测模型比分割网络更有效，但它仅为检测到的对象提供边界框。为了改善这一点，NGD-SLAM 使用深度信息来分割这些框内的物体。在识别动态对象（如人）之后，然后对每个边界框中的像素深度进行排序，使用中位数作为检测到的对象的估计深度。边界框被展开，所有接近这个深度值的像素被屏蔽。由于计算效率高，这种方法优于聚类方法。随后，使用连通分量标注算法排除深度相似但不属于主对象的对象的掩码，并通过扩展过程进一步细化掩码，然后将其传递到输出缓冲区。

_**◆抽样：**_ 

给定已识别动态对象的掩码，在掩码内提取动态点。图像被分成大小为 15 × 15 的单元格。在每个单元格内，识别被遮挡且满足 FAST 关键点标准的像素，并选择阈值最高的像素。如果没有合适的 FAST 关键点，则在该单元格中选择一个随机像素。这种网格结构允许动态关键点的均匀提取，并且由于关键点的独特性及其与高梯度区域的关联，保证了跟踪的稳定性。

_**◆跟踪：**_  

该系统采用 Lucas-Kanade 光流法来跟踪采样的动态点，该方法假设一个关键点的灰度值在不同帧之间保持不变。数学上表示为：

$$
I(x+d x, y+d y, t+d t)=I(x, y, t)
$$

式中$I(x,y,t)$为像素在坐标 (x, y) 处随时间 t 的灰度强度值。这个方程可以用泰勒级数进一步展开为：

$$
I(x+d x, y+d y, t+d t) \approx I(x, y, t)+\frac{\partial I}{\partial x} \cdot d x+\frac{\partial I}{\partial y} \cdot d y+\frac{\partial I}{\partial t} \cdot d t
$$

随后，假设像素在一个小窗口内均匀移动，可以形成一个优化问题，该问题利用窗口内的像素梯度通过最小化匹配像素的总灰度误差来估计像素运动（dx, dy）。

_**◆聚类：**_  

对于当前帧中跟踪的动态点，DBSCAN (Density-Based Spatial Clustering of Applications with Noise) 算法应用于彼此接近的聚类点，同时将明显远离任何聚类的点区分为噪声。采用这种方法可以有效地区分来自不同动态对象的跟踪点。此外，通过考虑每个跟踪点的二维位置和深度，该算法有效地排除了异常点，例如明显偏离聚类的点或在二维图像中看起来很近但深度差异明显的点。

_**◆预测：**_  

在得到聚类后，系统意识到动态实体在当前帧内的位置。因此，它最初为每个簇生成一个粗糙的遮罩，通常是一个矩形，以覆盖其中的所有点。然后，利用每个聚类中点的深度信息，将相应的掩模细化为精确的动态目标形状。如图 2 所示，为当前帧预测的掩码，效果十分精确。

**■2.4 双模跟踪**  

_**◆动态跟踪：**_  

![Image](https://mmbiz.qpic.cn/mmbiz_png/Nabxc8rdYrjkFrnPibaTsFCbp3fjwz6uPHTl3X1OB7fBqKFL1PP9mNyKHCGmVYTfpjqjY8v0WubAQvlL5CG2AYw/640?wx_fmt=png&from=appmsg&random=0.6751404689211598&random=0.1665306186419635&random=0.45220687634767787&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)▲图 3｜动态对象跟踪©️【深蓝 AI】编译

虽然掩模预测机制已经是一种动态跟踪，但它对 YOLO 的依赖有时会导致动态目标跟踪失败。例如，当相机旋转时，YOLO 可能很难检测到物体。因此，当语义线程无法提供用于跟踪的掩码时，NGDSLAM 提供了一个直接的解决方案：它使用前一帧的预测掩码来预测当前帧的掩码。这种方法允许对动态点进行连续跟踪，直到动态对象在相机中不再可见，在深度神经网络遇到问题的下情况下也能保持跟踪。该方法通过不断地从前一个掩模中采样新的点进行跟踪，而不是跨多个帧跟踪相同的点，确保了关键点保持最新并增强了跟踪的鲁棒性，具体实现效果如图 3 所示（在深度学习模型失效时，系统也能在 30 帧内持续追踪动态对象）。而替代方法涉及从相机姿态旋转部分的四元数表达式（w,x,y,z）中导出滚动值，使用：

$$
roll =\arctan 2\left(2 \times(w \times x+y \times z), 1-2 \times\left(x^2+y^2\right)\right) \times \frac{180}{\pi}
$$

并使用该值旋转图像进行检测，操作示意如图 4 所示，动态跟踪的目的是在一般情况下提供鲁棒跟踪，因为相机旋转并不是导致检测失败的唯一因素。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Nabxc8rdYrjkFrnPibaTsFCbp3fjwz6uPhuyTJqeu6X71TGAmoIIZvrN18A0PcibOhRj5EKR1q7oEwy3WbUUjkjA/640?wx_fmt=png&from=appmsg&random=0.5249956564323297&random=0.49969302145143524&random=0.7216576588247929&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)▲图 4｜旋转图像©️【深蓝 AI】编译

_**◆静态跟踪：**_  

与原始 ORB-SLAM3 依赖于每个输入帧的 ORB 特征匹配不同，NGD-SLAM 使用光流方法从最后一帧开始跟踪静态关键点。将上一帧的三维地图点与当前帧中跟踪的二维关键点链接形成几何约束，并根据以下运动模型计算当前帧的初始位姿：

$$
T_t=\left(T_{t-1} T_{t-2}{ }^{-1}\right) T_{t-1}
$$

其中是上一帧的位姿。基于这些信息，应用 RANSAC(Random Sample Consensus) 算法迭代估计实际姿态并滤除异常值。这种设计基于两个考虑因素：首先，当当前帧中出现一个动态物体但在用于预测的帧中没有出现时，掩码预测机制通常会失败。因此，提取新的 ORB 特征并基于每帧的描述符值差异进行匹配可能会导致动态点之间的不匹配或匹配。

_（具体演示可见图 5，在这些帧中，由于动态对象新出现，掩码预测失败。第一行提取和匹配每一帧的 ORB 特征，得到与动态点的匹配；第二行利用光流法跨多帧跟踪相同的静态点）_

相比之下，静态跟踪过程往往更具鲁棒性，该过程基于图像梯度和滤除异常值来跨多帧跟踪相同的静态点。尽管将为关键帧跟踪提取新的 ORB 特性，但当没有足够的跟踪静态点时，就会发生这种情况，这表明动态对象在多个帧中出现了一段时间。其次，静态跟踪过程跳过了计算代价高昂的非关键帧 ORB 特征提取，使跟踪过程更加高效。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Nabxc8rdYrjkFrnPibaTsFCbp3fjwz6uP0QFskSL99EMMTpBMv7LAleKBBwWZ6MFvlvprvUNWOqRHA9tneurMtA/640?wx_fmt=png&from=appmsg&random=0.7914590687378666&random=0.9452225736750968&random=0.06497500644590715&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)▲图 5｜静态点跟踪©️【深蓝 AI】编译

**■2.5 关键帧跟踪**  

关键帧保留 ORB 特性以建立它们之间的连接。它允许维护基本的 ORB-SLAM3 组件，如共同可见的图形和本地地图，这对于稳健的跟踪和有效的数据检索至关重要。当静态跟踪表现不佳时，它也可以作为备用跟踪方法，因为它对照明变化和运动模糊更健壮。关键帧跟踪是利用静态跟踪阶段估计的姿态，将最后一个关键帧的地图点投影到当前帧中，得到匹配并进行优化。与原来的 ORB-SLAM3 类似，添加新的地图点，并将帧传递给本地映射线程进行进一步优化。

## 实验效果

本文选择了动态环境下最先进的 SLAM 算法和其他几种基线方法进行比较，包括 DynaSLAM、DS-SLAM、CFPSLAM、RDS-SLAM 和 TeteSLAM。无双模跟踪 (仅保留掩膜预测) 的 NGD-SLAM 也用于消融实验。本研究利用了 TUM 数据集中的四个序列，这些序列捕捉了两个个体在桌子周围移动的场景，其中一个人穿着丰富纹理的格子衬衫。这些序列展示了多种相机运动，包括沿 xyz 轴的平移运动、旋转运动（显著的横滚、俯仰、偏航变化）、类似半球的轨迹，以及几乎静止的轨迹。为了保证准确性，选择了 ATE（绝对轨迹误差）和 RPE（相对轨迹误差）作为评估指标，其中 RPE 测量的间隔设置为 1 秒（30 帧）。为了评估效率，测量了处理给定函数所需的时间（以毫秒为单位）。

**■3.1 准确性分析**  

表 1 给出了 NGD-SLAM 与不同基线方法关于 ATE 均方根误差的比较。最小误差值被突出显示，并在旁边显示相应的等级 (除了没有 DST 的 NGD-SLAM，它表示与基本实现相比误差是增加还是减少)。很明显，DynaSLAM、CFP-SLAM 和 NGD-SLAM 在所有序列中都具有很高的精度。然而，DynaSLAM 通常给出稍高的误差，而 CFP-SLAM 在 f3/w\_rpy 序列中的表现不那么令人印象深刻，因为它们采用的极极约束在旋转场景中不太有效。此外，表 2 和表 3 显示了 RPE RMSE 比较，其中 NGD-SLAM 继续保持最先进的结果。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Nabxc8rdYrjkFrnPibaTsFCbp3fjwz6uP95DHukdtRxIIJ5b2j1lQU3ibE4eFhYMlic8HloByicvDbF0SnvEVVfdWg/640?wx_fmt=png&from=appmsg&random=0.18741156359318345&random=0.3037521605394402&random=0.6622825682366027&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)▲表 1｜绝对轨迹均方根误差比较（单位 m）©️【深蓝 AI】编译

![Image](https://mmbiz.qpic.cn/mmbiz_png/Nabxc8rdYrjkFrnPibaTsFCbp3fjwz6uPLGCn1lPgT96tju4qibzRlrpbgMVczpazIg9kCUC7Yr1wC5xZWjVrHcA/640?wx_fmt=png&from=appmsg&random=0.482357810086804&random=0.06472422771565922&random=0.5824453067579198&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)▲表 2｜平移相对位姿均方根误差比较（单位 m/s）©️【深蓝 AI】编译

![Image](https://mmbiz.qpic.cn/mmbiz_png/Nabxc8rdYrjkFrnPibaTsFCbp3fjwz6uP5cBicIuwLeJVBtMXsHpjuEeZbmsEw7UqVpvxTuLSJYpib32GDicYcmLBQ/640?wx_fmt=png&from=appmsg&random=0.5860722970847314&random=0.7704731935789535&random=0.6890118778454768&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)▲表 3｜旋转相对位姿均方根误差比较（单位°/s）©️【深蓝 AI】编译  

在上述结果中还可见双级跟踪在这里起着至关重要的作用，因为当双级跟踪被禁用时，系统的 ATE 和 RPE 通常都会增加。该现象在 f3/w\_rpy 和 f3/w\_half 等更具挑战性的序列中尤为明显。

**■3.2 效率分析**

表 4 给出了这些动态序列中每帧平均处理时间的比较。由于一些算法不是开源的，并且在各种设备上进行了测试，因此通过显示算法的每帧平均处理时间与原始 ORB-SLAM2/3 的平均处理时间的比率来确保公平的比较。ORB-SLAM2/3 的平均跟踪时间约为 20 毫秒;因此，在 1.65 以内的值可以认为是实时性能。虽然 DynaSLAM 和 CFP-SLAM 显示出高精度，但即使在 GPU 支持下也无法实现实时性能。RDS-SLAM 和 TeteSLAM 是实时运行的，但由于它们仅将深度学习模型应用于关键帧而降低了准确性。相比之下，NGD-SLAM 和没有 DST(只保持掩码预测) 的 NGD-SLAM 都实现了实时性能，且无需使用 GPU。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Nabxc8rdYrjkFrnPibaTsFCbp3fjwz6uPxic1Kdflz9miaAx8icaQcN4Iic3ia2nm62hibmE4SKH4BylzCLDRGXdUpKWQ/640?wx_fmt=png&from=appmsg&random=0.20193479897455924&random=0.3056866656651118&random=0.9403654140258808&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)▲表 4｜每帧平均跟踪时间比较©️【深蓝 AI】编译

**■3.3 实时性分析**

系统每帧的平均处理时间为 17.88 毫秒 (56 fps)，这归功于非关键帧的光流跟踪效率。由于只有关键帧的跟踪涉及到 NGD-SLAM 的所有组件，因此该研究还进一步对实验进行分段，以评估使用静态、动态和关键帧跟踪方法跟踪关键帧所需的平均时间，如表 5 所示。可见，采用光流的静态跟踪十分高效，尽管动态跟踪所需时间稍长，但合并后的跟踪过程在 33.3 毫秒 (30 fps) 以内，仍然满足实时性的要求。值得注意的是，虽然 YOLO 检测每帧大约需要 30 到 40 毫秒，但由于跟踪线程和语义线程以不同的频率并发操作，因此没有将其考虑到跟踪时间中。

![Image](https://mmbiz.qpic.cn/mmbiz_png/Nabxc8rdYrjkFrnPibaTsFCbp3fjwz6uPCBYXwMpejQ7dbSXlZ3YVexOFgPCSeSc9fg4bvLklNE3FBbj1PKzibDA/640?wx_fmt=png&from=appmsg&random=0.6617227852690557&random=0.768063344009144&random=0.6234036531816212&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)▲表 5｜NGD-SLAM 每关键帧平均跟踪时间（单位 ms）©️【深蓝 AI】编译

## 结论  

本研究介绍了一种基于 CPU 的动态环境下的实时可视化 SLAM 系统。它结合了一个框架独立的掩码预测机制，以减轻由于使用深度学习模型而导致的低效率，同时保持其在动态对象识别中的高精度。此外，为了弥补掩模预测机制的局限性，进一步提高系统的效率，提出了一种双级跟踪方法。在动态环境数据集上的实验评估以及与基线方法的比较表明，NGD-SLAM 可以达到与顶级算法相当的精度，并且无需任何硬件加速即可在单个笔记本电脑 CPU 上获得实时跟踪。这一进展强调了基于深度学习的方法在动态环境中提高 SLAM 系统有效性的潜力，即使没有 GPU 的支持。

**【邀请函】**

【深蓝 AI】开放授权分享通道，向广大【AI+ 自动驾驶 + 机器人】领域的实验室及个人征稿授权，期冀提供一个更方便读者与原作进行沟通交流的平台，也希望能促成更多有意义有价值的合作。

如果你有意通过【深蓝 AI】，向更多的人分享自己的最新工作，请点击如下推文了解详情👇

[![Image](https://mmbiz.qpic.cn/mmbiz_jpg/Nabxc8rdYrgDIeLDYoyNxDG7YHVSA6IXx6LoyPKEMW73PFv5Kn5LqLVArmHUnAEVl6iaUv4SNDc7wZdqOb7ehbw/640?wx_fmt=jpeg&from=appmsg&random=0.9931484592386159&random=0.1943335251229643&random=0.7056156362129191&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)](http://mp.weixin.qq.com/s?__biz=MzU2NjU3OTc5NA==&mid=2247584602&idx=2&sn=6fc4896b24753cfddf360c32880e401e&chksm=fca98c67cbde0571345e0779bf08afc97eee798fb3a2b082d7618f52b7829eec61eb4bde3bb1&scene=21#wechat_redirect)