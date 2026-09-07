# 第 11 章：视觉 DIY 与动态风景背景

本章把公共站点升级为五幕沉浸式风景体验。视觉原则是“真实风景承载空间感，内容保持安静可读”：背景全视口、缓慢运动并随滚动交叉淡入；导航、卡片和正文使用克制玻璃材质；后台不加载大图。所有素材必须能回答来源、许可、处理过程和 hash。

## 11-01 创建视觉分支和工作目录

```powershell
PS> Set-Location 'D:\CODEkingdom\rodolfoiolo'
PS> git switch main
PS> git pull --ff-only
PS> git status --short --branch
PS> git switch -c feat/immersive-scenes
PS> New-Item -ItemType Directory -Force -Path 'assets\source\backgrounds','assets\work\backgrounds','web\public\images\backgrounds','web\components\visual' | Out-Null
```

在根 `.gitignore` 的素材段加入：

```gitignore
assets/source/
assets/work/
```

原始大图和编辑工程不进 Git，最终优化图与来源登记进 Git。把原始文件同时放入加密备份盘；第 14 章验证恢复。

## 11-02 确定五幕叙事和构图约束

五幕固定为：

| 文件编号 | 场景     | 主色关系            | 构图要求               | 对应滚动位置 |
| -------- | -------- | ------------------- | ---------------------- | ------------ |
| 01       | 清晨山谷 | 冷青天空 + 暖日光   | 中央和左下留暗部给姓名 | 0%-20%       |
| 02       | 林间溪流 | 深绿 + 岩石灰       | 纹理细但不杂乱         | 20%-40%      |
| 03       | 高山湖泊 | 天蓝 + 植被绿       | 地平线位于上三分之一   | 40%-60%      |
| 04       | 云海日落 | 珊瑚暖光 + 中性云层 | 避免整图橙褐           | 60%-80%      |
| 05       | 星空营地 | 深炭黑 + 少量暖灯   | 不用纯深蓝单色         | 80%-100%     |

每幕输出桌面 `2400x1350` 和移动 `1080x1440` 两种裁切。画面不能含文字、Logo、水印、人脸特写或版权角色。

## 11-03 选择唯一素材路径

三种路径只选一种并在来源登记中写明：

1. **自己拍摄**：使用相机/手机 RAW 或最高质量 JPEG，保留原始 EXIF；避开可识别人物和私人场所；
2. **公开授权素材**：只用明确允许网站商业展示和修改的许可，下载许可页面 PDF 与原图；不能只写“来自搜索引擎”；
3. **AI 生成**：使用你有权用于公开网站的模型账户，保留模型名、版本、生成日期、seed（若有）、完整提示词和平台条款链接。

推荐学习路径是自己拍 2 张、AI 生成 3 张，能同时练习摄影后期和生成式素材治理。不要把五张不同摄影师、不同色彩科学的图库图直接拼在一起。

## 11-04 生成 AI 场景

在你选择的图像生成工具中逐张执行。画幅先选 16:9，关闭自动文字。统一基础提示词：

```text
Cinematic but natural landscape photography for a software engineer portfolio background,
real geographic detail, physically plausible light, restrained color grading, generous calm
negative space for readable interface, crisp foreground and distant atmospheric perspective,
no people, no buildings, no text, no logo, no watermark, no fantasy objects, 16:9.
```

在末尾分别追加：

```text
Scene 01: dawn in a layered mountain valley, pale cyan sky, first warm sunlight touching ridges, darker quiet foreground.
Scene 02: temperate forest stream after rain, moss, neutral gray stones, soft shafts of daylight, detailed but uncluttered.
Scene 03: alpine lake with accurate reflections, mixed green vegetation, clear sky, horizon on upper third.
Scene 04: high cloud sea at sunset, coral highlights balanced by neutral gray clouds, no dominant orange cast.
Scene 05: clear night above a mountain campsite, charcoal sky, visible Milky Way, one small warm practical light, no blue monochrome.
```

负面提示词全文：

```text
oversaturated, purple color cast, orange monochrome, teal-orange grade, bokeh orbs,
blurred foreground, fake HDR, illustration, anime, painting, low resolution, text, watermark,
logo, frame, UI mockup, person, face, vehicle, impossible mountains, duplicate trees
```

每幕生成至少 4 个候选，只下载 1 个。放入 `assets/source/backgrounds/scene-0N-original.png`。不要让工具直接覆盖同名文件。

## 11-05 用 Krita 完成 DIY 调色与清理

