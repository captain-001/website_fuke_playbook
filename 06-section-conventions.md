# 06 — Section 代码规范

把一段独立的 HTML+CSS+JS(素材库 / codepen / 抓取的网页)转写成 fluidglass section 的标准规范。配套的对外 AI 提示词在 [07-html-to-section-prompt.md](07-html-to-section-prompt.md)。

> 前置阅读:[04-shopify-deploy.md](04-shopify-deploy.md) §3 schema 结构。本文聚焦"代码转换",04 聚焦"部署机制"。

---

## 1. Section vs Block 划分

| 概念 | 角色 | 在 admin 表现 |
|---|---|---|
| **Section** | 一个独立 UI 单元(Hero、Featured Collection、Domain Grid 容器) | 老板可拖到 templates/index.json 的卡片 |
| **Block** | Section 内的可重复条目(每张卡、每个 FAQ 项、每个 CTA) | Section 设置面板里"添加 block"出来的子项 |

**判断规则**:如果原代码里有"列表式重复 N 个相似元素",且每个元素的内容/链接应该可配置 → 那 N 个就是 blocks,外层是 section。
单个不重复 UI(如 Hero、Banner、CTA)就是纯 section,无 blocks。

## 2. 文件结构

每个 section 文件四段式,顺序固定:

```liquid
<!-- 1. HTML + Liquid 模板 -->
<svg class="defs">...</svg>           <!-- 可选: SVG sprite -->
<div class="fg-<name>" style="--var: {{ ... }};">
  {%- ... -%}
</div>

{% stylesheet %}                      <!-- 2. 局部 CSS -->
  .fg-<name> { ... }
{% endstylesheet %}

{% javascript %}                      <!-- 3. 局部 JS,可选 -->
  (function () { ... })();
{% endjavascript %}

{% schema %}                          <!-- 4. 配置 schema -->
{ ... }
{% endschema %}
```

## 3. 必删清单(粘贴素材代码后第一件事)

- ❌ `* { box-sizing: ... }` — 影响全站,Dawn 已处理
- ❌ `html, body { margin/padding/background/font: ... }` — 覆盖整个主题
- ❌ `:root { --xxx: ... }` — 全局变量泄漏,移到 `.fg-<name>` 作用域内
- ❌ 外部资源:`<link rel="stylesheet">`、`<script src="https://...">`、`@import`(字体例外但优先用 asset)
- ❌ `position: fixed` 的全屏遮罩(除非确实是 modal/overlay,需标注用途)
- ❌ JS 中 `document.body.style.xxx = ...` 等全局副作用

## 4. 必改清单

| 原代码 | 改成 |
|---|---|
| `<img src="path/to.jpg">` | `image_picker` setting + `{{ s.x \| image_url: width: N \| image_tag: loading: 'lazy', alt: ..., widths: '600, 1200, 2400', sizes: '100vw' }}` |
| 写死的 N 个相似元素 | `{% for b in section.blocks %}` 循环,内容来自 `block.settings.xxx` |
| `<h2>Hardcoded Title</h2>` | `<h2>{{ section.settings.heading \| escape }}</h2>` |
| `<a href="/products/foo">` | `<a href="{{ block.settings.link }}">` |
| `<style>` 标签 | `{% stylesheet %}` 包裹(去掉 `<style>` 本身) |
| `<script>` 标签 | `{% javascript %}` 包裹 |
| `id="arrow-right"` 通用 id | `id="fg-<name>-arrow"` 加 section 前缀防撞 |
| `.card` / `.btn` / `.grid` 通用 class | `.fg-<name>__card` 等 BEM |

## 5. 必加清单

- ✅ 外层 wrapper class `fg-<feature>`(命名见 §7)
- ✅ schema 顶部 `"tag": "section", "class": "fg-<feature>"` — Shopify 用这个生成外层标签
- ✅ 每个 block 的根元素加 `{{ block.shopify_attributes }}` — theme editor 实时预览需要
- ✅ 所有 settings 字符串输出加 `| escape`
- ✅ 可空字段用 `{%- if x != blank -%}` 包裹 — 老板留空不渲染垃圾标签
- ✅ schema 末尾 `presets: [{ name, blocks?: [...] }]` — 缺它老板加不了 section
- ✅ `{%- ... -%}` 而不是 `{% ... %}` — trim 空白

## 6. schema 字段速查

| 用途 | type | 必填属性 |
|---|---|---|
| 单行文本 | `text` | id, label(可选 default) |
| 多行(渲染时 `\| newline_to_br`) | `textarea` | id, label |
| 富文本(块级) | `richtext` | id, label |
| 行内富文本(`<br><b><em>`) | `inline_richtext` | id, label |
| 图片 | `image_picker` | id, label |
| 视频 | `video` / `video_url` | id, label |
| 颜色 | `color` | id, label, default |
| 链接 | `url` | id, label |
| 数字滑块 | `range` | id, label, min, max, step, default |
| 下拉 | `select` | id, label, options[{value,label}], default |
| 开关 | `checkbox` | id, label, default |
| 商品 / 集合 / 文章 | `product` / `collection` / `article` | id, label |
| 视觉分组(只显示) | `header` | content |
| 字体(配 `\| font_face`) | `font_picker` | id, label, default |

## 7. 命名规范

