# 把语言模型装进机器人

> 不是教你怎么训 VLA 的书。是 LLM 装进具身之后，按下 deploy 之前要做的几十个判断的书。

📖 在线阅读: **<https://yingwang.github.io/robotics-llm-book/>**

---

## 这是什么

一本中文长篇技法书，写给 2024-2026 这一波具身智能浪潮里三种人：

1. 知道 transformer 怎么训、看得懂 RT-2 那张图，但不知道下一步该往哪学；
2. 做了多年传统机器人，公司转型要做具身大模型，不知道该用 LeRobot 还是自己写训练循环，不知道 teleop 多少条数据够用；
3. **懂点 ML，机器人完全是新领域**。看 demo video 看得激动，想看懂这些公司到底在赌什么、未来五年值不值得跳进去。生词遇到跳过去也能跟上每章的论点。

全书 14 章，每章约 5000 字，目标是把"端到端 vs 分层"这一把判断尺压到 14 个具体场景上：感知、动作、抓取、导航、长程任务、仿真到现实、数据、评估、安全、硬件、部署、团队与资本、五年后。

每章带场景钩子、公开 demo 引用、点名的 paper / 公司 / 年份 / 数字、判断表、和 3-4 道不写代码的练习。

## 立场

序里立了 6 条反对，整本书贯穿到底：

- 反 **端到端必胜论** - 长程、回退、结构化推理，VLA 没解
- 反 **人形必然论** - 仓库里轮式 + 单臂赢双足十倍
- 反 **仿真已死论** - 视觉 gap 缩了，接触动力学没缩
- 反 **demo 即真理** - 拆 5 种 cherry-pick 手法
- 反 **teleop 无限好** - 1k/10k/100k 三个工业阈值，超过 10k 该分层
- 反 **alignment = 具身安全** - 行为/软件/硬件三类要分开

## 目录

序：[这本书的定位](docs/00-preface.md)

**第一部分 框架变了**
1. [端到端与分层](docs/01-end-to-end.md)
2. [感知](docs/02-perception.md)
3. [动作](docs/03-action.md)

**第二部分 难题**
4. [抓取](docs/04-manipulation.md)
5. [导航](docs/05-navigation.md)
6. [长程任务](docs/06-long-horizon.md)

**第三部分 工程**
7. [仿真到现实](docs/07-sim2real.md)
8. [数据](docs/08-data.md)
9. [评估](docs/09-evaluation.md)
10. [安全](docs/10-safety.md)

**第四部分 落地**
11. [硬件](docs/11-hardware.md)
12. [部署](docs/12-deployment.md)
13. [团队与资本](docs/13-team-and-capital.md)
14. [五年后](docs/14-five-years.md)

## 本地构建

```bash
pip install mkdocs-material
mkdocs serve   # 本地预览 http://127.0.0.1:8000
mkdocs gh-deploy --force   # 推到 gh-pages 分支
```

## License

文字部分 CC BY-NC-SA 4.0，欢迎在署名 + 非商用 + 相同许可下转载和翻译。
