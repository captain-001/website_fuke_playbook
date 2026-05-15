# Shopify 网页移植 Master Playbook

把抓取得到的高端动效网页（Nuxt SSG / Webflow / 静态站）移植成可在 Shopify 上预览/部署的 Liquid theme 的完整流程规范。

源自两个真实项目的踩坑沉淀：
- `fluidglass-theme/` ← Nuxt SSG 站 `fluid.glass`（2026-05）
- `ffd-theme/` ← Nuxt SSG 站 `fluid-folding-door`（2026-05-14）
- `more-nutrition-theme/` ← Webflow 站 `morematcha.vercel.app`（2026-05-15）

> 本文是**入口和总览**。深入细节请按章节链接跳转到 `01-`~`07-` 章节文件和 `BUGS.md`。

---

## 0. 接到任务先做的 5 件事

1. **读这份 playbook 全文**（特别是 §1 决策矩阵和 §10 元教训）
2. **问清楚目标店铺**：装了哪些第三方 app？现役主题是 Dawn / Lumia / GemPages 派生？
3. **看一眼源材料**：HTML 是 Nuxt SSG / Webflow / 还是普通静态？有没有附带的 CSS bundle、JS bundle？
4. **统计资源规模**：源站资产（图片、字体、视频、JS、CSS）总大小？单文件大小？多少个？
5. **决定架构路线**：见 §1，绝大多数情况走 **Phase 2 原生 sections**

---

## 1. 架构决策矩阵

| 选项 | 适用 | 致命风险 |
|---|---|---|
| **Phase 1：SPA 嵌入** —— 把整站 HTML 塞进单个 `index.liquid` | 店铺**完全没装**任何 Vue/Nuxt/React 嵌入式 app，且页面是纯静态 | Shopify `{{ content_for_header }}` 注入的第三方 app script（评测、社交 feed、个性化推荐）会跟主页面 SPA 的全局 context 打架；hydration 报 `beforeEach undefined` / `unctx Context 冲突`，**不可救** |
| **Phase 2：原生 Liquid sections** —— 每个区块拆成独立可编辑 section | **默认选择** | 工作量大，需要逐 section 复刻动效；DOM 拆分时祖先关系容易断（marquee 案例），CSS 串扰需要逐条防御 |
| **Phase 1.5：单 section 含整 body** —— sections 架构，但只有 1 个 section 含全部内容 | 妥协方案：admin 不需要 per-section 编辑，但你又想用 sections 架构跳过 Phase 1 的 SPA context 问题 | DOM 完整保留，动效不易断；但失去 Theme Editor 的拖动重排能力 |

**判断口诀**：
- 店铺装了任何 Vue/Nuxt 嵌入式 app → 走 Phase 2
- 源站动效完全依赖独立 JS runtime（GSAP / Lenis / 大量 ScrollTrigger） → 走 Phase 2 或 1.5
- 都没有 → Phase 1 也可以试，但仍优先 Phase 2

---

## 2. 标准目录结构

```
{name}-theme/                       # Shopify 主题根（通常从母版主题 cp -r 来）
├── assets/
│   ├── {prefix}-base.css           # 设计 token + Dawn cascade 防御 + 字号体系
│   ├── {prefix}-{section}.js       # （可选）section 局部 JS
│   ├── {prefix}-{webflow-shared}.css   # 原站 CSS bundle（重命名）
│   ├── {prefix}-{webflow-app}.js       # 原站 JS bundle
│   ├── {prefix}-gsap-*.min.js          # GSAP + 全部插件
│   └── (Lumia/Dawn 自带文件全部保留)
├── config/
│   └── settings_data.json          # 母版 setting 全留
├── layout/
│   └── theme.liquid                # 改：index 模板上隐藏母版 header/footer/breadcrumbs，注入自家 CSS/JS
├── sections/
│   └── {prefix}-{name}.liquid      # 全部带 prefix
├── snippets/
│   └── {prefix}-{name}.liquid      # 同上
└── templates/
    ├── index.json                  # 自建 N 个 prefix-* section
    └── (其他模板保留母版默认)
```

详见 [01-architecture.md](01-architecture.md)。

---

## 3. 从源 HTML 到 sections 的转换流程

### 3.1 准备阶段