| 元素 | 规则 | 示例 |
|---|---|---|
| section 文件名 | `fg-<feature>.liquid` | `fg-domain-grid.liquid` |
| 外层 wrapper class | `fg-<feature>` | `.fg-domain-grid` |
| schema `class` 字段 | 同 wrapper class | `"class": "fg-domain-grid"` |
| 内部元素 class | `fg-<feature>__<elem>` (BEM) | `.fg-domain-grid__card` |
| 修饰符 | `fg-<feature>__<elem>--<mod>` | `.fg-domain-grid__card--featured` |
| asset 文件名 | `fg-` 前缀 | `fg-domain-grid.css` |
| SVG sprite id | `fg-<feature>-<icon>` | `id="fg-domain-grid-arrow"` |
| JS 幂等标志 | `window.__fg<Camel>Init` | `window.__fgDomainGridInit` |

理由:`fg-` 前缀保证零冲突。**别用通用名 `.card` / `.btn` / `.grid`**,会和 Dawn 撞。

## 8. 动态值桥接(schema → CSS)

`{% stylesheet %}` **不支持 Liquid 插值**。schema 里 `color` / `range` 等可调字段必须走 **wrapper inline style 注入 CSS 变量**:

```liquid
<div class="fg-<name>"
  style="
    --bg: {{ section.settings.bg_color }};
    --opacity: {{ section.settings.opacity | divided_by: 100.0 }};
    --speed: {{ section.settings.speed }}s;
  ">

{% stylesheet %}
  .fg-<name> { background: var(--bg, #001060); }
  .fg-<name>__card { opacity: var(--opacity, 0.1); }
  .fg-<name>__anim { transition-duration: var(--speed, 0.3s); }
{% endstylesheet %}
```

多实例无冲突:每个 section 的 wrapper 独立挂 inline style。

## 9. 图片资源处理

```liquid
<!-- 带 srcset 自适应,标配 -->
{{
  section.settings.image
  | image_url: width: 2400
  | image_tag:
    loading: 'lazy',
    widths: '600, 900, 1200, 1800, 2400',
    sizes: '100vw',
    alt: section.settings.heading,
    class: 'fg-<name>__img'
}}

<!-- bundled asset 兜底(开发期没图也能看效果) -->
{%- if section.settings.image != blank -%}
  {{ section.settings.image | image_url: ... | image_tag: ... }}
{%- else -%}
  <img src="{{ 'fg-<name>-fallback.jpg' | asset_url }}" alt="..." loading="lazy">
{%- endif -%}
```

参考 [04-shopify-deploy.md §3.2](04-shopify-deploy.md) 的 `fallback_image_filename` 模式。

## 10. JS 幂等保护

`{% javascript %}` 在每个使用该 section 的页面执行一次,**section 重新渲染时也会再跑**(Theme Editor 编辑时频繁发生)。必须幂等:

```js
{% javascript %}
  (function () {
    if (window.__fg<Name>Init) return;
    window.__fg<Name>Init = true;
    // ... 实际逻辑
    // 元素绑定用 document.querySelectorAll('.fg-<name>'),允许多实例
  })();
{% endjavascript %}
```

如果需要 per-instance 状态(每个 section 独立),改成 `[data-fg-init]` 属性标记:

```js
document.querySelectorAll('.fg-<name>:not([data-fg-init])').forEach(el => {
  el.dataset.fgInit = '1';
  // ... 初始化此实例
});
```

## 11. i18n 翻译(可跳过)

**fluidglass 单语言项目,直接写中文字面文本**:

```liquid
<button>了解更多</button>          <!-- ✓ 直接写 -->
<button>{{ 'fg.cta' | t }}</button>  <!-- ✗ 不必要的间接层 -->
```

如果未来要做多语言,再补 `locales/zh-CN.json`。

## 12. 转换工作流(对内)

1. 把素材代码丢给 AI(配 [07-html-to-section-prompt.md](07-html-to-section-prompt.md))
2. AI 输出 `.liquid` 文件,粘贴到 `sections/fg-<name>.liquid`
3. 走 [05-checklist.md](05-checklist.md) 验证
4. `shopify theme push --theme=FluidGlass-Phase2 --json` 部署
5. preview URL 实测
6. 出问题查 [BUGS.md](BUGS.md),修完回填

## 13. 常用 Liquid filter cheatsheet

| filter | 用途 | 示例 |
|---|---|---|
| `escape` | HTML 转义(防 XSS) | `{{ s.title \| escape }}` |
| `money` | 按店铺货币格式化 | `{{ product.price \| money }}` |
| `newline_to_br` | `\n` → `<br>` | `{{ s.heading \| newline_to_br }}` |
| `image_url: width: N` | CDN 缩放图片 | `{{ s.img \| image_url: width: 1200 }}` |
| `image_tag: ...` | 生成响应式 `<img>` | 见 §9 |
| `asset_url` | 引用主题 assets/ 文件 | `{{ 'fg-base.css' \| asset_url }}` |
| `default: x` | 空值兜底 | `{{ s.title \| default: '默认标题' }}` |
| `divided_by: x` | 数学运算(渲染 CSS var 用) | `{{ s.opacity \| divided_by: 100.0 }}` |
| `append: x` | 字符串拼接 | `{{ product.id \| append: '-form' }}` |
| `t` | 翻译(本项目不用) | `{{ 'foo.bar' \| t }}` |
