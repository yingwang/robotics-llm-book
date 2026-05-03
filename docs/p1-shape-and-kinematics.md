# 前置 1 · 机器人长什么样、动起来怎么算

> 这一章是给从 ML 这一边过来、没怎么碰过机器人的读者的。**懂得机器人结构和运动学的可以直接跳到第 1 章**。这一章只讲后面要反复用到的基本词汇和直觉，不替代正经的 *Modern Robotics*。

---

## 形态：四五种就够用了

ML 那一边模型架构基本都是 transformer 的变种。机器人这边稍微多样：

**机械臂 (manipulator)**。一只胳膊固定在桌面或底座上，末端装夹爪或工具。Universal Robots、Franka Emika、ABB、KUKA 卖了三十年的就是这种。研究里 ALOHA 是双臂版本。

**移动底盘 (mobile base)**。轮式（差速、麦克纳姆、阿克曼）、履带、平衡车。仓储 AMR 几乎全是。Locus Robotics、AutoStore 这些公司就靠这条线吃饭。

**四足 (quadruped)**。Boston Dynamics Spot、Unitree Go2、ANYmal。腿比轮子能走的地形宽，但每一步都要平衡。

**双足人形 (humanoid)**。Tesla Optimus、Figure 02、1X NEO、Unitree H1/G1。直立行走的难度跨过四足一个数量级，原因下一节讲。

**移动操作 (mobile manipulation)**。底盘 + 一只到两只胳膊。Mobile ALOHA、Hello Robot Stretch、Apptronik Apollo 都是。家庭机器人形态战的主战场之一。

把这五种记住，全书所有"那家公司在做什么"基本都能定位。后面要讲的 VLA、teleop、sim2real，跟具体形态都有关，但不会跳出这个谱。

---

## 各形态的典型参数

光记名字不够。读 paper 看 demo 时一句"60-DoF humanoid carries 25 kg payload"要能立刻判断它在不在合理范围。下面这张表是 2026 年初典型参数的速查，价格区间是公开能查到的最低参考价（DIY 套件、研究价或量产价，括号里说明）。

| 形态 | 代表机器 | 主要 DoF | 自重 (kg) | 负载 (kg) | reach / 行走速度 | 价格区间 |
|---|---|---|---|---|---|---|
| 工业单臂 | Universal Robots UR5e | 6 | 21 | 5 | 850 mm | $25-40k |
| 协作单臂 | Franka Research 3 | 7 | 18 | 3 | 855 mm | $40-55k (研究) |
| 双臂研究 | ALOHA (双 ViperX-300s) | 6×2 | 5+5 | 0.75×2 | 750 mm | ~$32k (整套) |
| 仓储 AMR | Locus LocusBot | 0 (轮) | 100+ | 30-90 | 1.8 m/s | RaaS 30 美元/小时 |
| 四足 | Unitree Go2 | 12 | 15 | 7 | 3.7 m/s | $1.6-3.5k |
| 四足旗舰 | Boston Dynamics Spot | 12 | 32.5 | 14 | 1.6 m/s | ~$75k |
| 国产双足 | Unitree G1 | 23-29 | 35 | 3-4 | 2 m/s | $16-90k |
| 美系双足 | Figure 02 | ~30 | ~70 | 25 | 1.2 m/s | 不公开 |
| 美系双足 | Tesla Optimus Gen 3 | ~28 | ~57 | 20 | 1 m/s | 不公开 |
| 移动操作 | Mobile ALOHA | 14 + 轮 | ~50 | 0.75×2 | 1.5 m/s | ~$32k (DIY) |
| 移动操作 | Apptronik Apollo | ~30 + 轮 | ~73 | 25 | 1.5 m/s | 不公开 |

几个看表的快速 sanity check：

**自重和负载的比通常 5-15:1**。Spot 32.5 kg 抬 14 kg 已经是这一档里很激进的设计，靠的是腿强度+QDD 关节的高 torque density。Figure 02 70 kg 抬 25 kg 也是同档。这个比例直接决定能不能搬包裹、抬纸箱、拎水壶。看一台机器人的 spec 第一件事就是用这个比衡量。

**双足比四足贵一个数量级**，主要不是关节多 (双足 25-30 vs 四足 12)，是因为双足要时刻平衡，关节扭矩、传感器、控制系统都更贵。Unitree G1 把双足压到 1.6 万美元起步是国产供应链做的事，第 11 章会单独讲。

