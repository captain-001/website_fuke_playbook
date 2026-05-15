# 01 — 架构 / 设计系统

## 1. 立项决策矩阵

| 选项 | 适用 | 风险 / 教训 |
|---|---|---|
| **SPA 嵌入**（把 Nuxt 整站打成单一 Liquid） | 极简静态展示页、店铺**没装**任何 Vue/Nuxt 第三方 app | Phase 1 实测：Shopify 通过 `{{ content_for_header }}` 注入的 app script（cdn.slpht.com/shopify-embed.js、cdn.nfcube.com/instafeed-*.js 等）跟 Nuxt 的 `unctx` 全局上下文打架，hydration 报 `beforeEach undefined` 系列错误，**不可救** |
| **原生 Liquid sections**（每个区块拆成独立可编辑 section）| 默认选择 | 工作量大，需要逐个 section 复刻动效；动效 1:1 复刻有大量细节坑 |

**判断标准**：店铺装了任何第三方 Vue/Nuxt 嵌入式 app（评测、社交 feed、个性化推荐等）→ 一律走 Phase 2。立项时先问商家「店铺装了哪些 app」。

## 2. 文件结构

```
fluidglass-theme/                  # Shopify 主题根
├── assets/
│   ├── fg-base.css                # 设计 token、字体、共享组件（base-button / base-title / cursor / line-reveal）
│   ├── fg-scroll-progress.js      # 单一全局 JS：滚动 progress + cursor + reveal-by-line
│   ├── fg-*.{jpg,mp4,woff2}       # 全部按 fg- 前缀
│   └── (Dawn 自带文件不动)
├── layout/
│   └── theme.liquid               # 改了：index 模板上隐藏 Dawn header/footer-group，引入 fg-floating-nav
├── sections/
│   └── fg-{name}.liquid           # 全部 section 前缀 fg-，schema 自带 presets
├── snippets/
│   ├── fg-button-arrow.liquid     # 3 元素 SVG 箭头（L + chevron + diamond）
│   ├── fg-button-label.liquid     # 滚动文字 label（两层 line）
│   └── fg-floating-nav.liquid     # 持久浮动 nav + 菜单 modal
└── templates/
    ├── index.json                 # 首页 10 个 fg-* section + 顺序
    └── (其他模板保留 Dawn 默认)
```

参考文件：
- 抓取得到的源材料在 `../007-fluidglass/`
- 原 Shopify 导出做参考用：`../theme_export__himaxlimited-...`（**只读**）

## 3. 设计 token（这套不可改）

```css
:root {
  --fg-size: 1600;
  --fg-font-s: calc((100vw / var(--fg-size)) * 10);

  --fg-color-black: #0b1012;
  --fg-color-white: #fff;
  --fg-color-cream: #f3f0ec;
  --fg-color-taupe: #d4cec6;
  --fg-color-grey:  #212325;

  --fg-ease-in-out-quart: cubic-bezier(.77, 0, .175, 1);
  --fg-ease-in-out-cubic: cubic-bezier(.645, .045, .355, 1);
  --fg-ease-out-quart:    cubic-bezier(.165, .84, .44, 1);
  --fg-ease-out-cubic:    cubic-bezier(.215, .61, .355, 1);

  --fg-font-mono: 'Aeonik Mono', ui-monospace, Menlo, Consolas, monospace;
  --fg-font-pro:  'Aeonik Pro', system-ui, -apple-system, sans-serif;
}
@media (max-width: 600px) {
  :root { --fg-size: 375; }
}
```

### 3.1 单位规则

**所有原站 `Nrem` 数值都换算成 `calc(N * var(--fg-font-s))`**。原站根字号是 `html { font-size: var(--font-s) }`，等价于我们的 `--fg-font-s`。

⚠️ **GSAP 数值是像素，不是 rem**：
- 原站 `gsap.to(el, { x: c*65 })`，`c = width/1600` → 在 1600px 视口下 = **65 像素**
- 换算到 `--fg-font-s` 体系 = `6.5 * var(--fg-font-s)`（10× 缩小）
- 曾因为这条 showroom 文字滑入幅度大了 10 倍

### 3.2 字体

下载到 `assets/`，在 `fg-base.css` 用 `asset_url` 引：
```css
@font-face {
  font-display: swap;
  font-family: 'Aeonik Pro';
  font-weight: 400;
  src: url('aeonik-regular.CDaMS559.woff2') format('woff2'),
       url('aeonik-regular.DSAxWCt1.woff') format('woff');
}
/* 同样定义 weight: 700 的 Aeonik Pro，weight: 500/600 的 Aeonik Mono */
```

⚠️ 任何需要测量文字位置的 JS（行拆分、宽度自适应）**必须等 `document.fonts.ready`**，否则按系统字体算位置，加载完 Aeonik 后会偏。

## 4. Section 命名规则

| Section 文件 | class 前缀 | section ID |
|---|---|---|
| `fg-home-header.liquid` | `.fg-section.fg-home-header` | `fg-home-header-{{ section.id }}` |
| `fg-banner-cta.liquid` | `.fg-section.fg-banner-cta` | … |

**双 class 必备**：`fg-section` 提供 reset（box-sizing、margin: 0、color、font-family）；具体 class 提供 section 特有样式。

## 5. 必备 snippets

- `fg-button-arrow.liquid` — 3 元素 SVG：
  - `<path class="arrow-line">` L 形线
  - `<path class="arrow-chevron">` 右端 `>` 小尖（hover 时跟 line 一起擦除）
  - `<rect class="arrow-diamond">` 旋转方块（hover 时画入）
- `fg-button-label.liquid` — 两层 `<span class="line">` 实现 hover 文字上滚

按钮变体（在 fg-base.css）：
- `.base-button.is-black` 黑底白字
- `.base-button.is-white` 半透白底白字
- `.base-button.is-alpha` 透明底

## 6. 全局视觉

- 整页背景：`body { background: var(--fg-color-cream) !important }` —— `!important` 必加，详见 [02-css-cascade.md §1](02-css-cascade.md)
- 鼠标 cursor：全局单一 `.fg-cursor` 元素 + `[data-fg-cursor="Label"]` 触发器，详见 [03-animations.md §6](03-animations.md)
- 浮动底部 nav：`fg-floating-nav.liquid` snippet 在 layout 全站渲染
