# 第 5 章 导航（地图死了吗？）

> 这一波最容易被误解的事是"foundation model 替代 SLAM"。没替代。它**包了一层语义皮**，让原来一张白茫茫的占据栅格图能听懂"去厨房"，仅此而已。下面那一层 ICP、loop closure、graph optimization 一个没省。

---

2024 年夏天我在一个亚马逊仓库式的客户现场看一台轮式底盘做盘点。它跑的栈很标准：前后两颗 RealSense D455、一颗 Velodyne 16 线、IMU 是 Bosch BMI088、Cartographer 跑 2D LIDAR SLAM、Nav2 做局部规划。

它在主货架区里转得非常稳。一进到货架尽头那一片**用作打包区缓冲的金属反光支架前面**，姿态估计开始抖。LIDAR 在那一面上回波多重反射，激光打出去回来像走了一段不存在的距离。Cartographer 内部的 scan match score 开始报黄。视觉那边 ORB 特征点在反光金属上漂得更厉害，每帧抓到的角点跟上一帧根本不在一个地方。IMU 先漏漂大概十秒，姿态就拐到墙里去了。

机器人当时在做的事是去某个货位扫一个箱子。它的内部坐标里那个货位现在在墙后面。它毫不犹豫地朝墙走，撞上之后 costmap 才把那一格涂红，重规划，绕一下又撞，三次之后全局重定位 trigger，把自己拉回来，前后掉了 40 秒。

这个故事在 2026 年仍然每天在工厂、仓库、医院里发生几千次。**不是因为没人懂 SLAM，是因为环境总有 SLAM 假设崩掉的那 5%**。这一章讲的就是新一波 foundation 视觉特征和 VLN 怎么处理那 5%，以及它们处理不了的另外几个 5% 还得交回给老栈。

写这一章的同一周里我还看了另一个对照。一台家庭服务机器人，纯视觉栈，跑在一栋瑞典老式公寓楼里。它在客厅里几乎完美。问题出在客厅通往卧室那条**长 4 米、两侧白墙、地板纹理一致**的走廊上。当时跑的是一个基于 DINOv2 patch feature 的 retrieval 定位。在走廊中间它的 retrieval top-1 命中了**一张完全不在这栋公寓里的图像**，是训练数据里另一户人家的同款走廊。机器人非常自信地按那张图的语境往前走，撞到了一扇训练数据里那户人家有、这户人家没有的门把手。

这件事工程上叫 hallucination。术语借自 LLM。**"foundation 视觉特征在低纹理重复结构里会幻觉"是 2024 - 2025 年家用机器人公司学到的最贵的一课**，每家几乎都付过这个学费。

---

先把经典栈是什么说清楚，不是为了讲历史，是因为后面每一段都要回头引它。

**视觉 SLAM 这一线**。Mur-Artal 的 ORB-SLAM 系列（ORB-SLAM、ORB-SLAM2、ORB-SLAM3）从 2015 到 2020 把 monocular / stereo / RGB-D / inertial 全打通了。它的核心是 ORB 特征点 + bag-of-words 做 loop closure + g2o 做 graph optimization。这一套到现在仍然是学术 baseline，也是很多消费级 AR 设备里跑的东西。同一时期还有 LSD-SLAM、DSO 走 direct method，特征点稀疏但稠密深度好。

**LIDAR SLAM 这一线**。Google 的 Cartographer（Hess 等，2016）把 2D LIDAR + IMU 做到了仓库级稳定。3D 那边走 LOAM 那一支，到 LIO-SAM（Shan 等，2020）把 IMU 紧耦合做出来之后，Hilti 比赛连续几年它都拿奖。再往后是 FAST-LIO2、Point-LIO 这些极致优化版本。**工业部署里 LIDAR SLAM 这一支几乎是默认**，因为它对光照、纹理、动态都比纯视觉抗造。