**研究价 vs 量产价 vs 不公开**。研究价是一台一台卖的小批量价，比真实成本高 30-50%。量产价是规模化之后的实际成本下界。"不公开"通常意味着公司自己也不确定 BOM 能压到哪 - Figure 和 Optimus 都在这一档，只 vague 说"长期目标 2-3 万美元"，没人押过这个数字。

---

## 自由度 (DoF)：维度数

DoF (degrees of freedom) 是机器人这一行最常用的一个词。**它就是控制空间的维度数**。

对 ML 人来说最直接的类比：你训一个网络，输出一个 K 维向量去打到电机上，K 就是 DoF。

- 一个普通的工业机械臂：6 DoF。三个平移 + 三个旋转，能让末端到达 3D 空间任意位置和姿态。
- 加上一个夹爪开合：7 DoF。
- 一只人形的整只手臂：6-7 DoF（肩 3 + 肘 1-2 + 腕 2-3）。
- 一只五指灵巧手：再加 11-24 DoF。
- 一台双足人形整体：25-35 DoF（双腿 10-12 + 双臂 12-14 + 躯干 1-3 + 头 0-2）。

**DoF 跟成本和控制难度都呈超线性关系**。原因是每一个关节都要：电机 + 减速器 + 编码器 + 走线 + 机械结构 + 控制律。35 DoF 的人形比 6 DoF 的工业臂贵不止 6 倍，控制难度差几个数量级。这是为什么第 11 章会反对人形必然论。

DoF 不等于"能做的事"。一个好的 6 DoF 机械臂能做的事比一个 30 DoF 的劣质双足多得多。**自由度多是潜力，不是成绩**。

---

## 关节、连杆、末端、底座

机器人最基本的几个词，全书反反复复用：

- **link (连杆)**：刚体段，比如大臂、小臂。
- **joint (关节)**：两个 link 之间的可动连接。最常见两类：rotational (转动) 和 prismatic (平移)。
- **base (底座)**：整个机器人的"根"。固定底座的就是工业臂；浮动底座 (floating base) 的就是双足/四足/无人机。
- **end-effector (末端执行器)**：机械臂最远端的工具，可能是夹爪、吸盘、电焊枪、画笔。
- **tool center point (TCP)**：末端上你定义"动作发生在这里"的那个点。

整个机器人在数学上是一棵 link-joint 树。固定底座是根，末端是叶。joint 的状态向量（每个关节的角度或位移）通常叫 `q`，长度等于 DoF。

---

## 正运动学 vs 逆运动学

这是必须知道的一对概念。

**正运动学 (forward kinematics, FK)**：给定 `q`（每个关节角度），算末端在世界坐标系里的位置和姿态。这是个纯几何问题，链式乘几个 4×4 齐次变换矩阵就出来了，永远有唯一解。

**逆运动学 (inverse kinematics, IK)**：给定末端目标位姿，反推每个关节该转多少。这是个反问题，**通常多解，可能无解**。一只胳膊伸到杯子前面有"上肘伸进去"和"下肘伸进去"两种方式，都能到。

IK 是经典机器人栈的核心痛点之一。多解的存在意味着"目标 6 个数 → 关节 7 个数"是欠定的，你需要再加约束（避奇异点、避自碰撞、最小关节运动）才能选出一组。这就是 MoveIt（ROS 里的运动规划库）做的事。

VLA 跟这件事的关系在第 3 章会讲。简单说，VLA 可以选择直接输出关节角（绕过 IK），也可以输出末端位姿（让下游 IK 求解器解）。两种选择各有代价。

---

## 工作空间、奇异点、关节限位

几个工程上的硬约束。

**工作空间 (workspace)**：末端能达到的所有位置的集合。一个 6 DoF 机械臂的工作空间是个有限的近球形区域，正前方臂展之外的东西它够不到。

**关节限位 (joint limits)**：每个关节的转动范围有限。人的肘关节大概 0-150 度，机器人的肘关节通常是 -120 到 +120 度这种。任何 IK 解都要落在所有关节的限位区间内。

**奇异点 (singularity)**：在某些位形下，机械臂会"瞬时失控"，几个关节的速度叠加到末端只能在某个方向上动。Jacobian 矩阵在这一点退化（秩降）。绕开奇异点是经典轨迹规划要解的事，VLA 一般靠数据本身把这种位形"学得不去"。

这些细节本书后面不会展开。但读 paper 时遇到 "singularity-avoiding"、"manipulability index"、"workspace coverage" 这些词，知道它们指什么就够。