1. 把源站本地副本完整下载到 `designs/{NNN-name}/`（HTML + 全部 assets 子目录）
2. **统计**：单文件最大多少？总数？最大的 JS / CSS 多大？
3. **grep 关键标记**：
   ```bash
   grep -oE '<(section|div)[^>]*class="[^"]*(section-|hero|stage|footer)[^"]*"' index.html | head -30
   ```
   定位顶层 section 边界。Webflow 通常用 `<section class="{name}">` 或 `<div data-{role} class="{name}-section">`。

### 3.2 切片脚本 `split-{name}-body.mjs`

参考 `d:/shopify_try/split-mn-body.mjs`。逻辑：
1. 抽出 `<body>` 内 innerHTML
2. 定义有序 markers 数组（每个标记一个顶层 section 的开头字符串）
3. 按 marker 切片，每片写到 `sections-raw/{NN}-{name}.html`

**关键验证**：每个切片必须 `<div>` / `<section>` 开合平衡（用 node 数 open/close）。**不平衡的切片在 Shopify Theme Editor 会报 `HTML error found`**。

边界陷阱：
- 源 DOM 嵌套关系跟视觉无关。源里 `.footer-bridge` 看着像独立区块，但实际嵌在 `.payment-section` 里。切错会导致：(a) tag 不平衡，(b) absolute 定位的子元素失去定位祖先
- 父 section 用 `<section>` 标签，子 marquee 在父内 → 父的 `</section>` 会被切到下一片
- **永远在切片后跑 div/section 平衡校验**：
  ```js
  const opens = (slice.match(/<div\b[^>]*>/g) || []).length;
  const closes = (slice.match(/<\/div>/g) || []).length;
  ```

### 3.3 资源扁平化 `flatten-{name}-assets.mjs`

参考 `d:/shopify_try/flatten-mn-assets.mjs`。逻辑：
1. 遍历 `designs/{NNN-name}/assets/**` 全部文件
2. 给每个文件生成扁平化目标名（`assets/{flat}`）
3. 命名规则：
   - 大块 bundle（CSS/JS）→ `{prefix}-{role}.{ext}`，如 `mn-webflow.js`、`mn-vercel-app.js`、`mn-gsap-scrolltrigger.min.js`
   - 哈希前缀的 CDN 资源（图片、字体、SVG）→ 保留 hash 前缀（已经唯一），把空格替换成下划线
4. 拷贝时构建 `old-path → new-name` 映射
5. **同时改写 HTML 切片**：把所有 `assets/cdn.prod.website-files.com/.../foo.png` 替换成 placeholder `{{ASSET:foo.png}}`，最终包装时再变成 `{{ 'foo.png' | asset_url }}`

**容易漏的一步：改写 CSS bundle 内的 `url()` 引用**：
- 原 CSS 文件位于 `assets/.../css/foo.min.css`，引用 `url(../68d3d0d_font.woff2)`
- 扁平化后 CSS 位于 `assets/{prefix}-foo.css`，原相对路径 `../68d3d0d_font.woff2` 找不到
- **必须扫 CSS 里所有 `url(...)`，把相对路径改成裸文件名**（跟 CSS 同 dir）
- Shopify 静态 CSS **不会处理 Liquid**，所以直接写裸名 `url("foo.woff2")` 即可。CSS 同目录解析 = `/cdn/shop/t/{theme}/assets/foo.woff2`

如果漏掉这一步：
- 字体 404 → 退回 Arial → 同字号下文字宽 30-40%，看起来"字号偏大"（这是 more-nutrition 项目的真实踩坑）
- pattern.png 404 → loader 缺少底纹但页面照常

### 3.4 生成 section.liquid `build-{name}-sections.mjs`

参考 `d:/shopify_try/build-mn-sections.mjs`。每片 raw HTML：
1. `{{ASSET:foo}}` → `{{ 'foo' | asset_url }}` 全局替换
2. 包 `{% schema %}` 块（`tag: "div"`, `class: "shopify-section--{prefix}"`, 空 settings + 一个 preset）
3. footer slice 要剥离末尾的 `<script src=...>` 链（这些移到 layout/theme.liquid）

写完跑一次平衡校验，**任何不平衡都要在 Theme Editor 之前修掉**。

详见 [06-section-conventions.md](06-section-conventions.md)。

---

## 4. `layout/theme.liquid` 改造

### 4.1 模板判定的坑（**重要新教训**）