**融合**。IMU 做 short-term 高频姿态，wheel odometry 做地面平移先验，GPS / RTK 做户外全局 anchor，LIDAR / 视觉做长期回环。EKF / UKF / factor graph 把它们焊到一起。Maybeck 那本 *Stochastic Models, Estimation, and Control* 是这一行的圣经，但实际工程里 ROS 那个 robot_localization 包就够用。

**地图表示**。occupancy grid（Moravec & Elfes，1985）到现在还是最常见的栅格地图。costmap_2d 在 ROS 里把静态层、障碍层、膨胀层叠起来，让规划器拿到一张可以直接 A*/Dijkstra 的地图。3D 那边 OctoMap 主导了很多年，最近 NeRF / Gaussian Splatting 这一线开始挤进来当 dense 表示。

**规划栈**。MoveBase 是 ROS1 时代的事实标准，Nav2 是 ROS2 重写版（Macenski 等，2020 年起），把 behavior tree 当顶层调度，下面 plug 不同的 planner（NavFn、SmacPlanner）和 controller（DWB、TEB、MPPI）。**到 2026 年 Nav2 仍然是绝大多数轮式平台部署的栈**，没有哪个 foundation model 替代了它。

把这一段记住。下面讲新东西的时候每一句都要对着这一段问一句"这件事是真的替代了上面哪一行，还是只是叠了一层"。

---

新东西是什么。

**Foundation 视觉特征**。DINOv2（Oquab 等，Meta，2023）和它的后续把 ViT 在大规模无标注图像上 self-supervised 训出来的 patch feature 给开源了。这个 feature 的好处是**对光照、视角、风格变化抗造**，而且语义信息密度比 ResNet 那一代高一个数量级。直接的影响是：原来视觉 place recognition 用 NetVLAD 那一套，现在拿 DINOv2 patch feature 平均池化一下就能比它好。原来视觉里程计在低纹理墙面失效，现在 DINOv2 patch 在白墙上仍然能 match。SuperPoint + SuperGlue（Sarlin 等，2020）那一支也还在演进，LightGlue（2023）把推理速度拉到实时。

**Open-vocabulary semantic mapping**。这是过去三年最热的一线。

- **CLIP-Fields**（Shafiullah、Pinto，2022）把 CLIP feature 烧进一个隐式神经场，让你能用自然语言查询场景里的位置。"sink" 这个词进去，出来场景里水池所在的 3D 位置。
- **OpenScene**（Peng 等，CVPR 2023）把每个 3D 点的 feature 跟 CLIP 文本 embedding 对齐，在 ScanNet 上做 zero-shot 3D 分割。
- **ConceptGraphs**（Gu 等，2024）走得更远，把场景拆成 object node，每个 node 带 CLIP feature 和文本描述，graph 上跑 LLM query。"找一个能装下我这只杯子的容器"这种问题它能在 graph 上推。
- **OK-Robot**（Liu 等，Meta + NYU，2024）把 ConceptGraphs 那一套接到了一台真机上，跑了 pick-and-place。**这是第一个让人觉得"open-vocab 语义地图"在家里可以真用"的工作**。
- **HOV-SG**（Werby 等，2024）把分层场景图带进来，区域、房间、物体三层，规划器可以在不同层做粗细不一的 query。

这一线里要点名的另一个是 **VLMaps**（Huang 等，2023）。这个工作把 CLIP feature 直接 project 到 2D top-down 地图上，让你能在 occupancy grid 同样的栅格里查"沙发在哪"。它的工程直接，部署简单，到 2025 年很多公司内部地图都长这个样子。

注意这些工作没有一个**替换了 SLAM**。它们都建立在一个已经 SLAM 出来的几何 backbone 上，往上叠语义。下层那一坨 LIDAR + Cartographer + factor graph 没有任何变化。**这是这一波最容易被外面人误解的事**。

