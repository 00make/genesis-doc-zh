# 🧬 为什么需要一个新的物理模拟器

与之前的模拟平台相比，这里我们强调Genesis的几个关键特性：

- 🐍 **Pythonic** 和完全透明。Genesis是用Python开发的完全开源项目，使代码理解和贡献更加容易。
- 👶 **轻松安装** 和 **极其简单** 及 **用户友好** 的API设计。
- 🚀 **并行化模拟** 具有 ***前所未有的速度***：Genesis是**世界上最快的物理引擎**，模拟速度比现有的*GPU加速*机器人模拟器（Isaac Gym/Sim/Lab、Mujoco MJX等）快***10~80倍***（是的，这有点科幻），***在模拟精度和保真度上没有任何妥协***。
- 💥 一个**统一**的框架，支持各种最先进的物理求解器，建模**广泛的材料**和物理现象。
- 📸 具有优化性能的照片级光线追踪渲染。
- 📐 **可微分性**：Genesis设计为完全兼容可微分模拟。目前，我们的MPM求解器和工具求解器是可微分的，其他求解器的可微分性将很快添加（从刚体模拟开始）。
- ☝🏻 物理精确且可微分的**触觉传感器**。
- 🌌 原生支持 ***[生成模拟](https://arxiv.org/abs/2305.10455)***，允许**语言提示的数据生成**，包括各种模态：*交互场景*、*任务建议*、*奖励*、*资产*、*角色动作*、*策略*、*轨迹*、*相机运动*、*(物理精确的)视频*等。