| 写法 | 在 Dawn 上 | 在 Lumia/GemPages 派生上 | 推荐 |
|---|---|---|---|
| `template == 'index'` | ✓ | 有时不灵 | ✗ |
| `template.name == 'index'` | ✓ | 有时不灵 | ✗ |
| `request.page_type == 'index'` | ✓ | ✓ | **✓ 用这个** |

**症状**：在 Lumia/GemPages 派生主题上，`template == 'index'` 在 body class 字符串里第 N 次使用时不firs，但 `template-{{ request.page_type | handle }}` 渲染出 `template-index`——说明 `request.page_type` 是可靠的。

**根因猜测**：GemPages 通过 `index.gem-*-template.json` 等模板别名 + 多 layout 文件（`theme.gempages.*.liquid`），中间某层 render 可能改写了 `template` 对象的 stringify 结果。`request.page_type` 是 Shopify 原生的请求级变量，不受模板别名影响。

**铁律**：所有 index 模板 gating 一律用 `{%- if request.page_type == 'index' -%}` / `{%- unless request.page_type == 'index' -%}`。

### 4.2 必做的 gating（在母版 theme.liquid 上嫁接）

```liquid
{%- if request.page_type == 'index' -%}
  {{ '{prefix}-webflow-shared.css' | asset_url | stylesheet_tag }}
  {{ '{prefix}-vercel-main.css' | asset_url | stylesheet_tag }}
  {{ '{prefix}-base.css' | asset_url | stylesheet_tag }}
{%- endif -%}

{# 隐藏母版 header / announcement / breadcrumbs / navigation-sub / promo-products / footer #}
{%- unless request.page_type == 'index' -%}
  {%- liquid
    section 'announcement-bar'
    section 'header'
    render 'navigation-sub'
    render 'header-navigation-mobile-bottom'
    render 'breadcrumbs'
  -%}
{%- endunless -%}

{# 关闭 Lumia ajax-scroll（详见 BUGS.md P3-04），方法是把 if 条件搞成永远 false #}
{%- if settings.use_ajax_scroll and request.page_type == 'index' and request.design_mode == false and request.page_type != 'index' -%}
  {# unreachable #}
{%- else -%}
  {{ content_for_layout }}
{%- endif -%}

{# 内容包一层 wrapper（CSS 作用域用） #}
{%- if request.page_type == 'index' -%}
  {%- render '{prefix}-navbar' -%}
  <div class="page-wrapper {prefix}-page">
    {{ content_for_layout }}
  </div>
{%- endif -%}

{# body 末尾按依赖顺序加载 JS #}
{%- if request.page_type == 'index' -%}
  <script src="{{ '{prefix}-jquery.min.js' | asset_url }}"></script>
  <script src="{{ '{prefix}-webflow.js' | asset_url }}"></script>
  <script src="{{ '{prefix}-gsap-gsap.min.js' | asset_url }}"></script>
  {# 所有 GSAP plugin 紧跟 core #}
  <script>gsap.registerPlugin(ScrollTrigger,SplitText,CustomEase,DrawSVGPlugin,InertiaPlugin);</script>
  <script src="{{ '{prefix}-vercel-app.js' | asset_url }}" defer></script>
{%- endif -%}
```

详见 [04-shopify-deploy.md](04-shopify-deploy.md)。

---

## 5. CSS 串扰防御（最关键的一章）

母版主题（Dawn/Lumia）会注入大量全局 CSS。下面是已踩过的坑和防御。

### 5.1 字号体系的三层（最常见踩坑）

| 单位 | 相对谁 | 母版破坏方式 |
|---|---|---|
| `px` | 自己 | 不受影响 |
| `em` | 直接父元素 `font-size` | 父 font-size 被改时跟着错 |
| `rem` | `<html>` 元素 `font-size` | **Lumia 设 `html { font-size: 20px }`**，所有 rem 放大 25% |
| `var(--size-font)` | viewport 宽度（calc 派生） | 一般不变，除非 viewport CSS 被改 |

**Lumia html font-size 案例**（more-nutrition 实测）：
- 源站 `.marquee-text-svg text { font-size: 11rem }` 在浏览器默认 16px 下 = 176px
- Lumia 把 html 字号改成 20px → 11rem = 220px，**大 25%**
- 同样 `.button { padding: 0.5rem 1rem }` 等也按 20px 算，按钮、间距全大 25%

**修法**：在 `{prefix}-base.css`（只在 index 模板加载）顶端加：
```css
html { font-size: 16px !important; }
```
mn-base.css 加载范围只在首页 → 不影响其他模板。

