---
name: awesome-remotion
description: 使用 React 创建 Remotion 视频项目的最佳实践。适用于创建、搭建或重构多场景 Remotion 视频项目。提供项目结构约定、使用 Sequence+map 的场景组合模式、时长管理最佳实践以及整洁的 Root.tsx 组织方式。
---

# Awesome Remotion

创建多场景 Remotion 视频项目的最佳实践。

## 何时使用此技能

当用户想要以下操作时使用此技能：
- 从零创建新的 Remotion 视频项目
- 为现有 Remotion 项目添加场景或进行重构
- 使用合理的结构组织多场景视频
- 搭建带动画的场景组合

## 项目结构约定

所有 Remotion 视频项目遵循以下结构：

```
src/
├── index.ts              # 入口文件：registerRoot(RemotionRoot)
├── Root.tsx              # 仅包含 Composition 定义（保持整洁，不留模板残留）
├── index.css             # 全局样式（Inter 字体 + 等宽字体）
└── <VideoName>/          # 场景组件目录（以视频名称命名）
    ├── index.tsx         # 主视频组件（编排所有场景的序列）
    ├── Opening.tsx       # 场景 1 组件
    ├── SceneTwo.tsx      # 场景 2 组件
    ├── SceneThree.tsx    # 场景 3 组件
    └── ...
```

核心原则：
- **场景组件**统一放在以视频名称命名的目录中
- **主视频组件**（`index.tsx`）是编排器——负责将所有场景按序列排布
- **Root.tsx** 仅包含 `<Composition>` 定义——不留任何模板残留代码
- 每个场景组件**自包含**，拥有独立的动画逻辑

## Root.tsx 模式

Root.tsx 应保持最简和整洁。仅在此定义 Composition：

```tsx
// src/Root.tsx
import "./index.css";
import { Composition } from "remotion";
import { MyVideo, TOTAL_DURATION } from "./MyVideo";

export const RemotionRoot: React.FC = () => {
  return (
    <>
      <Composition
        id="MyVideo"
        component={MyVideo}
        durationInFrames={TOTAL_DURATION}
        fps={30}
        width={1920}
        height={1080}
      />
    </>
  );
};
```

Root.tsx 的规则：
- 导入主视频组件及其导出的 `TOTAL_DURATION`
- 每个视频对应一个 Composition（完整视频）
- 不要包含模板中残留的 HelloWorld / Logo Composition
- 除非明确需要调试，否则不要包含单独的场景 Composition

## 主视频组件模式（场景目录内的 index.tsx）

主视频组件使用**基于数组的 `<Sequence>` 映射**来实现整洁的场景编排：

```tsx
// src/MyVideo/index.tsx
import { AbsoluteFill, Sequence } from "remotion";
import { Opening } from "./Opening";
import { SceneTwo } from "./SceneTwo";
import { SceneThree } from "./SceneThree";

const FPS = 30;

// 集中式时长配置 - 唯一事实来源
const SCENES = [
  // [场景名称,    组件,         起始帧,        帧时长]
  ["Opening",     Opening,      0,            5 * FPS],     // 0s - 5s
  ["SceneTwo",    SceneTwo,     5 * FPS,      7 * FPS],     // 5s - 12s
  ["SceneThree",  SceneThree,   12 * FPS,     10 * FPS],    // 12s - 22s
] as const;

// 导出总时长，供 Root.tsx 中的 Composition 使用
export const TOTAL_DURATION = SCENES.reduce((sum, [, , , dur]) => sum + dur, 0);

export const MyVideo: React.FC = () => {
  return (
    <AbsoluteFill style={{ backgroundColor: "#0f172a" }}>
      {SCENES.map(([name, Component, from, duration]) => (
        <Sequence key={name} from={from} durationInFrames={duration}>
          <Component />
        </Sequence>
      ))}
    </AbsoluteFill>
  );
};
```

为何推荐此模式：
- **基于数组的 `.map()`** 比手动书写每个 `<Sequence>` 并计算偏移更整洁
- **`SCENES` 数组**是时序的唯一事实来源
- **`TOTAL_DURATION` 导出**避免组件与 Composition 之间的数值不匹配
- **添加/删除/重排场景**只需编辑数组中的一行
- **每个场景的内部帧计数从 0 开始**（Sequence 负责处理偏移）

## 场景组件模式

每个场景组件是独立的 React 组件，专注于自身的内容和动画：