要再补一个最近的趋势：**Gaussian Splatting** 在 2024-2025 年挤进了导航这个圈子。3D Gaussian Splatting（Kerbl 等，SIGGRAPH 2023）原本是 graphics 里做新视角合成的工作，但它的副产品是一个**稠密、可微、能实时渲染的 3D 表示**。SplaTAM（Keetha 等，CVPR 2024）、Gaussian-SLAM、MonoGS 把 3DGS 跟 SLAM 焊在一起，让你边跑 SLAM 边建一个 photorealistic 的稠密地图。这件事在仿真闭环（拿真实 splat 当 sim 环境）和远程巡检（远端工程师戴 VR 头盔走进机器人扫出来的 splat 里看现场）上都开始有产品。**但作为定位 backbone，它仍然不够 robust**，跑长了会漂，目前还是叠在传统 SLAM 之上当 dense layer。

---

VLN 这一支是另一回事。

Vision-Language Navigation 的 setup 是：人给一句自然语言（"走出这个房间，往左拐，进第二个有钢琴的房间"），机器人要按这句话从 A 走到 B。这个 task 在学术上是 R2R（Anderson 等，2018）、REVERIE（Qi 等，2020）、RxR（Ku 等，2020）、VLN-CE（Krantz 等，连续动作版，2020）这几个 benchmark。

跑 VLN 的环境主要三个：

- **Habitat**（Savva 等，Facebook AI，2019 起到现在 Habitat 3.0）。3D 重建场景里跑 photorealistic agent。
- **Matterport3D**（Chang 等，2017）。90 个真实建筑扫描，绝大多数 VLN benchmark 的底层数据。
- **ScanNet**（Dai 等，2017）。1500+ 室内场景的 RGB-D 扫描，更偏 3D understanding。

VLN 这条线学术上做了七年，**真实部署到 2025 年才开始有像样的工作**。原因是 sim 里的视觉 feature 跟真实 RGB 差得太多，sim2real gap 比抓取那边大得多。

而真实部署这一线的转折点是 **GNM → ViNT → NoMaD**，Sergey Levine 那个组在 Berkeley 推的：

- **GNM**（General Navigation Model，Shah 等，2022）。在多个机器人平台、多个场景的 6000+ 小时数据上训一个 goal-conditioned policy，输入当前图像 + 目标图像，输出 action。
- **ViNT**（Visual Navigation Transformer，2023）。把 backbone 换成 transformer，做了 image-goal 的 zero-shot 跨平台泛化。
- **NoMaD**（Navigation with goal-Masked Diffusion，2024）。用 diffusion policy 把 exploration 和 goal-reaching 统一到一个网络里。

这一线的卖点是**不需要 metric map**。给它一张目标图像，它从当前帧一步步走过去。它内部有一种隐式拓扑，但不显式建图。

值得再单独强调：GNM/ViNT/NoMaD **不是 VLN**。它接受的是 image goal 不是语言 goal。这件事 Levine 那个组很坦白，他们认为图像目标比文本目标 robust，因为图像消除了语言里的歧义（"那个红色的椅子"指哪一把）。要把语言塞进来需要再叠一层 VLM 把句子翻成 sub-goal 图像，**这一层在 2025 年才开始有像样的工作**（CoW、LM-Nav 这一线）。

VLN sim2real 这件事还要单挑一句。R2R / VLN-CE 这种 benchmark 在 Habitat 里跑出来的 SR（success rate）能到 60-70%，听起来很好。但拉到真机上，两年前能到 20% 就算优秀，现在最好的工作能到 40%。这个 gap 不是模型不够强，是 **Habitat 里的视觉跟真实摄像头采到的 RGB 在颜色分布、模糊、运动伪影上差太多**，在 sim 上 train 出来的 policy 视觉那一头基本要重训。这件事第 7 章会讲怎么办。

---

讲到这里就到了第一个要立的态度。

**度量地图（metric map）vs 拓扑地图（topological map）**这件事在传统机器人圈讨论了三十年。度量地图给你精确坐标，拓扑地图只给你节点和边。Kuipers 在 90 年代的 Spatial Semantic Hierarchy 把分层这件事讲透了，但工业上一直跑度量，因为度量好做精确规划。