### 5.2 body 背景 / 字号 / 字体被覆盖

母版常见的 body 规则：
- Webflow shared CSS: `body { font-size: 14px } body { font-size: 16px } .body { background-color: var(--bg-pale); font-family: Founders Grotesk Condensed }`
- Lumia bundle-head: `body { font-size: 12px; background-color: var(--body-bg); color: var(--text-color) }`

**关键观察**：Webflow 的 `.body` rule 依赖 body 元素有 `class="body"`，但 Shopify 主题 body class 已经被母版填了一堆系统 class（`shopify-theme template-{type} ...`）。

**两种解决路径**：

**A. 在 theme.liquid 给 body 加上 `body {prefix}-index` class**：
```liquid
class="shopify-theme template-... {% if request.page_type == 'index' %} body {prefix}-index{% endif %}"
```
然后所有 cascade 防御写在 `body.{prefix}-index { ... !important }`。

**B. 在 `.{prefix}-page` 包裹层上重新声明等价 CSS**（推荐，不依赖 body class 加成功）：
```css
.{prefix}-page {
  font-size: var(--size-font);
  font-family: "Founders Grotesk Condensed", Arial, sans-serif;
  background-color: var(--bg-pale, #e8efe4);
  color: var(--dark-green, #335c30);
}
```

两条路径同时上更稳（A 是 belt，B 是 suspenders）。

### 5.3 Dawn 全局 class 雷区

| 通用 class | Dawn 注入啥 | 防御 |
|---|---|---|
| `.button` | padding, min-height, box-shadow, font, letter-spacing | 全部显式重置 `appearance: none; padding: 0; min-height: 0; box-shadow: none; letter-spacing: normal` |
| `.icon` | 全局尺寸/颜色 | 重命名为 `.{prefix}-icon` 或显式重置 |
| `.media` / `.field` / `.title` | 各种重置 | 同上 |
| `body.gradient` | `background-color: rgb(var(--color-background))` | `body.{prefix}-index.gradient { background: ... !important }` 两层属性都覆盖 |

**铁律**：任何源代码里使用通用名 class（`.button`, `.card`, `.grid`, `.icon`, `.title`, `.field`, `.media`）的，要么**重命名加 prefix**，要么在 base.css 里**显式重置**。

详见 [02-css-cascade.md](02-css-cascade.md)。

### 5.4 ajax-scroll 和 first-section-off-animate

母版 Lumia 在 `template == 'index'` 上启用 ajax-scroll —— **第三个 section 起被替换成占位符**，等用户滚到附近才 ajax 加载。GSAP ScrollTrigger 在 init 时测不到尚未存在的元素，所有滚动驱动动画失效。

**修法**：见 §4.2 的 gating 模式，把 ajax-scroll 的 if 条件改成永远 false。

同时 body class 上的 `first-section-off-animate` / `initial-hide` / `full-load-hide` 会让 section 在加载完前 opacity:0。修法：
```css
body.{prefix}-index .shopify-section { opacity: 1 !important; transform: none !important; animation: none !important; }
```

---

## 6. JS / 动效注入

### 6.1 加载顺序

```
<head>:
  1. Lumia bundle-head.css       (母版)
  2. content_for_header           (Shopify scripts)
  3. {prefix}-webflow-shared.css  (源站 CSS bundle)
  4. {prefix}-vercel-main.css     (源站自定义 CSS)
  5. {prefix}-base.css            (我们的防御 + 字号体系)

<body 末尾>:
  6. jquery
  7. webflow.js                   (Webflow runtime)
  8. gsap + 全部 plugin            (按依赖顺序)
  9. <script>gsap.registerPlugin(...)</script>
  10. vercel-app.js / 源站 JS      (依赖 GSAP 已注册)
```

`mn-vercel-app.js` 这种大 bundle 一般体积 300-500KB；Shopify 单文件上限是 20MB，没问题。

### 6.2 幂等保护

每个独立 JS 用 IIFE + 全局标志位，防止 Theme Editor 编辑 section 时重跑：
```js
(function () {
  if (window.__{prefix}{Name}Init) return;
  window.__{prefix}{Name}Init = true;
  // ... 实际逻辑
})();
```

### 6.3 Smooth-scroll 在 Lumia 派生母版上的坑

**Lenis 等第三方平滑滚动库在装了 wheel listener 的母版上会静默失败**——母版 listener 抢在 Lenis 前面，`preventDefault` 形同虚设。