---

## 浮动底座 vs 固定底座

最后一个概念差别。

**固定底座 (fixed base)**：机械臂底座焊在桌上、地上、AGV 上。整个系统的"根"是不动的，运动学只算手臂自己。工业机械臂、桌面 ALOHA 都是这一类。

**浮动底座 (floating base)**：双足人形、四足、无人机。整个机器人的"根"自己也在动。这种系统多了 6 个 DoF（底座本身的 3 平移 + 3 旋转），而且这 6 个 DoF 不是直接被电机驱动的，是被 footprint 跟地面的约束力推着走的。

**浮动底座是控制难度阶跃式上升的根本原因**。一个固定的 7 DoF 机械臂你可以独立控制每个关节。一个 25 DoF 的双足人形，你不能独立控制每个关节，因为如果腿乱动整个人会摔倒。所有动作都要先满足"重心还在支撑面里"这个约束才能继续。这件事专门叫 ZMP (Zero Moment Point) 控制，是双足行走研究了三十多年的核心。

ML 类比：浮动底座 ≈ 强制 hard constraint 的优化问题，固定底座 ≈ 无约束的优化问题。前者难度跨过后者一个量级。

---

## ROS 和 URDF：机器人代码生态

最后留一段给 ML 人最不熟的：机器人这一行的"PyTorch"是什么。

**ROS (Robot Operating System)**。不是操作系统，是一套消息传递框架。每个模块（感知、规划、控制）跑在自己的进程里，互相通过 topic（pub/sub）和 service（RPC）通信。ROS1 已经停更，ROS2 (Foxy / Humble / Iron / Jazzy) 是现在的事实标准。绝大多数学术机器人代码都基于 ROS2 写。

**URDF (Unified Robot Description Format)**。一个 XML 文件，描述机器人有哪些 link、哪些 joint、每个 link 的 mesh、惯性、碰撞模型。仿真器、运动规划库、可视化工具都吃 URDF。

**MoveIt**。运动规划库，做 IK + 碰撞检查 + 轨迹生成。ROS 生态里最常用的运动规划栈。

**Gazebo / Isaac Sim / MuJoCo**。仿真器。第 7 章会展开。

ML 人来到机器人这一行最大的环境冲击是：**这边不是装个 pip install torch 就能写代码的世界**。一个工程项目要装 ROS、装 driver、跑 calibration、配 URDF，调试链路远比 LLM 那边长。LeRobot (HuggingFace, 2024 起) 是 PyTorch 这一边在尝试把这件事简化的工作，目前也只覆盖了一小段。

---

## ML 栈 vs 机器人栈：一张对照表

ML 那一边一句 `pip install torch` 你就上路了。机器人这边没有这种快捷键。下面这张表把两边的对应关系列清楚，遇到一个不熟的工具往这张表上一对，大致知道它在干什么：

| ML 那边 | 机器人这边 | 区别 |
|---|---|---|
| PyTorch / TensorFlow | ROS2 + ros_control | 不是训练框架，是消息总线 + 实时控制器接口 |
| pip / conda | rosdep + apt | 系统级依赖更多，版本锁更严，跨发行版兼容差 |
| Hydra / OmegaConf | ROS2 launch.py / params yaml | 启动多节点的配置 |
| Jupyter / VSCode | RViz / Foxglove | 可视化是 3D 场景 + 多通道时序，不是 dataframe |
| TensorBoard / W&B | Foxglove / PlotJuggler / rqt_plot | 实时多通道时序，不是回合级 metric |
| HuggingFace Hub | LeRobot Hub / OXE | 模型 + 演示数据混着传，格式还在收敛 |
| dataset (parquet, jsonl) | rosbag (.bag, .mcap) | 多模态时间序列容器 |
| `model.eval()` | safety controller / e-stop | 部署时更怕错，硬件级保护是必须 |
| Slurm / Ray | ROS launch + system services | 编排是单机多进程为主，不是集群 |

LeRobot 是这两年在两边之间架桥的工作。它在 PyTorch 之上提供 dataset 标准 (LeRobotDataset 格式)、policy 训练循环、teleop 接口、和一些 baseline policy。ML 人能用熟悉的 transformers 风格代码训一个 ACT 或 diffusion policy，跑到 ALOHA / SO-100 / Koch 这几款支持的硬件上。**它的边界**：只覆盖单臂或双臂操作，不接 ROS，不做导航，不做规划。所以学完 LeRobot 你能跑 demo，但项目部署仍然要回去学 ROS。