LLM 时代拓扑地图回潮，原因有三个。

第一，**LLM 天然在节点-边的图上推理得很好**，给它一张精确坐标系它反而不知道怎么用。第二，**foundation 视觉特征让"两张图是不是同一个地点"这个判断不再需要精确几何 match**，纯靠 feature 距离就能判，于是拓扑地图的节点合并问题第一次有了 robust 的解。第三，**长程任务里坐标误差累积是要命的**，但拓扑边只关心"能不能从 A 到 B"，对累积误差不敏感。

ConceptGraphs、HOV-SG 这一线本质上都是分层图（scene graph），就是拓扑地图加上语义 node 属性。**2026 年的趋势是：底层 metric SLAM 仍然跑，但上层规划越来越多在拓扑/语义图上做**。

这是一个本来就该这么发展的方向。LLM 只是把它催熟了。

---

下面是这一章的几个硬立场。

**立场一：LIDAR 在户外和工业仍然是默认，VLM/foundation 视觉在户外最多当辅助**。

这一条 2026 年仍然没有反例。Waymo 的 robotaxi、Boston Dynamics Spot、Nuro、Agility Digit 在工厂部署的版本，**全部跑 LIDAR 主导的 SLAM/定位栈**。视觉那一块负责语义（行人、标牌、车道线），但 ego pose 不靠它。原因不复杂：户外光照变化太大、距离太远、动态太多，视觉的 metric 精度撑不住。LIDAR 给你 ±5 cm 的距离测量是物理保证。

如果你做的是户外或半户外（园区、停车场、码头）任务，**第一选择仍然是 LIDAR**。要省钱可以选固态 LIDAR（Livox、Hesai 都到了几千美金一颗），但别想着用 4 颗 RGB 摄像头加 DINOv2 替掉它。能跑 demo，跑不了部署。

**立场二：家用场景里纯视觉能接管 80%，剩下 20% 死在低光、重复走廊、玻璃墙**。

家里这件事跟户外刚好相反。家里大部分场景是**强纹理、近距离、光照可控、动态少**，纯视觉栈能跑得很好。1X、Figure、Apptronik 这几家面向家用 / 商用室内的人形，视觉占比在不断提高。Figure 的 Helix 公开 demo 看不到 LIDAR。1X NEO 在家庭场景里走的是相机阵列方案。

但纯视觉在三个家庭子场景里会一直死下去：

- **晚上灯关掉之后**。RGB 摄像头在 < 5 lux 下信号噪声比崩掉，DINOv2 那一套也帮不了你。这种场景要么加红外，要么加一颗低成本 LIDAR，要么干脆告诉用户晚上别让机器人巡逻。
- **重复走廊和重复墙面**。公寓楼里两条几乎一模一样的过道，纯视觉的 place recognition 会 confuse。这种地方要么用 WiFi RSSI / UWB anchor 帮一下，要么加一颗 LIDAR 让几何形状打破 ambiguity。
- **大块玻璃墙**。视觉看到的是玻璃后面的东西，机器人会以为路通的。这件事 LIDAR 也死，因为 905 nm 激光会穿透很多家用玻璃。**目前最 robust 的方案是加超声波或者 ToF camera 在底盘前段做近距离禁区**。

家用部署的 lesson：**至少留一颗超声波或一颗低线数 LIDAR 做兜底**。整机算一颗 RPLIDAR A1 加几颗超声波也就 100 美金不到，省下来的客诉远不止这个数。

再补一条工程上的脏经验：**WiFi RSSI 指纹定位在公寓楼里比想象中好用**。Google 自家的 indoor location 那一套用了十几年。家里路由器一般固定不动，BSSID 列表本身是一个稳定的"我在哪个房间"的指纹。家用机器人厂商不愿意承认在用，但翻过几家 firmware 之后会发现这件事被默默叠在视觉栈下面当 prior。它不能给你 cm 级精度，但能在视觉漂的时候告诉你"你应该在客厅而不是卧室"，这个 prior 一加，重定位的搜索空间就缩小一个数量级。