修法：弃用 Lenis，手写 ~40 行 wheel handler：
```js
window.addEventListener('wheel', (e) => {
  e.preventDefault();
  target += clampedDelta * WHEEL_MULTIPLIER;
}, { passive: false });
// rAF tick: current += (target - current) * LERP; window.scrollTo(0, current);
```

调参顺序：`LERP`（0.08-0.12 决定丝滑度）→ `WHEEL_MULTIPLIER`（0.6-0.9 决定累积速度）→ `MAX_DELTA`（80-120 防高 DPI 飞屏）。**绝对不要加硬速度上限**——会破坏 native 速度感。

详见 [BUGS.md](BUGS.md) §P3-05。

### 6.4 测量类 JS 必须等字体加载

任何依赖文字宽度测量的 JS（行拆分、宽度自适应、SplitText）**必须等 `document.fonts.ready`**：
```js
await document.fonts.ready;
splitText.init();
```
否则按系统字体测，加载完真字体后偏移。

详见 [03-animations.md](03-animations.md)。

---

## 7. 部署流程

### 7.1 首次推送

```bash
cd /d/shopify_try/{name}-theme
shopify theme push --unpublished --theme="{Name}-v1" --json --no-color -s {store}.myshopify.com
```

`--json` 输出 `preview_url` 和 theme ID（**记住，迭代时要用**）。`--unpublished` 创建未发布主题，**不冲突现有 live theme**。

### 7.2 迭代推送

```bash
# 全量推
shopify theme push --theme="{Name}-v1" --no-color -s {store}.myshopify.com

# 单文件推（快 10x）
shopify theme push --theme="{Name}-v1" --no-color -s {store}.myshopify.com --only "assets/{prefix}-base.css"
shopify theme push --theme="{Name}-v1" --no-color -s {store}.myshopify.com --only "sections/{prefix}-stage.liquid"
```

**别每次都 `--unpublished`**，会创建一堆未发布主题。

### 7.3 PowerShell 坑

- `--nodelete=$false` PowerShell 会展开成 `--nodelete=False`，CLI 报错。**别写这个 flag**
- `Compress-Archive` 写 zip 时用反斜杠，Shopify 不识别。**永远用 `shopify theme push`，不手工打包上传**

详见 [04-shopify-deploy.md](04-shopify-deploy.md)。

---

## 8. 诊断方法（**新增**：避免错误诊断浪费时间）

### 8.1 优先级顺序

1. **Console errors** → 看红色报错。`gsap is not defined`、`Cannot read property of undefined`、`MIME type`、`404`
2. **Network 404s** → filter 输入 `{prefix}-`，看 CSS/JS/字体/图片是不是 200。**字体 404 最隐蔽**，回退到 Arial 后视觉上看着像"字号偏大"
3. **Computed font-size** → dev tools 选元素 → Computed tab。看 `font-size` 那行哪个 CSS 文件赢了，验证 cascade
4. **DOM 结构** → 看预期的 `.{prefix}-page`、`.shopify-section--{prefix}` 等 wrapper 是否真的存在
5. **截图对比** → 永远只是辅助，不是诊断起点

### 8.2 ❌ 不要做的事

- **不要用 `curl` 当 preview 诊断工具**。curl 没有 preview cookie，会被重定向到 live theme，给你完全不相关的 HTML。永远在**浏览器实际开 preview URL** + dev tools 诊断
- **不要凭截图猜原因**。截图最多告诉你"出问题了"，但根因可能是字体/布局/JS/CSS 任一层
- **不要反复调动效参数**。如果用户连续 3+ 次反馈"还是不对"，**先停下，grep 原站对应 CSS scope**，一次性把规格拿全。详见 [README.md](README.md) §元教训

### 8.3 字号偏大的标准排查

按这个顺序查：

1. `<html>` computed font-size：是不是 20px（Lumia 痕迹）？应该是 16px。修法见 §5.1
2. 测量元素的 computed font-size：跟你预期算出来的对不对？
3. 字体加载状态：dev tools → Network → Fonts，看 woff2 是不是 200。404 → 退回 Arial → 宽 30-40%
4. 容器宽度：是不是被 grid 错配挤窄了？文字溢出看着像字号大

详见 BUGS.md 中 more-nutrition 项目的 rem 案例。

---

## 9. 已修 bug 速查（按类型分组）

完整 23+ 条 bug 见 [BUGS.md](BUGS.md)。下面按**根因类型**分类摘要：

