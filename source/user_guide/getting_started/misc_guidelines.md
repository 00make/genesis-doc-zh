# 📝 杂项指南

（我会不断更新此文档）

- 尽可能使用 genesis.tensor。注意，当我们将 genesis tensor 传递给 taichi 内核时，调用 tensor.assert_contiguous() 检查其是否是连续的，因为 taichi 仅支持连续的外部张量。
- 不要向用户暴露任何 taichi 相关的用法。
- 添加新类时，也要实现 `__repr__()` 以便于交互式调试。（参见 genesis/engine/states.py 作为示例）
- 将所有与模拟相关的内容包含在 genesis.engine 中。
- 使用 `genesis.logger.info()`/`debug()`/`warning()` 代替 `print()`。
- 用户会被提醒不建议过多查询场景状态并且不使用它们。所有访问的场景状态将存储在场景级别的列表中，并被视为计算图的一部分。调用 `scene.reset_grad()` 时，该列表将被释放，从而释放所有占用的 GPU 内存。
- 层次结构 - 我们抽象了每一级实体创建，使它们统一且相互独立：
  - 物理求解器：我们支持各种类型：MPM、PBD、SPH、FEM、刚体等。目的是让用户灵活选择，而无需更改任何前端 API。
  - 材料 -> 这决定了后端物理求解器。我们将有 MPMLiquid、SPHLiquid、PBDLiquid、MPMElastic、FEMElastic 等。
  - 几何 -> 这定义了实体的几何形状。可以是形状原语之一，也可以是网格或 URDF 等。这些几何形状独立于使用的求解器。
  - 所有不同的实体都通过相同的 `scene.add_entity()` 添加。
- 默认求解器顺序（为了代码一致性）
  - 刚体
  - 角色
  - mpm
  - sph
  - pbd
  - fem
  - sf
- 一些顺序约定
  - 四元数：`[w, x, y, z]`
  - 欧拉角
    - 用户输入：我们使用外在 x-y-z，单位为 `度`，因为这更直观
      - 解释：我们使用 `scipy.Rotation` 的 `xyz` 顺序。
    - 内部 xyz：
      - 欧拉角在各种来源中定义不同
      - 在我们的情况下，我们使用 xyz 指代内在旋转顺序 x-y-z，与 mujoco 相同。注意，这与其他内容一致，例如 dof 力和位置。
      - 对于角速度，我们使用旋转向量。
- 我们使用 `id` 表示每个对象的 `uuid`，使用 `idx` 表示其索引
- `uv` 顺序
  - assimp, trimesh：从左下角开始
  - 我们的，pygltflib，luisa：从左上角开始
- 模拟选项与求解器选项
  - 对于在模拟和求解器选项中都存在的任何参数，求解器选项中的参数具有更高优先级，并将在未定义时使用模拟选项中的值进行初始化
  - 推荐的方法是通过模拟选项定义 dt，以便所有求解器在相同的时间尺度上运行。然而，用户也可以为不同的求解器设置不同的 dt
  - 刚体求解器在步骤级别操作，所有其他求解器在子步骤级别操作。为了使它们兼容，所有非刚体求解器在模拟选项中使用 `substeps`。
