# Mesh2Motion 调研落地报告

> 调研时间：2026-05-07
> 官网：https://mesh2motion.org/
> 在线体验：https://app.mesh2motion.org/

---

## 一、项目概述

Mesh2Motion 是一个**开源 3D 动画库 + 自动绑定工具**，定位为 **Web 版 Mixamo 替代品**。

### 核心能力
- 上传任意 3D 网格模型 → 自动绑定骨骼 → 应用动画 → 导出 GLB
- 完全运行在浏览器中（Three.js）
- 代码 MIT 协议，资产 CC0 协议

### 仓库信息
| 仓库 | Stars | Forks | 语言 |
|------|-------|-------|------|
| [Mesh2Motion/mesh2motion-app](https://github.com/Mesh2Motion/mesh2motion-app) | 2,521 | 224 | TypeScript |
| [Mesh2Motion/mesh2motion-assets](https://github.com/Mesh2Motion/mesh2motion-assets) | 45 | 4 | Python/Blender |

---

## 二、核心技术架构

### 2.1 技术栈
- **Three.js r183** — 3D 渲染和动画
- **Vite** — 构建工具
- **TypeScript** — 全部源码
- **纯前端** — 无后端依赖

### 2.2 骨骼自动绑定算法

核心文件：`src/lib/solvers/SkinningAlgorithm.ts` + `WeightCalculator.ts`

**三步 pipeline：**

```
WeightCalculator          WeightSmoother              WeightNormalizer
(找到每个顶点        →   (边界权重平滑)            →   (归一化到1.0)
  最近的骨骼)              + HeadWeightCorrector
```

**WeightCalculator 算法（最近中点法）：**
1. 对每个骨骼，计算它与**第一个子骨骼**的中点位置
2. 对每个顶点，遍历所有骨骼中点，找最近的骨骼
3. 顶点 100% 归属最近骨骼（单骨骼权重）
4. 特殊处理：人类骨盆区域 — 低于盆骨高度的点忽略盆骨归属，防止腿部落回盆骨

**WeightSmoother（分类感知平滑）：**

| 骨骼分类 | 示例 | 平滑策略 |
|---------|------|---------|
| Torso | spine, chest, neck | 宽域多环渐变（3 rings，向邻居扩散） |
| Limb | upperarm, forearm, thigh, calf | 方向性向子骨骼单向平滑 |
| Extremity | hand, foot, finger, toe | 最小化单环 50/50 |
| Other | root, head | 默认单环 |

### 2.3 骨骼重定向（Retargeting）

**HumanChainConfig.ts** — 硬编码骨骼映射表，支持：

```typescript
mesh2motion_config   // 标准 Mix2Motion 骨骼命名
mixamo_config        // Mixamo 标准命名
```

支持导入**预绑定模型**（已绑定骨骼的 GLB）进行动画重定向。

支持预置骨骼自动映射器（Mixamo / Rigify / Mesh2Motion / 自定义映射）。

### 2.4 三种使用模式

| 模式 | 说明 |
|------|------|
| **Explore** | 直接浏览 150+ 预置动画（Human 124个，Fox 14个等），预览和下载 |
| **Use Your Model** | 上传自己的网格模型 → 自动骨骼绑定 → 选择骨骼类型 → 预览动画 → 导出 |
| **Use Your Rigged Model** | 上传已有的 rigged+skinned GLB → 骨骼映射 → 重定向动画 → 导出 |

---

## 三、已支持的骨骼类型

`src/lib/enums/SkeletonType.ts`：

- Human（人类）
- Human (A-Pose)
- Fox（狐狸）
- Bird（鸟）
- Dragon（龙）
- Kaiju（怪兽）
- Spider（蜘蛛）
- Snake（蛇）

每种骨骼类型对应独立的 rig 文件和动画集合。

---

## 四、动画库规模

**实测数据（下载验证）：**

- `human-base-animations.glb` — 5.8 MB，含 **87 个动画**，每个动画 **198 个 channels**（完整人体骨骼动画）
- 动画命名风格：Mixamo 风格 + 游戏实用动作（如 Sword_Attack, Farm_Harvest, Zombie_Scratch, Spell_Simple 等）
- Human 总计 **124 个动画**（base + addon 分包）

---

## 五、落地验证结果

### 5.1 在线体验
- ✅ 访问 https://app.mesh2motion.org 成功
- ✅ Explore 页面正常加载，3D 预览正常
- ✅ Fox 模型 14 个动画正常预览
- ✅ Human 模型 124 个动画正常预览和播放

### 5.2 资源下载验证
- ✅ 模型：`https://app.mesh2motion.org/models/model-human.glb` — 428 KB，GLB v2 有效
- ✅ 动画包：`https://app.mesh2motion.org/animations/human-base-animations.glb` — 5.8 MB，含 87 个有效动画

### 5.3 本地运行能力
- ✅ Node.js v24.13.1 + npm 11.8.0 已具备
- ✅ `npm install && npm run dev` 可本地启动
- ⚠️ 由于是纯前端应用，WebGL 渲染依赖浏览器环境

### 5.4 算法质量评估

**优点：**
- 骨骼分类合理（Torso/Limb/Extremity），避免肢体关节处动画变形
- 盆骨区域特殊处理是正确思路，防止腿部落回盆骨
- 权重分配明确（1-bone initial），便于后续平滑处理
- 重定向架构完整，支持多骨骼系统映射

**局限：**
- WeightCalculator 是纯最近距离算法（k-NN, k=1），没有考虑几何形状语义
- 无 ML/AI 成分，纯几何算法
- 没有"穿衣服"场景的处理（单一网格）
- 无实时物理/碰撞

---

## 六、与 Mixamo 对比

| 维度 | Mixamo | Mesh2Motion |
|------|--------|-------------|
| 运行平台 | Adobe 云服务 | 纯浏览器 |
| 开源 | ❌ 闭源 | ✅ MIT |
| 骨骼自动绑定 | ✅ | ✅ |
| 动画数量 | 更多 | ~150+ |
| 自定义模型 | ✅ | ✅ |
| 导出格式 | FBX 等 | GLB/GLTF |
| 服务器依赖 | 需要联网 | 完全离线可用 |

---

## 七、应用场景分析

### 7.1 适合的场景
- 🎮 **游戏开发** — 快速给 3D 角色绑定骨骼和应用动画（尤其独立游戏）
- 🌐 **Web 3D 应用** — 游戏化教育、虚拟展厅、交互式故事
- 🐴 **动物/怪兽动画** — Mixamo 没有的骨骼类型（Spider, Kaiju, Snake）
- 🔧 **动画资产生产** — Blender 制作源文件 + Mesh2Motion 批量导出游戏引擎可用 GLB

### 7.2 不适合的场景
- 需要 AI 生成动画（没有此能力）
- 高精度影视级绑定（需要 Maya/MotionBuilder）
- 复杂多角色交互动画
- 物理模拟类动画（布料、头发）

---

## 八、可落地性评估

### 综合评分：★★★★☆（4/5）

| 维度 | 评分 | 说明 |
|------|------|------|
| 功能完整性 | ⭐⭐⭐⭐ | 骨骼绑定+重定向+动画库完整 |
| 动画质量 | ⭐⭐⭐⭐ | 基于 Blender 专业绑定，质量可靠 |
| 部署难度 | ⭐⭐⭐⭐⭐ | 纯前端，零部署，直接 `npm run dev` |
| 开源可控 | ⭐⭐⭐⭐ | MIT 协议，代码可修改 |
| 动画数量 | ⭐⭐⭐ | ~150够用，但比 Mixamo 少 |
| 自动化潜力 | ⭐⭐⭐⭐ | CLI 可集成，API 中转站可用 |

### 落地建议

1. **直接使用在线版**：`https://app.mesh2motion.org` 作为动画资产管理工具
2. **本地部署**：fork `mesh2motion-app`，自己部署，添加更多骨骼类型
3. **动画资产采集**：批量下载 GLB 动画文件，集成到现有视频/游戏管线
4. **技术借鉴**：SkinningAlgorithm 的分类平滑逻辑可迁移到其他 3D 工具

---

## 九、参考资料

- 官网：https://mesh2motion.org/
- 在线体验：https://app.mesh2motion.org/
- GitHub：https://github.com/Mesh2Motion/
- Discord：https://discord.gg/UChE936q7y
- 支持/捐助：https://support.mesh2motion.org/
