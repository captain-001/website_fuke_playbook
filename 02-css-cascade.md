# 02 — CSS Cascade 与 Dawn 干扰

## 0. 核心原则

Dawn 主题在 `assets/base.css` 里塞了几百条全局规则。**任何无前缀的 class 都可能被它污染**。规范：

1. **所有自创 class 一律加 `fg-` 前缀**
2. **section 内规则一律用嵌套选择器**（`.fg-banner-showroom .heading`，不是 `.heading`）
3. **不确定时打开 DevTools → Computed 看实际生效值**

## 1. body 背景必须 !important

```css
/* Dawn 的规则（specificity 0,1,0）：
   body.color-background-1 { background-color: rgb(var(--color-background)) }
   会盖过我们的 body { background: cream } （specificity 0,0,1） */

/* 正确写法：*/
body,
body.gradient {
  background: var(--fg-color-cream) !important;
  background-color: var(--fg-color-cream) !important;
}
```

为什么 `background` 和 `background-color` 都写：Dawn 用的是后者，速记 `background` 不一定能压住。
为什么 `body.gradient` 也写：Dawn 给 body 加了 `.gradient` class，是其全站样式钩子。

## 2. 命名地雷清单

以下 class 名 **绝对不要单独使用**（无前缀），Dawn 都占了：

| 受污染 class | Dawn 干了什么 | 后果 |
|---|---|---|
| `.button` | padding, min-height, box-shadow, font-family, line-height, border-radius… | 按钮变形 |
| `.gradient` | 给 body 加的钩子 class | 你的 .gradient 元素被联动 |
| `.field` | input/textarea 包装 | 输入框变形 |
| `.media` | 图片容器 padding & aspect | 图片变形 |
| `.card`, `.icon`, `.image`, `.title`, `.subtitle` | 各种全局样式 | 不可预测 |

**安全做法**：
- 用 `.fg-arrow-btn` 而不是 `.button`
- 用 `.fg-nav-title` 而不是 `.title`
- 用 `.fg-menu-cta` 而不是 `.cta`

## 3. 嵌套元素的命名

section 内部 class 不必都加 `fg-` 前缀，但要靠**父级 section class** 形成作用域：

```css
/* 安全：被 .fg-banner-cta 作用域包住 */
.fg-banner-cta .container { ... }
.fg-banner-cta .heading { ... }

/* 不安全：全局 .container .heading */
.container .heading { ... }
```

⚠️ 例外：跨 section 复用的（如 `.base-heading`, `.base-button`, `.base-title`）放在 `fg-base.css`，用 `.fg-section .base-button` 作用域。

## 4. 强制重置 Dawn 污染（按钮 / input 必备）

任何自创的 `<button>` 或 `<input>`，**显式重置**所有可能被污染的属性：

```css
.fg-arrow-btn {
  appearance: none;
  -webkit-appearance: none;
  padding: 0;
  margin: 0;
  border: 0;
  border-radius: 0;
  box-shadow: none;
  background: none;
  font: inherit;
  letter-spacing: 0;
  min-height: 0;
  min-width: 0;
  /* 然后才写你想要的样式 */
  width: ...;
  height: ...;
  ...
}
```

或者在 `<button>` 上加 `!important` 反复抹掉：`background: none !important; border: 0 !important; padding: 0 !important;`

## 5. Section bundle CSS 加载顺序

Shopify 把所有 section 的 `{% stylesheet %}` 合并到一个 bundle CSS，跟 `{{ 'fg-base.css' | asset_url | stylesheet_tag }}` 的 `<link>` **加载顺序不保证**。

后果：相同 specificity 的规则，谁后加载谁赢。在不同环境（pure preview vs editor）可能不一样。

**对策**：永远不要靠加载顺序赢 specificity。section 里的样式如果要覆盖 base，**用更高 specificity**：

```css
/* fg-base.css */
.fg-section { color: var(--fg-color-grey); }

/* fg-banner-showroom 的 section stylesheet */
/* ✗ 错（同 specificity 0,1,0）：*/
.fg-banner-showroom { color: var(--fg-color-white); }

/* ✓ 对（specificity 0,2,0）：*/
.fg-banner-showroom .base-heading,
.fg-banner-showroom .base-title,
.fg-banner-showroom .text {
  color: var(--fg-color-white);
}
```

## 6. `currentColor` 陷阱

```css
/* 不可靠：当 cascade 复杂时，currentColor 可能解析为父级灰色 */
.fg-banner-showroom .text {
  color: color-mix(in srgb, currentColor 80%, transparent);
}

/* 可靠：写死变量 */
.fg-banner-showroom .text {
  color: color-mix(in srgb, var(--fg-color-white) 80%, transparent);
}
```

