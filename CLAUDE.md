# CLAUDE.md

本文件为 Claude Code 在此仓库中工作时提供指导。

## 项目概览

FitKit 是一个注重隐私、完全离线可用的健身伴侣应用，通过单页 HTML 实现，使用摄像头或视频进行 AI 驱动的动作形态分析。所有 AI 推理均在浏览器本地运行（TensorFlow.js + MoveNet）。

## 技术栈

- **纯静态 HTML/CSS/JS** — 无构建工具、无包管理器、无框架
- **TensorFlow.js 4.17**（CDN: `jsdelivr`）+ **MoveNet SinglePose Lightning** 姿态估计
- **Canvas API** 骨骼叠加渲染
- **localStorage** 训练记录持久化
- 国内加速镜像：`jsd.onmicrosoft.cn`（已在代码中注释备用）

## 开发方式

- 无构建流程，无 npm，无测试框架
- 直接用浏览器打开 `index.html`，或本地服务器：`python -m http.server 8080`
- `test.html` 是独立的单元测试页面，浏览器打开即可运行
- 首次使用时从 tfhub.dev 下载 MoveNet 模型（约 10MB），之后由 IndexedDB 缓存
- 国内用户可能无法直接下载模型，应用内有详细的排查指引

## 架构

### 文件结构

```
fitkit-demo/
├── index.html      # 完整应用（HTML + CSS + JS，约 1200 行）
├── test.html       # 核心生物力学逻辑单元测试
├── docs/
│   └── PRD.md      # 产品需求文档
└── README.md
```

### 应用架构（index.html）

单页应用，通过标签切换页面。使用 CSS `.page.active` 控制页面显隐。

**全局状态** 集中于 `S` 对象（约 576 行）：
- `currentPage` — 当前页面 ID
- `detector` — MoveNet 检测器实例
- `cameraStream` / `isRunning` / `repCount` — 视频分析状态
- `videoSource` — 视频来源：`camera | demo | upload | url`
- `timerRunning` / `timerRemaining` / `timerPhase` — 间隔计时状态

**页面模块：**

| 页面 | ID | 功能 |
|------|-----|------|
| 工具箱首页 | `page-home` | 功能卡片导航面板 |
| 视频分析 | `page-analysis` | 四种视频输入 + MoveNet 实时分析 |
| 间隔计时 | `page-timer` | 可自定义训练/休息间隔计时 |
| 训练记录 | `page-log` | 基于 localStorage 的训练历史 |
| 动作库 | `page-library` | 可展开的标准动作参考 |

**导航：** `navigateTo(page)` 隐藏所有页面，显示目标页。侧边栏 `.nav-item` 的点击事件触发导航。

### 姿态估计引擎

位于视频分析板块（约 752 行起）：

1. **模型加载**（`loadModel`）：设置 WebGL 后端，用 `MODEL_CONFIG` 创建 MoveNet 检测器。首次加载从 tfhub.dev 下载约 10MB 模型文件。针对常见失败场景（网络受限、WebGL 不可用）显示详细的中文错误提示。

2. **视频源** — 四种类型，由 `S.videoSource` 管理：
   - `camera` — `startCamera()` 调用 `getUserMedia`（640x480，前置摄像头）
   - `demo` — 预设的 Mixkit 视频链接（国内可能无法加载）
   - `upload` — `loadVideoFile()` 通过 File API 创建 Object URL
   - `url` — `loadVideoFromUrl()` 直接设置 video.src

3. **检测循环**（`detectPose`）：读取视频帧 → 运行检测器 → 绘制骨骼 → 执行动作分析 → 更新计数。通过 `requestAnimationFrame` 驱动。

4. **关键点常量**（`KP`）：COCO 17 关键点（NOSE 到 RIGHT_ANKLE），所有分析器共用。

5. **骨骼绘制**（`SKELETON`）：连接关键点的 12 组骨骼线。颜色编码：绿色=良好，红色=错误，蓝色=默认。

### 动作状态机（`ExerciseAnalyzers`）

每个动作都有 `analyze(keypoints, prevState)` 方法，返回分析结果对象。所有分析器共享防抖机制（`DEBOUNCE_FRAMES = 4`），避免阶段闪烁。

**深蹲分析器** — 使用双侧关键点：
- **测量指标：** 膝角、髋角、躯干前倾比、膝内扣比、髋低于膝深度检测、髋膝协调比
- **阶段：** UP → DESCENDING → BOTTOM → ASCENDING → UP
- **错误（按优先级）：** knee_valgus（严重）→ torso_lean（警告）→ shallow（警告）→ knee_first（警告）→ hip_high（警告）
- **阈值：** kneeBottom=85°、kneeTop=165°、hipAngleMin=60°、torsoMax=0.35、valgusMax=0.12