**立场三：SLAM 没死，只是被外面包了语义皮**。

这是这一章最想钉死的一句。

外面一些公司的 marketing 会说"我们用 foundation model 直接做 navigation，不需要 SLAM"。**99% 的情况下是因为他们的 sim 里地面是平的、光照恒定、不需要长期定位**。一旦真机跑过 30 分钟以上、跨房间、有地毯有门槛，SLAM 那一坨该跑的还是要跑。

OK-Robot 的论文你仔细看，下面跑的是 ORB-SLAM3（论文里写得很清楚）。ConceptGraphs 跑实验的时候用的是预先扫好的 mesh，扫 mesh 那一步就是 SLAM。VLMaps 也假设有 metric 地图作为 backbone。

新一波语义地图是叠在 SLAM 之上的一层 representation。LLM/VLM 进来之后多了一层调用 interface，让你能用自然语言 query 这张地图。**底层那一坨 ICP、scan matching、loop closure、bundle adjustment、factor graph 一个都没省**。

这件事重要在哪？重要在你做技术选型和招人的时候。如果你信"SLAM 死了"，你会去招一队全是搞 VLM 的人，没有一个懂 g2o 怎么调。等部署到客户家发现机器人跑半小时之后位姿漂了 30 cm，没人会修。**这一行里能修 SLAM 的工程师在 2026 年比能跑 VLM 的贵**，珍惜你团队里那几个。

---

行人和动态障碍是另一个一直没解决的硬问题。

**Social navigation** 这件事，从 Trautman 在 2010 年代初的 IGP 开始就在做。最近的 benchmark 是 CrowdNav（Chen 等，2019）、SocialNav-SF（2022）、SACSoN（2023）。NoMaD 那条线也开始在数据里加行人。

到 2026 年，**这件事仍然是 open problem**。原因是：

- 行人意图是高度多模态的（要走过去 / 要停下 / 要让 / 要跟你聊天），机器人猜错代价不对称（撞到人远比绕远代价高）。
- 文化差异巨大。日本行人会主动让机器人，纽约行人会站着不动看你怎么办。一份社交数据集很难泛化。
- 评估指标没共识。"自然"是什么很难量化，跑出来好看不好看常常靠 demo video 而不是数字。

工业上大部分公司目前的做法是**保守**：检测到 5 米内有人就降速到 0.3 m/s，2 米内停下让路，3 秒之后还堵着就语音提示。这种策略不优雅但 safe。**未来两年这件事不会被 VLM 端到端解决，是会被一份够大的真实社交数据集 + 行为克隆解决**。

顺手把一个常被忽略的点说掉：**最后一米**。导航栈把机器人送到目标点附近 30-50 cm 之后，最后一米通常交回给 visual servoing 或者 manipulation 那一边的策略。这一段的失败原因跟前面整章讲的 SLAM 没关系，是物体识别和精确定位的事。但很多人在算 navigation success rate 的时候不算这一米，**论文里 SR 80% 拉到产品里能用的成功率经常掉到 50%**，差的就是这最后一米里物体没识别清楚。看 demo 的时候要专门盯着这段。

---

户外这一段拉远看一眼，作为对照。

Boston Dynamics Spot 的导航栈是公开的：前后多颗双目摄像头做近距离 obstacle，LIDAR 做远距离 mapping，IMU + leg odometry 做姿态，graph nav 做地点之间的 topological route。它的 graph nav 跟前面讲的拓扑地图思路是一致的，但里面**没有任何 LLM**。语义那一块是后来 Spot AI 那条线在叠的。

Waymo 那一套不属于本书范畴，但作为对照值得提一句：它是工业上 SLAM 做得最稳的一家，跑的是 HD map + LIDAR localization + 多传感器融合的路线，**完全不依赖 foundation model 做定位**。VLM 在里面只用作场景理解和异常检测的辅助。这件事过去十年没变，未来五年也不会变。