工具栈这件事 ML 人最常踩的坑：**以为读 paper 学算法就够，实际项目里 80% 的时间花在 driver、calibration、tf2 配置、ROS topic 调试这些上**。这不是机器人这一行设计差，是物理世界本身决定的。一个机械臂跟一只 GPU 的根本区别就是前者会发热、会松螺丝、会因为电压不稳跳闸。

---

## 机器人项目第一周该做的事

如果你 ML 出身，刚被分到一个机器人组，老板说"做个 demo 出来"。**第一周不是去读 RT-2 paper**，是去做下面这五件事，做完你才有底再去读 paper。

**1. 让机器人动起来**。装 driver，写一个 hello-world 让它从 home pose（机械臂的安全初始姿态）移到一个固定的目标位姿，然后回来。这一步通常会让你撞上版本不兼容、固件升级、网络配置、力矩限制太低导致动不起来这一类问题。**这是你认识硬件最快的方式**。

**2. 录一段 ros bag**。开摄像头、开关节状态、开 IMU，让机器人随便动几下，记录 30 秒。把 bag 文件回放出来在 RViz 里看一遍。**学会 rosbag 读写比学会任何一篇 paper 重要**，因为后面所有数据采集和 debug 都基于它。

**3. 跑一次 hand-eye calibration**（手眼标定，把相机和机械臂的相对位姿算出来）。拿一个 ArUco 板或 charuco 板放在机械臂面前，机械臂走 20 个不同位姿，每个位姿拍一张照片，扔进 OpenCV 的 calibration 例程算出相机到末端的变换矩阵。**没做过这一步的人不知道为什么"视觉看到杯子在那儿，机械臂伸过去差 5 cm"是常态**。这件事经典栈每个项目第一周都要做。

**4. 把视觉、本体感觉、规划路径在 RViz 里同时画出来**。让机器人跑一个简单的轨迹，你一边看 RViz 里的 3D 场景一边看终端里的 joint state log，**学会同时盯三块屏幕和一台真机**。这是机器人 debug 的标准姿势，跟 ML 人坐在一台屏幕前看 loss 曲线完全不一样。

**5. 收一条 teleop demo**。借一个 ALOHA、Gello 或 phone-based teleop 接口（LeRobot 文档里有最简单的），让一个工程师手动遥控机器人完成一个最简单的 pick-and-place，整段录下来。回放时观察自己的"操作员风格"：在哪儿犹豫、在哪儿用力、抓得多快多稳。**这一段 demo 是你后面所有数据采集决策的基线**。

做完这五件事大概一周，你不会读任何论文。这是对的。**机器人这一行先有体感再有理论是工作的**。当你过两周回头读 RT-2 paper，每一段都会比 ML 出身的同事懂得多。

---

## 几个该有的直觉

把这一章压成几句 takeaway：

- **DoF 是控制空间的维度数**。机械臂 6-7、人形 25-35、灵巧手再加 10-20。维度多代价超线性涨。
- **运动学是几何问题，控制是动力学问题**。前者算位置，后者算力。两者都要会。
- **浮动底座 vs 固定底座是控制难度的分水岭**。
- **ROS / URDF / MoveIt 是这边的基础设施**。看 paper 时不熟可以先跳，但部署项目就绕不开。
- 最后：**机器人写代码的痛苦不在算法，在调试链路**。一个 bug 可能要重启 driver、重 calibrate、换电池、检查 ROS topic 频率。这是这一行 ML 人转过来最不适应的部分。

---

## 练习

**找一台你最熟悉的机器人** (Tesla Optimus、Unitree G1、Figure 02、随便挑)，**数它的 DoF**：双腿 + 躯干 + 双臂 + 双手 + 头，全部加起来。记住这个数。后面看 paper 提到 "32-DoF humanoid" 时你就有体感。

**读 ALOHA 的 URDF** (在 Stanford ALOHA 仓库里)。不需要懂每一行 XML，但翻看 link 树和 joint 列表，对一个真实机器人在代码里长什么样有个具象认识。

**找一个 ROS2 的 hello-world 教程** 跑一遍。哪怕只是 publisher / subscriber 来回打印一个数字。**理解 ROS 的 pub/sub 心智模型，比理解某个具体算法更影响你后面 debug 机器人项目的速度**。

下一章：[前置 2 · 机器人怎么看怎么定位](p2-perception-and-localization.md)