## 7. `overflow` + `position: sticky` 冲突

```css
/* ✗ sticky 失效 */
.parent { overflow: hidden; }
.parent .sticky-child { position: sticky; top: 0; }

/* ✓ sticky 正常 */
.parent { overflow: clip; }
.parent .sticky-child { position: sticky; top: 0; }
```

`overflow: clip` 不创建滚动容器，sticky 跳过它继续找祖先；`overflow: hidden` 创建了，sticky 卡住。
浏览器支持：Chrome 90+, Safari 16+, Firefox 81+（2026 受众 OK）。

## 8. flex 三槽位"假居中"

```html
<!-- ✗ 不真居中：title 在 logo 和 burger 之间的剩余空间居中，
     如果左右槽位视觉重心不均，title 会偏 -->
<div style="display: flex">
  <div class="logo" style="width: 5rem">...</div>
  <div class="title" style="flex: 1; text-align: center">HOME</div>
  <div class="burger" style="width: 5rem">...</div>
</div>

<!-- ✓ 真居中：side 槽 absolute 贴边，title absolute inset:0 + flex center -->
<div style="position: relative">
  <div class="logo" style="position: absolute; left: 0; ...">...</div>
  <div class="title" style="position: absolute; inset: 0;
       display: flex; align-items: center; justify-content: center">HOME</div>
  <div class="burger" style="position: absolute; right: 0; ...">...</div>
</div>
```

## 9. 隐藏 Dawn UI 局部覆盖

在 `layout/theme.liquid`：

```liquid
{%- unless template.name == 'index' -%}
  {% sections 'header-group' %}
{%- endunless -%}

{# ... main content ... #}

{%- unless template.name == 'index' -%}
  {% sections 'footer-group' %}
{%- endunless -%}
```

`header-group` 有 announcement bar / nav / cart icon；`footer-group` 有 newsletter / link list / payment icons。只在 index 上吃掉，其他模板（product / cart / collection）保留 Dawn 完整 UI，避免破坏 Shopify 默认链路。

## 10. `z-index: -1` 在 transform / isolation 容器内不可靠

经典坑：想做"内容在最上、背景在下"的层叠，给背景 `z-index: -1`，给父容器加 `isolation: isolate`（或父本身有 `transform`、`filter` 等创建 stacking context 的属性）。

**理论上** 子元素的 z-index: -1 留在父的 stacking context 里、只下沉到父背景层下方。

**实际上**：当父同时有 `transform` + `overflow: auto/hidden` + `isolation: isolate` 时，部分浏览器（Chromium 系尤其，跟 backdrop-filter 同时存在更易触发）会把 z-index: -1 子元素的渲染**裁掉**——元素的存在被推到"父背景层之下"，但父没显式背景，导致整个负 z-index 元素不可见。

**症状**：你写了 `<div class="bg">` 当背景层 + 内容元素叠在上面，DevTools 里所有元素都在，computed style 也对，但 `.bg` 看不见，内容像浮在透明面板上。

**修法二选一**：

- **把背景直接放到容器上**（最简单）。如果不需要分离的"背景层独立动画"，就别多分一层 div。
- **改用 `position: relative; z-index: 1` 提升内容**：背景元素留默认堆叠，内容元素提到正 z-index，自然盖在背景上。不用负 z-index，不用 `isolation`。

**反例**（菜单暗色背景看不见）：
```css
.fg-menu-panel { transform: translateX(-50%); overflow-y: auto; isolation: isolate; }
.fg-menu-bg    { position: absolute; inset: 0; background: black; z-index: -1; }
.fg-menu-title { /* 默认堆叠 */ }
```

**正例**：
```css
.fg-menu-panel {
  transform: translateX(-50%);
  background: color-mix(in srgb, var(--fg-color-black) 88%, transparent);
  backdrop-filter: blur(2rem);
}
/* 不要单独的 .fg-menu-bg 元素 */
```

## 11. 速查：Dawn cascade debug 流程

发现某元素样式不对时：
1. DevTools → Elements → 选中元素 → Computed 面板
2. 看实际 `background` / `color` / `padding` 的最终值
3. Computed 里点开任何属性 → 看它来自哪条规则
4. 如果是 Dawn 的 `.button` / `.media` / 类似的污染 → **重命名你的 class**
5. 如果是 specificity 不够 → **加更具体的选择器**或 `!important`