这两个例子要说明的事是：**外面看起来 LLM 全面接管的领域，工业最严肃的玩家都在底下保留着传统栈**。家用机器人有理由跑得激进（容错高、成本敏感），户外/驾驶没有。

农业和巡检这一段也值得提一句。Carbon Robotics、Naio、John Deere 这一线田间机器人跑的是 RTK GPS + LIDAR + 视觉 row detection 的混合栈。**RTK 在开阔田地里给你 cm 级绝对定位**，比任何 SLAM 都准。但树荫下、温室里、楼房间，RTK 信号马上崩，这时候要无缝切回 LIDAR/视觉 SLAM。**多源融合的 health monitoring 比 SLAM 算法本身在工程上更难**，工业部署里 80% 的 bug 修在融合层而不是 SLAM 层。新工程师常常觉得 SLAM 是难的部分，做过几年才知道难的是后面。

---

最后一段，关于这一章里所有判断的 meta 立场。

这一波导航的真正变化不是 SLAM 死了，是**"什么是地图"**这件事的定义变宽了。

2018 年之前地图就是 occupancy grid。2020 年加上 OctoMap 和 NeRF dense 表示。2023 年加上 CLIP-Fields 和 ConceptGraphs。2026 年的"地图"可能是一个混合物：底层 metric grid 还在，上面叠一层 dense 几何（Gaussian Splatting），再上面叠一层语义图（scene graph），最上面挂一个可以被 LLM query 的 text interface。

每一层服务一种 query。规划器查 metric。VLM 查 dense 几何。LLM planner 查 scene graph。用户查 text。**它们共存，不互相替代**。

招人的时候要找的是能在这四层都说上话的人。只懂最下面一层的工程师在 2026 年还有饭吃，但天花板被锁住了。只懂最上面那层的工程师能写漂亮 demo，但部署一上线就崩。中间那两层是大多数公司目前最缺的。

也别被 paper 标题里的"end-to-end navigation"骗了。**几乎没有一个真做产品的导航栈是端到端的**。即便是 NoMaD 这种走得最激进的工作，部署的时候也要在外面包一层避障 safety controller、一层 recovery behavior、一层 mission supervisor。第 1 章那张端到端 vs 分层判断表搬到导航这边，**90% 的 row 都倒向分层**。导航这件事跟抓取相反，时间长、误差累积、失败代价不对称（撞人/撞物比走错路严重一个量级），几乎所有特征都把指针指向分层。这是为什么这一行最资深的人对"全端到端导航"普遍冷淡。

---

## 练习

**重看一遍 OK-Robot 的 paper（Liu 等，2024）**，但只读它的 hardware section 和 implementation details，不读 method。它底下到底跑了哪些"经典"组件？数一下，然后回到这一章问自己：哪些"新"工作其实是在老栈上叠了薄薄一层？

**找一个你最熟悉的家庭/仓库机器人 demo video**，按"户外 / 室内 / 半户外"打一个标签。然后猜它用的是什么定位栈（纯视觉 / LIDAR 主导 / 融合）。如果公司公开了技术博客，对照看看猜对了没有。如果猜错了，是在哪个直觉上偏了？

**画一个你自己项目的"地图分层图"**。从下到上：metric SLAM 用什么、dense 几何要不要、scene graph 怎么建、text query 怎么接 LLM。每一层标上现在团队里谁负责。如果某一层没人负责或者所有人都说"这个我不懂"，那就是你下一个招聘点。

**找一段你公司或竞品 demo 里机器人在走廊或者家里跑的视频**，把视角对着背景的玻璃门、镜面、长走廊看，注意机器人在那几个位置有没有顿一下、抖一下、绕一下。它在那里挣扎的几秒里，**底下的栈正在告诉你它到底是纯视觉还是有 LIDAR 兜底**。

下一章：[第 6 章 长程任务](06-long-horizon.md)