```tsx
// src/MyVideo/Opening.tsx
import {
  AbsoluteFill,
  useCurrentFrame,
  useVideoConfig,
  interpolate,
  spring,
} from "remotion";

export const Opening: React.FC = () => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();

  // 所有动画基于此场景内从 0 开始的 frame
  const fadeIn = interpolate(frame, [0, 20], [0, 1], {
    extrapolateLeft: "clamp",
    extrapolateRight: "clamp",
  });

  // ... 更多动画 ...

  return (
    <AbsoluteFill style={{ backgroundColor: "#0f172a" }}>
      {/* 场景内容 */}
    </AbsoluteFill>
  );
};
```

场景组件的关键规则：
- 始终使用 `useCurrentFrame()`——帧 0 = 此场景的开始（由 Sequence 保证）
- 始终用 `<AbsoluteFill>` 包裹内容
- 在 AbsoluteFill 中设置背景色（确保不会出现透明间隙）
- 保持每个场景自包含——不要引用其他场景的状态
- 使用 `spring()` 实现自然运动，使用 `interpolate()` 实现线性/定时过渡

## 动画速查表

各场景中常用的动画模式：

### 滑入（从左/从右）
```tsx
const slideProgress = spring({ frame: frame - delay, fps, config: { damping: 12 } });
const xPosition = interpolate(slideProgress, [0, 1], [-500, 0]); // 从左侧滑入
```

### 缩放弹入
```tsx
const scaleProgress = spring({ frame: frame - delay, fps, config: { damping: 8 } };
// 通过 transform 应用：`scale(${scaleProgress})`
```

### 淡入
```tsx
const opacity = interpolate(frame, [startFrame, endFrame], [0, 1], {
  extrapolateLeft: "clamp",
  extrapolateRight: "clamp",
});
```

### 文字逐行出现
```tsx
// 逐行动画，依次出现
const line1Opacity = interpolate(frame, [10, 30], [0, 1], { clamp: true });
const line2Opacity = interpolate(frame, [35, 55], [0, 1], { clamp: true });
```

## 全局 CSS 模板

```css
/* src/index.css */
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800;900&family=Fira+Code:wght@400;500;600;700&family=JetBrains+Mono:wght@400;500;600;700&display=swap');

* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'SF Pro Display', sans-serif;
  background-color: #0f172a;
  color: #f8fafc;
}

code, pre, .monospace {
  font-family: 'Fira Code', 'JetBrains Mono', 'Cascadia Code', monospace;
}
```

## 常用配色方案（深色主题视频）

| 标记 | 色值 | 用途 |
|------|------|------|
| 背景色 | `#0f172a` | 主背景 |
| 表面色 | `#1e293b` | 卡片、面板 |
| 边框色 | `#334155` | 面板边框 |
| 主文字 | `#f8fafc` | 标题 |
| 次文字 | `#94a3b8` | 正文 |
| 弱化文字 | `#64748b` | 提示、标签 |
| 强调紫色 | `#c084fc` | @ 作用域、命名空间 |
| 强调黄色 | `#fbbf24` | 高亮、警告 |
| 强调粉色 | `#f472b6` | 版本标签 |
| 强调绿色 | `#4ade80` | 命令、成功 |
| 强调红色 | `#CB3837` | npm 品牌 |
| Python 蓝 | `#387EB8` | Python 品牌 |

## 工作流：创建新视频

1. 在 `src/<VideoName>/` 下创建场景目录
2. 编写各场景组件（`.tsx`）
3. 创建 `src/<VideoName>/index.tsx` 作为编排器，包含 `SCENES` 数组 + `TOTAL_DURATION` 导出
4. 更新 `src/Root.tsx`，添加指向主视频组件的新 `<Composition>`
5. 运行 `npm run dev` 在 Studio 中预览
6. 在 Studio 侧边栏点击 Composition 预览完整视频时间线

## 竖屏短视频排版经验（1080×1920）

面向抖音、微信视频号等短视频平台，观众主要用手机观看时的排版最佳实践。

### 核心原则

1. **字号要占屏幕宽度的 5-10%** — 1080px 宽的屏幕，标题至少 68px+，正文至少 44px+，最小字号不低于 36px
2. **间距不超过字号的 1.5 倍** — 宁可稍挤，也不能上下留白一片
3. **内容靠上不居中** — `justifyContent: "flex-start"` + 顶部大 padding，比居中留白更饱满
4. **撑满宽度** — 卡片/终端等大块元素宽度接近 1000px（屏幕 1080），两侧只留 40px 边距
5. **圆角和边框要粗** — 3-4px 边框、24-28px 圆角，在小屏幕上才够醒目
6. **纵向堆叠为主** — 横向并排改上下排列，2×2 网格替代横排