安装 Krita 后逐张执行相同流程：

1. File -> Open，打开原图；File -> Save As 保存为 `assets/work/backgrounds/scene-0N-master.kra`；
2. Image -> Properties 确认色彩空间为 `RGB/Alpha 16-bit integer/channel, sRGB-elle-V2-srgbtrc.icc`；8 位原图保持 8 位，不虚构色深；
3. 用 Freehand Brush/Clone Tool 清理生成瑕疵、水印残留和重复纹理；放大 100% 检查树枝、反射、星星和地平线；
4. Filter -> Adjust -> Color Balance，用小幅调整让五张的黑位和饱和度接近；任何单通道调整绝对值不超过 12；
5. 新建纯色 `#101714` 图层，置于顶部，混合模式 Normal，不透明度 18%-32%，模拟网站遮罩后检查白字对比；检查后隐藏该层，它不导出到图片；
6. 用 Crop Tool 创建 16:9 桌面构图，Image -> Scale Image to New Size 设为 `2400x1350`；
7. File -> Export Advanced，导出 `assets/work/backgrounds/scene-0N-desktop.png`；
8. Undo 回到 master，再裁 3:4 移动构图，主体不能被中心裁掉，缩放 `1080x1440`，导出 `scene-0N-mobile.png`；
9. 保存 `.kra`，关闭再重开确认工程可恢复。

画面亮暗不能代替前端可读性遮罩；后期不要把图片压得一团黑。

## 11-06 安装本地图片工具

PowerShell 管理员终端：

```powershell
PS> winget install --id ImageMagick.ImageMagick -e
PS> winget install --id Google.WebP -e
```

关闭并重开普通 PowerShell：

```powershell
PS> magick -version
PS> cwebp -version
PS> magick -list format | Select-String 'AVIF|WEBP'
```

若 ImageMagick 构建不支持 AVIF，安装 `libavif` 官方 Windows 二进制并把目录加入当前用户 PATH；不要把来历不明的 exe 放进仓库。

## 11-07 批量输出 WebP 与 AVIF

创建 `scripts/optimize-backgrounds.ps1`，文件全文：

```powershell
[CmdletBinding()]
param()

$ErrorActionPreference = 'Stop'
$root = Split-Path -Parent $PSScriptRoot
$inputRoot = Join-Path $root 'assets\work\backgrounds'
$outputRoot = Join-Path $root 'web\public\images\backgrounds'
New-Item -ItemType Directory -Force -Path $outputRoot | Out-Null

foreach ($number in 1..5) {
    $id = '{0:D2}' -f $number
    foreach ($variant in @('desktop', 'mobile')) {
        $input = Join-Path $inputRoot "scene-$id-$variant.png"
        if (-not (Test-Path -LiteralPath $input)) { throw "Missing $input" }
        $base = Join-Path $outputRoot "scene-$id-$variant"
        & magick $input -strip -colorspace sRGB -define webp:method=6 -quality 80 "$base.webp"
        if ($LASTEXITCODE -ne 0) { throw "WebP conversion failed for $input" }
        & magick $input -strip -colorspace sRGB -define heic:speed=4 -quality 52 "$base.avif"
        if ($LASTEXITCODE -ne 0) { throw "AVIF conversion failed for $input" }
    }
}

$files = Get-ChildItem -LiteralPath $outputRoot -File | Sort-Object Name
$files | Select-Object Name, Length
$total = ($files | Measure-Object -Property Length -Sum).Sum
if ($total -gt 2500000) { throw "Background total is $total bytes; budget is 2500000." }
```

执行：

```powershell
PS> .\scripts\optimize-backgrounds.ps1
PS> Get-ChildItem 'web\public\images\backgrounds' -File | Get-FileHash -Algorithm SHA256 | Select-Object Path,Hash
```

预算是 10 个响应式文件合计 2.5 MB；浏览器每幕只选择 AVIF 或 WebP，不会下载两种格式。第一幕桌面 AVIF 建议不超过 300 KB，其余单文件不超过 350 KB。

## 11-08 重写素材登记

把 `docs/design/assets/background-sources.md` 替换为以下文件全文，并把尖括号字段替换为本次真实记录：