### CSS cascade
- **P2-01 Hero 文字颜色** — Dawn `.fg-section{color:grey}` 压过自家 `.fg-home-header{color:white}`。修：高 specificity + 多 selector
- **P2-13 button 撞 Dawn** — 改名 `arrow-btn` + 显式重置 appearance/padding/box-shadow
- **P2-15 body 米色背景** — Dawn `body.color-background-1` 压过 `body { background: cream }`。修：`!important` + `body.gradient` 双 selector
- **P3-02 rem 单位全大 2-3 倍** — Dawn `html { font-size: ~16px }` 干预 `var(--font-s)`。修：`html { font-size: var(--font-s) !important }` 限定 index
- **P3-06 base-button CSS 用了 `.ffd-page` 前缀** — modal/floating-nav 在 body 上没接到样式。修：共享组件用 bare class

### DOM 拆分 / 嵌套
- **marquee 跑到底部** — 源 DOM 里 marquee 是 stage 的 child（`position: absolute; bottom: 0` 依赖 `.stage{position:relative}` 做祖先）。修：合并 marquee 到 mn-stage section
- **mn-payment HTML error** — splitter 切错边界，`.footer-bridge` 在源 DOM 里是 payment-section 的 child，但被分到 mn-footer slice。修：手工 rebalance 切片

### 模板 gating
- **template.name 不灵** — GemPages/Lumia 派生主题上 `template.name == 'index'` 时灵时不灵。修：全部用 `request.page_type == 'index'`
- **P3-04 ajax-scroll 拆 section** — Lumia 在 index 上把第 3+ section 替换成占位符。修：if 条件改永远 false
- **P3-03 breadcrumbs 钻进** — `{% render 'breadcrumbs' %}` 不在 unless 块。修：unless template

### 资源 / 字体
- **Webflow CSS url() 引用 404** — 扁平化时只复制文件，CSS 内 `url(../path/foo.woff2)` 没改。修：扫 CSS 把相对路径改成裸文件名
- **P2-19 Storyblok CDN 404** — 原 URL 用了被禁用的变换 filter。修：下载到本地

### 动效 / 滚动
- **P3-05 Lenis 在 Lumia 上失效** — 母版 wheel listener 抢断。修：弃用 Lenis，手写 wheel handler ~40 行
- **P2-24 滚动卡顿** — 两层平滑（Lenis 等价 + progress lerp）。修：`SCROLL_LERP` 调到 0.06 + `normalizeDelta` 处理 Firefox lines mode
- **P2-08 lazy image 测量失败** — `loading="lazy"` 时 JS 测 `clientHeight` 返回 0。修：改 eager + 挂 `load` 事件 fallback
- **P2-16 入场动画只第一个有** — 一次性广播 `is-revealed`。修：IntersectionObserver per section
- **P2-17 行拆分错位** — 字体没加载完就测宽。修：`await document.fonts.ready`

### 部署 / Liquid
- **P1-02 单 Liquid > 256KB** — 873KB Nuxt `__NUXT_DATA__` 内联。修：拆 chunk + `{%- render -%}` 拼接
- **P1-03 render 插换行破 JSON** — 用 `{%- render -%}` 双侧 trim
- **P3-01 schema tag: null** — Shopify 不接受。修：用 `"tag": "div"` 或 `"section"`

---

## 10. 元教训（**最重要**）

### 10.1 反复修不到位 → grep 原站 CSS

**任何动效返工 5 次以上**（菜单、carousel、入场……），用户连续 3+ 次反馈"还是不对"——**先停下**，去 grep 原站对应 data-v scope 的 CSS：

```bash
# 在源站 CSS 里找对应 selector
grep -oE '\.{name}[^{,]*\{[^}]{0,400}\}' designs/{NNN-name}/assets/.../shared.css
```

一次性拿全所有规格（尺寸、位置、transform-origin、动画时长、easing），比反复迭代快 5-10 倍。

这条**踩了至少 3 次**：FFD 菜单 5 轮、按钮箭头 6 轮、wordmark 2 轮。

### 10.2 字号问题先验证字号体系，不是改 CSS

用户反馈"字号大"时，第一反应不是改 CSS，而是 dev tools：
1. computed font-size 实际是多少？跟预期算出来的对不对？
2. html / body font-size 各是什么？（Lumia 的 20px 是首要嫌疑）
3. 用的字体加载没？（字体 404 退 Arial 会让宽度 +30%）