### 推荐字号体系

| 用途 | 变量名 | 推荐值 | 占屏幕宽度比 |
|------|--------|--------|-------------|
| 开场大标题 | `hero` | **80px** | 7.4% |
| 场景标题 | `title` | **68px** | 6.3% |
| 副标题 | `subtitle` | **56px** | 5.2% |
| 正文 | `body` | **44px** | 4.1% |
| 标签 | `label` | **40px** | 3.7% |
| 最小文字/注释 | `caption` | **36px** | 3.3% |
| 等宽代码 | `mono` | **36px** | 3.3% |
| 大数字 | `counter` | **112px** | 10.4% |

> 原则：最小字号不低于 36px，手机上再小就看不清了。

### 推荐间距体系

| 用途 | 变量名 | 推荐值 | 说明 |
|------|--------|--------|------|
| 区块间隔 | `section` | **36px** | 场景内标题与内容之间，控制留白的关键 |
| 大间距 | `xl` | **48px** | 大区块之间 |
| 中间距 | `lg` | **36px** | 段落之间 |
| 基础间距 | `md` | **28px** | 行间距 |
| 小间距 | `sm` | **20px** | 标签间距 |
| 最小间距 | `xs` | **12px** | 图标与文字间距 |

> 间距要克制——竖屏空间宝贵，留白太多会让内容稀疏，观众觉得"没什么东西"。

### 布局模式

#### 场景通用结构
```
┌──────────────────────┐
│  顶部安全区 100-120px │  ← paddingTop，不要用 justifyContent: center
│  ─────────────────── │
│  场景标题 (68px)      │
│  间距 36px            │
│  ─────────────────── │
│  内容区               │
│  （撑满宽度，左右40px）│
│                       │
│  标签/卡片等          │
│                       │
│  底部文字/注释        │
│  底部安全区 80-100px   │
└──────────────────────┘
```

#### 对比布局（左右分屏）
- 左右各占约 48%，中间留 4% 给分隔线/VS 标志
- 每侧内部纵向堆叠
- 字号不缩减，保持 44px+ 正文
- VS 圆圈/标志 160×160px

#### 网格布局
- 用 2×2 网格替代横向 4 列排列
- 每个格子至少 480×240
- 图标容器 240×240px

### 终端/代码窗口样式

```tsx
// 终端窗口撑满宽度
width: 1000,  // 屏幕 1080，两侧各留 40px
borderRadius: 28,
border: `3px solid ${COLORS.border}`,

// 终端标题栏
height: 56,
borderBottom: `2px solid ${COLORS.border}`,

// 代码字号
fontSize: 36,  // 不要再小了
lineHeight: 1.6,
padding: 32,
```

### 横屏 → 竖屏迁移清单

| 参数 | 横屏 (1920×1080) | 竖屏 (1080×1920) | 变化 |
|------|-------------------|-------------------|------|
| 画面比例 | 16:9 | **9:16** | 旋转 90° |
| 标题字号 | 52px | **68px** | +31% |
| 正文字号 | 36px | **44px** | +22% |
| 最小字号 | 28px | **36px** | +29% |
| 区块间距 | 56px | **36px** | -36% |
| 卡片宽度 | 880px | **1000px** | 撑满 |
| 布局方式 | 横向并排 | **纵向堆叠** | — |
| 对齐方式 | `center` 居中 | **`flex-start` + 顶部 padding** | 去掉居中留白 |
| 圆角 | 20px | **28px** | 更醒目 |
| 边框 | 2px | **3-4px** | 更醒目 |

### 常见错误

1. **用 `justifyContent: "center"` 居中** → 上下大片空白，内容看起来稀疏。改为 `flex-start` + 顶部 padding
2. **字号照搬横屏** → 手机上根本看不清。最小 36px，标题 68px+
3. **间距过大** → `gap: 64` 在竖屏上浪费太多空间。区块间距 36px 足够
4. **横向排列 3+ 元素** → 宽度不够，元素挤成一条。改 2×2 网格或纵向堆叠
5. **卡片不撑满宽度** → 880px 卡片在 1080 屏幕上左右各空 100px，浪费空间。改为 1000px

## 渲染

渲染最终视频：

```bash
# 通过 Studio UI：点击 Composition 上的 "Render" 按钮
# 或通过 CLI：
npx remotion render src/index.ts MyVideo out/video.mp4
```