```markdown
# 背景素材来源登记

## 使用规则

只有本表中许可、处理记录和最终 SHA-256 全部非空的文件才能发布。原图和编辑工程位于加密备份，不进入 Git；最终 WebP/AVIF 进入 Git。

| 场景        | 原始来源或生成方式 | 许可/条款链接与快照 | 作者/模型版本 | 生成日期/拍摄日期 | 编辑工程            | 最终文件 SHA-256                    |
| ----------- | ------------------ | ------------------- | ------------- | ----------------- | ------------------- | ----------------------------------- |
| 01 清晨山谷 | <填写>             | <填写>              | <填写>        | <YYYY-MM-DD>      | scene-01-master.kra | <填写桌面和移动两种格式的四个 hash> |
| 02 林间溪流 | <填写>             | <填写>              | <填写>        | <YYYY-MM-DD>      | scene-02-master.kra | <填写>                              |
| 03 高山湖泊 | <填写>             | <填写>              | <填写>        | <YYYY-MM-DD>      | scene-03-master.kra | <填写>                              |
| 04 云海日落 | <填写>             | <填写>              | <填写>        | <YYYY-MM-DD>      | scene-04-master.kra | <填写>                              |
| 05 星空营地 | <填写>             | <填写>              | <填写>        | <YYYY-MM-DD>      | scene-05-master.kra | <填写>                              |

## 处理命令

统一由 `scripts/optimize-backgrounds.ps1` 生成。脚本版本对应提交：`<填写 git commit SHA>`。

## 人工审查

- [ ] 100% 缩放无生成瑕疵、水印、人物或隐私信息；
- [ ] 桌面与移动裁切不遮挡主体；
- [ ] 五幕没有单一紫、蓝、橙或褐色统治；
- [ ] 来源页面和许可快照已放入加密备份；
- [ ] hash 与仓库最终文件逐项一致。
```

这些尖括号是你必须产生的项目事实，不是可直接复制的示例值。未填完时本章不能提交。

## 11-09 创建平滑滚动控制器

创建 `web/components/visual/smooth-scroll.tsx`，文件全文：

```tsx
'use client';

import { useEffect, type ReactNode } from 'react';
import Lenis from 'lenis';

export function SmoothScroll({ children }: { children: ReactNode }) {
  useEffect(() => {
    if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) return;
    if (window.matchMedia('(pointer: coarse)').matches) return;
    const lenis = new Lenis({ autoRaf: true, duration: 0.9, smoothWheel: true });
    return () => lenis.destroy();
  }, []);
  return <>{children}</>;
}
```

移动端保留浏览器原生滚动，减少触摸延迟。减少动效模式完全不实例化 Lenis。

## 11-10 创建五幕背景组件

创建 `web/components/visual/scene-background.tsx`，文件全文：

```tsx
'use client';

import { motion, useReducedMotion, useScroll, useTransform, type MotionValue } from 'motion/react';

const source = (number: number, format: 'avif' | 'webp', mobile = false) =>
  `/images/backgrounds/scene-${String(number).padStart(2, '0')}-${mobile ? 'mobile' : 'desktop'}.${format}`;

function Scene({
  number,
  opacity,
  eager = false,
}: {
  number: number;
  opacity: number | MotionValue<number>;
  eager?: boolean;
}) {
  return (
    <motion.picture className={`scene scene-${number}`} style={{ opacity }}>
      <source media="(max-width: 720px)" srcSet={source(number, 'avif', true)} type="image/avif" />
      <source media="(max-width: 720px)" srcSet={source(number, 'webp', true)} type="image/webp" />
      <source srcSet={source(number, 'avif')} type="image/avif" />
      <img
        className="scene-image"
        src={source(number, 'webp')}
        alt=""
        width="2400"
        height="1350"
        fetchPriority={eager ? 'high' : 'auto'}
        loading={eager ? 'eager' : 'lazy'}
        decoding="async"
      />
    </motion.picture>
  );
}

export function SceneBackground() {
  const { scrollYProgress } = useScroll();
  const reduced = useReducedMotion();
  const first = useTransform(scrollYProgress, [0, 0.16, 0.28], [1, 1, 0]);
  const second = useTransform(scrollYProgress, [0.14, 0.28, 0.38, 0.5], [0, 1, 1, 0]);
  const third = useTransform(scrollYProgress, [0.36, 0.5, 0.58, 0.7], [0, 1, 1, 0]);
  const fourth = useTransform(scrollYProgress, [0.56, 0.7, 0.78, 0.9], [0, 1, 1, 0]);
  const fifth = useTransform(scrollYProgress, [0.76, 0.9, 1], [0, 1, 1]);

  return (
    <div className="scene-background" aria-hidden="true">
      {reduced ? (
        <Scene number={1} opacity={1} eager />
      ) : (
        <>
          <Scene number={1} opacity={first} eager />
          <Scene number={2} opacity={second} />
          <Scene number={3} opacity={third} />
          <Scene number={4} opacity={fourth} />
          <Scene number={5} opacity={fifth} />
        </>
      )}
      <div className="scene-veil" />
    </div>
  );
}
```