without 测量直接改 CSS = 蒙。

### 10.3 切片永远做平衡校验

写完 split-{name}-body.mjs 后，**强制跑一次 div/section 平衡校验**：
```js
for (const f of slices) {
  const opens = (f.match(/<div\b[^>]*>/g) || []).length;
  const closes = (f.match(/<\/div>/g) || []).length;
  if (opens !== closes) throw new Error(`${f.name}: ${opens} opens vs ${closes} closes`);
}
```
不平衡的切片在 Theme Editor 会报 HTML error，preview 也会因为 tag 错位错乱。

### 10.4 共享组件用 bare class，section 用前缀作用域

- Section 级 CSS：`.{prefix}-{section}__elem` 前缀作用域，跟 Dawn 不撞
- 共享组件 CSS（base-button / cursor / floating-nav / cookies）：**用 bare class**`.base-button`，**不要**`.{prefix}-page .base-button`——因为共享组件经常浮在 body 上（modal/cursor），不在 `.{prefix}-page` wrapper 内

### 10.5 不要 curl 当 preview 诊断

curl 不带 preview cookie 时拿到的是 live theme HTML，不是 preview。永远用浏览器开 preview URL + dev tools。如果一定要程序化抓 preview，用 Shopify CLI 的 `theme preview` 或带上 storefront access token。

### 10.6 维护规则

- **每修一个非显而易见的 bug** → 回填 [BUGS.md](BUGS.md)
- **新动效模式实现后** → 回填 [03-animations.md](03-animations.md)
- **发现新的 Dawn/Lumia 干扰** → 补到 [02-css-cascade.md](02-css-cascade.md)
- **新踩坑/新教训** → 补到本文档相应章节

---

## 11. 文件清单

| 文件 | 内容 |
|---|---|
| [README.md](README.md) | 入口索引 + 元教训速记 |
| [01-architecture.md](01-architecture.md) | 技术选型、目录结构、设计 token、字体 |
| [02-css-cascade.md](02-css-cascade.md) | Dawn/Lumia 全局污染、specificity 规则 |
| [03-animations.md](03-animations.md) | transition/animation 取舍、scroll-driven、GSAP↔CSS 映射 |
| [04-shopify-deploy.md](04-shopify-deploy.md) | Liquid 上限、schema vs index.json、CLI 工作流 |
| [05-checklist.md](05-checklist.md) | 每个 feature 必走的 7 步验证 |
| [06-section-conventions.md](06-section-conventions.md) | HTML/CSS/JS 素材 → section 转写规范 |
| [07-html-to-section-prompt.md](07-html-to-section-prompt.md) | 外部 AI 用的自包含转换提示词 |
| [BUGS.md](BUGS.md) | 已修 bug 目录，含症状/根因/修法/教训 |

---

## 12. 项目实战速记

### fluidglass-theme（Nuxt SSG，Phase 1 失败 → Phase 2 成功）

- 源站 `fluid.glass`，整站 Nuxt SSG，873KB `__NUXT_DATA__`
- Phase 1 单 Liquid 嵌入：撞 `unctx` 全局上下文冲突，hydration 失败，**不可救**
- Phase 2 拆 10 个 fg-* section：1:1 复刻成功

### ffd-theme（Nuxt SSG，Phase 2）

- 源站 `fluid-folding-door`，10 个 section
- 部署在 himaxlimited.myshopify.com，theme ID 154882703533
- 关键踩坑：ajax-scroll 把 section 拆占位符（P3-04）、Lenis 在 Lumia 上失效（P3-05）、rem 单位被 Dawn 干预（P3-02）

### more-nutrition-theme（Webflow + Vercel Next.js 层，Phase 2）

- 源站 `morematcha.vercel.app`，Webflow 站 + 自定义 Vercel Next.js 层
- 部署在 himaxlimited.myshopify.com，theme ID 154912456877
- 关键踩坑：
  - GemPages/Lumia 派生上 `template == 'index'` 失效，换 `request.page_type == 'index'` 才稳
  - 切片把 `.footer-bridge` / `.marquee` 等 child 切成 sibling，DOM 嵌套断了
  - 扁平化资产时漏了 CSS 内 `url(../...)` 引用 → 字体 404 → 退 Arial → "字号偏大"
  - Lumia `html { font-size: 20px }` 让 11rem 的 marquee 文字大 25%