**弯举分析器** — 仅使用右侧关键点：
- **测量指标：** 肘角、肘部漂移比、肩屈曲度
- **阶段：** DOWN → CURLING → UP → RELEASING → DOWN
- **错误：** elbow_drift（警告）→ shoulder_swing（严重）→ partial_extension（警告）→ partial_flexion（警告）
- **阈值：** elbowTop=45°、elbowBottom=155°、driftMax=0.25、shoulderFlexMax=0.12

**推举分析器** — 使用右侧关键点：
- **测量指标：** 肘角、肘外展度、手腕过头
- **阶段：** UP → DESCENDING → BOTTOM → ASCENDING → UP
- **错误：** elbow_flare（严重）→ partial_lockout（警告）→ shallow_bottom（警告）
- **阈值：** elbowBottom=80°、elbowTop=160°、elbowFlareMax=0.7

每个分析器均配备 `getSuggestions()` 方法，将错误码映射为中文改进建议，带 emoji 严重度标记（🔴 严重、🟡 注意、✅ 良好）。

**错误追踪**（`formErrors`）：按动作-错误码累计错误次数，供 `generateSuggestions()` 生成训练后分析报告使用。

### 生物力学工具函数

- `angleBetween(a, b, c)` — 三点关节角度计算。通过 arctan2(cross, dot) 求弧度并转度数。边对称，交换端点结果不变。
- `verticalAngle(a, b)` — 线段与水平面的夹角。用于躯干直线度检测。

### 间隔计时器

- 可自定义训练/休息时长及循环次数
- 显示格式：`MM:SS`，附带阶段标签
- 通过 `setInterval` 以 100ms 分辨率计时

### 训练记录

- 数据存储于 `localStorage`，键名 `fitkit_log`
- 每条记录包含：日期、动作名称、次数
- `renderLogs()` 渲染格式化的日期和训练数据

### 测试套件（test.html）

- 轻量测试框架：`test(name, fn)`、`assert(cond, msg)`、`assertEqual`、`assertClose`
- `makeKP(overrides)` 生成模拟的 MoveNet 关键点用于生物力学测试
- 覆盖：`angleBetween` 正确性、深蹲阶段检测、膝内扣、深度、弯举肘部漂移、推举肘外展、防抖逻辑
- 纯 JS，无依赖，浏览器直接打开即可运行

## 重要约定

- 所有代码集中在单文件中 — 除非用户明确要求，否则不要拆分模块
- CSS 变量定义在 `:root` — 使用变量，不要硬编码颜色
- 界面语言为中文 — 所有用户可见文本均为中文
- 不调用任何外部 API — 所有逻辑均在客户端执行
- 模型加载失败的错误提示需包含国内用户的特殊处理方案（tfhub.dev 被墙 → 自托管模型、CDN 镜像切换）

## 常见坑点

- `videoEl.srcObject` vs `videoEl.src` — 摄像头使用 `srcObject`，文件/URL 使用 `src`。切换视频源时务必先调用 `stopCamera()` 释放视频流。
- `videoEl` autoplay：摄像头模式需要 `autoplay` 属性；文件/URL 模式**不能**设置该属性（否则后续 src 变更可能失败）
- 浏览器 autoplay 策略：文件播放时设置 `videoEl.muted = true` 以避免限制
- MoveNet `minPoseScore: 0.2`（低于默认值）— 这对检测不完全可见的姿态至关重要
- 关键点 `score < 0.3` 时分析器返回 `null` — 检测循环跳过该帧，保留上帧状态
- 防抖机制要求连续 `DEBOUNCE_FRAMES` 帧处于同一阶段才会触发阶段切换
- 导航至 `log` 页面时需显式调用 `renderLogs()`，因为日志数据可能已变更
- demo 视频托管在 Mixkit（海外服务器）— 国内用户基本无法加载

## CSS 规范

- 深色主题，通过 CSS 自定义属性控制：`--bg`、`--surface`、`--surface2`、`--border`、`--text`、`--text2`、`--accent`、`--green`、`--yellow`、`--red`
- 响应式：768px 断点处侧边栏收缩为仅图标模式，`tool-grid` 切换为单列
- `.page` 元素默认隐藏；`.page.active` 显示
- 交互元素使用 `.btn` 基类 + 修饰符：`.btn-primary`、`.btn-danger`、`.btn-outline`
- 动作反馈类型：`.good`（绿）、`.warn`（黄）、`.bad`（红）