动态模式和减少动效模式都只渲染一份第一幕；首图始终高优先级，其余场景延迟解码。

## 11-11 替换公共布局全文

把 `web/app/(public)/layout.tsx` 替换为：

```tsx
import type { ReactNode } from 'react';
import { Footer } from '@/components/layout/footer';
import { Header } from '@/components/layout/header';
import { SceneBackground } from '@/components/visual/scene-background';
import { SmoothScroll } from '@/components/visual/smooth-scroll';
import { api } from '@/lib/api';

export default async function PublicLayout({ children }: { children: ReactNode }) {
  const settings = await api.settings();
  return (
    <SmoothScroll>
      <SceneBackground />
      <div className="public-layer">
        <Header />
        {children}
        <Footer settings={settings} />
      </div>
    </SmoothScroll>
  );
}
```

## 11-12 替换全局样式全文

把 `web/app/globals.css` 替换为以下全文：

```css
@import 'tailwindcss';

@theme {
  --color-ink: #18201d;
  --color-paper: #f5f7f2;
  --color-surface: #ffffff;
  --color-muted: #5f6b65;
  --color-line: #d6ddd7;
  --color-moss: #2f6b4f;
  --color-sky: #397b9c;
  --color-coral: #c75b4a;
  --font-sans: 'Noto Sans SC', 'Segoe UI', system-ui, sans-serif;
  --font-display: 'Space Grotesk', 'Segoe UI', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', Consolas, monospace;
}
:root {
  color-scheme: light;
  --page-bg: #f5f7f2;
  --panel-bg: rgb(255 255 255 / 78%);
  --text: #18201d;
  --text-muted: #46544e;
  --border: rgb(24 32 29 / 18%);
  --focus: #397b9c;
  --shadow: 0 12px 30px rgb(16 30 23 / 16%);
}
@media (prefers-color-scheme: dark) {
  :root {
    color-scheme: dark;
    --page-bg: #111714;
    --panel-bg: rgb(18 27 23 / 82%);
    --text: #eff4ef;
    --text-muted: #bdc9c2;
    --border: rgb(239 244 239 / 20%);
    --focus: #7fc4de;
    --shadow: 0 14px 36px rgb(0 0 0 / 34%);
  }
}
* {
  box-sizing: border-box;
}
html {
  scroll-behavior: smooth;
  background: var(--page-bg);
}
body {
  margin: 0;
  min-height: 100vh;
  background: transparent;
  color: var(--text);
  font-family: var(--font-sans);
  line-height: 1.7;
  letter-spacing: 0;
}
a {
  color: inherit;
  text-decoration-thickness: 1px;
  text-underline-offset: 4px;
}
img {
  display: block;
  max-width: 100%;
}
button,
input,
textarea,
select {
  font: inherit;
}
:focus-visible {
  outline: 3px solid var(--focus);
  outline-offset: 3px;
}
.public-layer {
  position: relative;
  z-index: 1;
  min-height: 100vh;
}
.public-layer main > section {
  background: rgb(245 247 242 / 34%);
}
.glass {
  border: 1px solid var(--border);
  border-radius: 8px;
  background: var(--panel-bg);
  box-shadow: var(--shadow);
  backdrop-filter: blur(16px) saturate(118%);
}
.prose {
  max-width: 72ch;
  border-radius: 8px;
  padding: clamp(18px, 4vw, 44px);
  background: var(--panel-bg);
  box-shadow: var(--shadow);
  backdrop-filter: blur(18px) saturate(115%);
}
.prose h2,
.prose h3 {
  font-family: var(--font-display);
  line-height: 1.25;
}
.prose pre {
  overflow-x: auto;
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 1rem;
  background: #101714;
  color: #eef6ef;
}
.prose code {
  font-family: var(--font-mono);
}
.prose :not(pre) > code {
  border-radius: 4px;
  padding: 0.12rem 0.34rem;
  background: color-mix(in srgb, var(--text) 9%, transparent);
}
.scene-background {
  position: fixed;
  z-index: 0;
  inset: 0;
  overflow: hidden;
  background: #17201d;
  pointer-events: none;
}
.scene {
  position: absolute;
  inset: 0;
  display: block;
}
.scene-image {
  width: 100%;
  height: 100%;
  max-width: none;
  object-fit: cover;
  transform: scale(1.045);
  animation: scene-drift 24s ease-in-out infinite alternate;
}
.scene-2 .scene-image,
.scene-4 .scene-image {
  animation-direction: alternate-reverse;
}
.scene-veil {
  position: absolute;
  inset: 0;
  background: rgb(8 16 12 / 30%);
}
@keyframes scene-drift {
  from {
    transform: scale(1.045) translate3d(-0.4%, -0.25%, 0);
  }
  to {
    transform: scale(1.085) translate3d(0.4%, 0.25%, 0);
  }
}
@media (max-width: 720px) {
  .scene-image {
    animation: none;
    transform: scale(1.02);
  }
  .public-layer main > section {
    background: rgb(245 247 242 / 44%);
  }
}
@media (prefers-color-scheme: dark) {
  .public-layer main > section {
    background: rgb(10 17 14 / 42%);
  }
  .scene-veil {
    background: rgb(5 10 8 / 42%);
  }
}
@media (prefers-reduced-motion: reduce) {
  html {
    scroll-behavior: auto;
  }
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
  .scene-image {
    transform: none;
  }
}
```