- 刚体求解器的一些设计和约定
  - 对于属性，我们使用 `*_idx`。例如 `link_info.parent_idx`
  - 对于循环迭代中的 id，我们使用 `i_*`。
  - 对于内核循环中的所有变量
    - 后缀缩写：
      - `i_l`：链接 id
      - `i_p`：父链接 id
      - `i_r`：根链接 id
      - `i_g`：几何 id
      - `i_d`：自由度
    - 对于前缀，我们使用：
      - `l_`：链接
      - `l_info`：links_info[i_l, i_b]
      - `g_`：几何
      - `p_`：父级
      - ...
  - 关于索引存储
    - 我们是在每个类中存储偏移索引（`link`、`geom` 等）还是仅存储本地索引？
      - 让我们选择前者，因为当用户查询例如一个 `entity` 时，如果显示全局链接索引会更好
      - 这适用于 `link`、`geom`、`verts` 等的索引。
    - 每个对象存储
      - 它的父类。例如 `link` 存储它的 `entity`
      - 它的全局 `idx`（偏移后）
      - 它自己子项的偏移值。例如，一个 `geom` 仅存储 `idx_offset_*` 用于 `vert`、`face` 和 `edge`。
  - 根 vs 基础
    - `root` 是一个更自然的名称，因为我们使用树结构来表示链接，而 `base` 从用户的角度来看更具信息性
    - 让我们在内部使用 `root` 链接，在文档等中使用 `base` 链接。
  - 根姿态 vs q（当前设计可能会有所变化）
    - 对于手臂和单个网格，其加载时指定的位置和欧拉角是其根姿态，q 将相对于此
  - 控制接口
    - 根位置应与第一个（固定）关节的关节位置合并，该关节连接世界和第一个链接
    - 如果我们想控制某些东西，我们将命令速度
      - 因此，如果是自由关节，可以。我们将覆盖速度
      - 如果是固定关节，则没有自由度，因此我们无法控制它
    - 如果需要位置控制，我们将编写一个 PD 控制器并在后台发送速度命令。
    - 即使是固定的，我们仍然可以更改根位置（第一个关节位置）。但这不推荐。
      - 这适用于固定和自由关节。在两种情况下都不推荐，因为即使是自由关节，设置位置也会违反物理规律。
      - 那么自由关节和固定关节有什么区别？
        - 自由度将受到外部影响，而固定关节不会
  - mjcf vs urdf
    - mjcf xml 总是有一个 worldbody，所以我们将跳过它。有时这个 worldbody 有一些相关的几何体，我们暂时不支持它。
    - urdf 只有机器人，所以我们将加载所有内容。有时机器人可以包含一个包含的世界链接，然后它将被加载到 genesis 的世界中并成为实体的根链接。
  - 碰撞处理：我们基于凸化几何体存储
    - 对于基于网格的资产，我们生成网格中存储的所有组的凸包
      - 这些组可以是原始存储的子网格，或者如果 group_by_material=True，我们将按材料分组
      - 每个组将是一个 RigidGeom
    - 对于 mjcf，我们基于 mujoco 的几何体进行凸化。每个 mj 几何体将是一个 RigidGeom
    - 对于 urdf
      - 每个 urdf 可以包含多个链接，每个链接包含多个几何体（碰撞和视觉），每个几何体将是一个原始或一个外部资产。由于 `.obj` 包含多个子网格，一个 urdf 几何体可以有多个网格
      - 我们将最低级别的网格凸化并存储为 RigidGeom
  - 控制接口设计
    - 我们不会明确有 `base pose` 的概念
      - 在 pybullet 中，可移动网格是它自己的基础链接，当被推动时其基础姿态会改变
      - 在 genesis 中
        - 所有东西都将连接到世界（链接 -1）
        - 所有东西都有根姿态。这是初始姿态，不会改变。这是我们计算 q 时使用的参考。
        - 自由移动的对象将通过具有 6 个自由度的自由关节连接到世界。当被推动时，这种状态会改变，但其根姿态保持不变。
  - 前缀 `v`
    - 这用于可视化的全局参数（视觉几何体、顶点、边、法线等）
- `surface.vis_mode`：
  - 对于刚体，支持的模式有 ['visual', 'collision', 'sdf']。默认类型是 `visual`。
  - 对于可变形非流体体，支持的模式有 ['visual', 'particle', 'recon']。默认类型是 `visual`。
    - `visual` 将渲染输入的完整视觉网格，使用内部粒子状态进行蒙皮
    - `particle` 将渲染内部粒子。如果输入纹理是颜色纹理，将使用颜色。如果是图像纹理，粒子将使用纹理的平均颜色渲染。
    - `recon` 将使用粒子进行表面重建。
  - 对于流体体，支持的模式有 ['particle', 'recon']。默认类型是 `particle`。
    - `particle` 将渲染内部粒子。如果输入纹理是颜色纹理，将使用颜色。如果是图像纹理，粒子将使用纹理的平均颜色渲染。
    - `recon` 将使用粒子进行表面重建。