遮罩使用单一透明色，不用装饰渐变。玻璃模糊只用于真实承载内容的表面，不在卡片中再嵌套卡片。

## 11-13 调整首页阅读节奏

第 08 章首页已有三段内容。把 Hero 段的 `min-h-[72vh]` 保留；把两个内容段的 `py-20` 改为 `py-24 md:py-32`，让滚动有足够距离完成场景交叉淡入。不要添加解释动画机制的可见文字。

## 11-14 做视觉和像素验收

先启动 API 与 Web，再打开 Chrome DevTools：

1. Desktop `1440x900`：姓名在首屏出现，下一节有可见提示；导航、文字和背景没有争夺；
2. Mobile `375x812`：使用竖图，无横向滚动，标题单词不溢出；
3. Elements -> Rendering -> Emulate CSS media feature `prefers-reduced-motion: reduce`：只显示第一幕，背景不缩放，原生滚动；
4. Network -> Disable cache，刷新：首页首幕优先，未滚到的图片 lazy；同一场景只下载 AVIF 或 WebP；
5. Performance 录制一次从顶到底滚动：主线程无持续长任务，帧率目标 50fps 以上；
6. Lighthouse Accessibility：对比度不得用“背景看起来暗”主观通过，正文与玻璃面板需达到 WCAG AA；
7. 禁用某张图片请求：底色仍可读，没有白屏；
8. 检查后台：不请求任何 `scene-*` 文件。

使用 Playwright 截图的命令在第 13 章统一加入 CI；本章先把桌面、移动和 reduced-motion 三张验收图放进 PR 描述，不提交截图文件。

## 11-15 运行静态门禁并提交

```powershell
PS> pnpm --dir web lint
PS> pnpm --dir web typecheck
PS> pnpm --dir web format
PS> pnpm --dir web format:check
PS> .\scripts\optimize-backgrounds.ps1
PS> git diff --check
PS> git add -- .gitignore scripts/optimize-backgrounds.ps1 docs/design/assets/background-sources.md web/components/visual 'web/app/(public)/layout.tsx' web/app/globals.css web/public/images/backgrounds
PS> git diff --cached --stat
PS> git commit -m 'feat(visual): add responsive scroll-driven landscape scenes'
PS> git push --set-upstream origin feat/immersive-scenes
```

PR 中逐项附上素材来源、授权结论、总字节数、三种视口截图、Performance 录制结论和 reduced-motion 结果。CI 全绿并人工视觉复核后合并。

## 11-16 本章停止点

- [ ] 五幕都有桌面/移动 AVIF 与 WebP；
- [ ] 素材来源、许可、编辑工程、处理命令和 hash 全部登记；
- [ ] 总预算不超过 2.5 MB，首幕桌面 AVIF 不超过 300 KB；
- [ ] 滚动交叉切换和轻微 Ken Burns 生效；
- [ ] 移动端不用 Lenis 和背景动画；
- [ ] reduced-motion 只显示静止第一幕；
- [ ] 玻璃界面可读、无卡片套卡片、圆角不超过 8px；
- [ ] 375px 与 1440px 无溢出或遮挡；
- [ ] 后台不加载场景素材；
- [ ] lint、typecheck、format 和生产 build 通过。
