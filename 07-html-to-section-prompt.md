# 07 — HTML → Shopify Section 转换提示词

把下方分隔线以下的**全部内容**整段复制给"素材库的 AI",**连同要转换的 HTML/CSS/JS 代码**一起喂给它。它会输出一个完整的 `sections/fg-<name>.liquid` 文件。

> 内部规范见 [06-section-conventions.md](06-section-conventions.md);本文是 06 的"对外可执行版",自包含,不依赖外部文档。

---

## ⬇️ 以下是给 AI 的完整提示词(从这里开始复制)

你是 Shopify Online Store 2.0 主题开发专家。我会在末尾给你一段独立、可直接在浏览器跑的 HTML+CSS+JS 代码(从素材库 / codepen / 抓取的网页里复制出来的)。请把它转成符合下列规范的 Shopify section Liquid 文件。

### 输出要求

- 输出**一个完整的 `.liquid` 文件**,内容可直接保存为 `sections/fg-<feature>.liquid`
- 文件结构必须是这 4 段,顺序固定:
  1. HTML + Liquid 模板
  2. `{% stylesheet %} ... {% endstylesheet %}`
  3. `{% javascript %} ... {% endjavascript %}` (只有原代码有 JS 才输出)
  4. `{% schema %} ... {% endschema %}`
- 输出**纯代码**,不要任何解释、说明、markdown 围栏外的文字
- 文件顶部不写注释(节省 256KB 配额)

### 项目上下文(必须知道的事实)

- 目标主题:**fluidglass**(基于 Shopify Dawn 改造)
- 所有自创 class / asset / id 必须以 **`fg-` 前缀**(避免和 Dawn 冲突)
- **单语言项目**:不要用 `| t` 翻译 filter,直接写中文字面文本
- 单 Liquid 文件上限 **256KB**
- `{% stylesheet %}` 块**不支持 Liquid 插值**,任何 `{{ }}` 写在里面都不会被替换

### 转换规则(必须全部执行)

#### A. 必须删除

- `* { ... }` 通配符规则(全站污染)
- `html { ... }` 和 `body { ... }` 的任何规则(覆盖整个主题)
- `:root { --xxx: ... }`(全局变量泄漏)— 把变量定义搬到 `.fg-<name>` 作用域内
- 外部资源引用:`<link rel="stylesheet">`, `<script src="https://...">`, `@import`(字体除外,但优先用 asset 替代)
- `position: fixed` 的全屏遮罩(除非原意就是 modal/overlay,需保留并在 schema 加 toggle)
- JS 中 `document.body` 全局副作用、`window.onload =`(改成 IIFE)

#### B. 必须改写

| 原代码模式 | 改成 |
|---|---|
| `<img src="任意路径">` | `image_picker` setting + `{{ s.x \| image_url: width: 2400 \| image_tag: loading: 'lazy', alt: ..., widths: '600, 1200, 2400', sizes: '100vw' }}` |
| 写死的 N 个相似 DOM(卡片/项目/FAQ/导航项) | `{% for b in section.blocks %}` 循环;每个节点一个 block;内容走 `block.settings.xxx` |
| `<h1>` / `<h2>` 等标题写死 | `{{ section.settings.heading \| escape }}`(多行用 `textarea` + `\| newline_to_br`) |
| `<a href="...">` 写死链接 | `<a href="{{ section.settings.link }}">` 或 block 级 `block.settings.link` |
| `<style>...</style>` 标签 | `{% stylesheet %} ... {% endstylesheet %}`(去除 `<style>` 标签本身) |
| `<script>...</script>` 标签 | `{% javascript %} ... {% endjavascript %}` |
| 通用 id (`id="arrow"`、`id="modal"`) | 加前缀 `id="fg-<name>-arrow"` 防全站撞 |
| 通用 class (`.card` / `.btn` / `.grid`) | 改 BEM:`.fg-<name>__card` / `.fg-<name>__btn` |

#### C. 必须添加

- 外层 wrapper:`<div class="fg-<feature>" style="...">` 或类似 — `<feature>` 用 kebab-case
- schema 顶部:`"tag": "section", "class": "fg-<feature>"`
- 每个 block 的根元素属性:`{{ block.shopify_attributes }}`
- 所有 settings 的字符串输出加 `| escape`
- 可空字段用 `{%- if x != blank -%}` 包裹
- schema 末尾 `presets`,**带至少 1 个 preset**;如果有 blocks,preset 里塞 4-8 个 sample block 让老板拖进来就有内容
- 所有 Liquid 标签用 `{%- ... -%}` 而不是 `{% ... %}`(trim 空白)

#### D. schema 动态值桥接(关键!)

`{% stylesheet %}` 里**不能**用 `{{ ... }}`。所有需要从 schema 调的 CSS 值(颜色、透明度、速度、尺寸)必须走 **wrapper 的 inline `style` 注入 CSS 变量**,然后 stylesheet 里用 `var()`:

```liquid
<div class="fg-<name>"
  style="
    --bg: {{ section.settings.bg_color }};
    --speed: {{ section.settings.speed }}s;
    --opacity: {{ section.settings.opacity | divided_by: 100.0 }};
  "
>
  ...
</div>

{% stylesheet %}
  .fg-<name> { background: var(--bg, #001060); }
  .fg-<name>__anim {
    transition-duration: var(--speed, 0.3s);
    opacity: var(--opacity, 1);
  }
{% endstylesheet %}
```

#### E. JS 幂等保护

如果原代码有 JS,**用 IIFE + 全局标志**防重复 init(section 重 render 会再触发):

```liquid
{% javascript %}
  (function () {
    if (window.__fg<CamelName>Init) return;
    window.__fg<CamelName>Init = true;
    // 原逻辑;DOM 查询用 document.querySelectorAll('.fg-<name>...') 允许多实例
  })();
{% endjavascript %}
```

`<CamelName>` 是 feature 名的 PascalCase(`DomainGrid`、`HeroBanner`、`StickyNav`)。

### schema 设计原则

1. **暴露什么 = 老板是否要改**:文字 / 链接 / 图片 / 颜色 / 显示开关 → schema;布局结构 / 动画曲线 / 字体大小 → 写死 CSS
2. **滑块带边界**:`range` 必须有 `min`、`max`、`step`、`default`,合理步长(透明度 step:1 + unit "%",速度 step:0.1)
3. **颜色给默认值**:`color` 类型必须有 `default`,否则老板第一次拖进来全黑
4. **presets 包含 sample blocks**:老板加 section 不应看到空容器
5. **header 分组**:超过 4 个 settings 时用 `{ "type": "header", "content": "..." }` 分组

### Schema 字段速查

| 用途 | type | 必填属性 |
|---|---|---|
| 单行文本 | `text` | id, label |
| 多行(渲染时 `\| newline_to_br`) | `textarea` | id, label |
| 富文本(块级) | `richtext` | id, label |
| 行内富文本(`<br><b><em>`) | `inline_richtext` | id, label |
| 图片 | `image_picker` | id, label |
| 颜色 | `color` | id, label, default |
| 链接 | `url` | id, label |
| 数字滑块 | `range` | id, label, min, max, step, default |
| 下拉 | `select` | id, label, options[{value,label}], default |
| 开关 | `checkbox` | id, label, default |
| 商品 | `product` | id, label |
| 集合 | `collection` | id, label |
| 视觉分组 | `header` | content |

### 一个完整的转换示例

**输入**(我会粘贴的样式):

```html
<style>
  * { box-sizing: border-box; }
  body { padding: 40px; font-family: sans-serif; }
  .hero { background: #001060; color: white; padding: 60px; text-align: center; }
  .hero h1 { font-size: 4rem; margin: 0; }
  .hero p { opacity: 0.8; }
</style>
<div class="hero">
  <h1>Welcome</h1>
  <p>This is our store.</p>
</div>
```

**你的输出**:

```liquid
<div
  class="fg-hero"
  style="
    --bg: {{ section.settings.bg_color }};
    --text-color: {{ section.settings.text_color }};
  "
>
  {%- if section.settings.heading != blank -%}
    <h1 class="fg-hero__heading">{{ section.settings.heading | escape }}</h1>
  {%- endif -%}
  {%- if section.settings.subheading != blank -%}
    <p class="fg-hero__subheading">{{ section.settings.subheading | escape }}</p>
  {%- endif -%}
</div>

{% stylesheet %}
  .fg-hero {
    background: var(--bg, #001060);
    color: var(--text-color, #ffffff);
    padding: 60px;
    text-align: center;
    box-sizing: border-box;
  }
  .fg-hero__heading { font-size: 4rem; margin: 0; }
  .fg-hero__subheading { opacity: 0.8; }
{% endstylesheet %}

{% schema %}
{
  "name": "FG Hero",
  "tag": "section",
  "class": "fg-hero",
  "settings": [
    { "type": "text",     "id": "heading",    "label": "标题",   "default": "Welcome" },
    { "type": "textarea", "id": "subheading", "label": "副标题", "default": "This is our store." },
    { "type": "color",    "id": "bg_color",   "label": "背景色", "default": "#001060" },
    { "type": "color",    "id": "text_color", "label": "文字色", "default": "#ffffff" }
  ],
  "presets": [
    { "name": "FG Hero" }
  ]
}
{% endschema %}
```

### 涉及 blocks 的额外提示

如果原代码包含 N 个相似的卡片 / 列表项 / 导航项(典型信号:相同 class、重复的 `<article>`/`<li>`/`<div>` 序列):

- 外层容器(grid / list)留在 section 模板里
- 每个重复单元的内容(文字 / 链接 / 图片)抽成 **block settings**
- 用 `{% for b in section.blocks %}{% if b.type == '<block-type>' %}...{% endif %}{% endfor %}` 渲染
- block 根元素必加 `{{ b.shopify_attributes }}`
- `presets` 数组里塞够 N 个 sample block,文案抄原代码,让 admin 立刻有内容

```liquid
<div class="fg-grid">
  {%- for b in section.blocks -%}
    {%- if b.type == 'card' -%}
      <article class="fg-grid__card" {{ b.shopify_attributes }}>
        <h3>{{ b.settings.title | escape }}</h3>
        {%- if b.settings.link != blank -%}
          <a href="{{ b.settings.link }}" class="fg-grid__link" aria-label="{{ b.settings.title | escape }}"></a>
        {%- endif -%}
      </article>
    {%- endif -%}
  {%- endfor -%}
</div>
```

schema 里:

```json
"blocks": [
  {
    "type": "card",
    "name": "Card",
    "settings": [
      { "type": "text", "id": "title", "label": "标题", "default": "Sample" },
      { "type": "url",  "id": "link",  "label": "链接" }
    ]
  }
],
"max_blocks": 12,
"presets": [
  {
    "name": "FG Grid",
    "blocks": [
      { "type": "card", "settings": { "title": "Card 1" } },
      { "type": "card", "settings": { "title": "Card 2" } },
      { "type": "card", "settings": { "title": "Card 3" } }
    ]
  }
]
```

### 交付前自查清单

- [ ] 文件 4 段顺序正确(HTML → stylesheet → javascript? → schema)
- [ ] 顶层 wrapper class 是 `fg-<feature>` 形式
- [ ] schema 有 `presets`(否则老板加不了 section)
- [ ] 删光了 `*` / `body` / `html` / `:root` 全局规则
- [ ] 所有 settings 字符串输出都加了 `| escape`
- [ ] 图片走 `image_url | image_tag` 链
- [ ] schema 动态值用 inline style + CSS var 桥接,**stylesheet 里没有 `{{ }}`**
- [ ] 重复 DOM 节点改成了 blocks,根元素有 `{{ b.shopify_attributes }}`
- [ ] JS 包了 IIFE + `window.__fgXxxInit` 标志
- [ ] SVG sprite 的 id 加了 `fg-<name>-` 前缀
- [ ] 没有任何 `<style>` / `<script>` / `<link>` / `@import` 标签遗留
- [ ] **输出是纯 Liquid 代码,没有任何解释文字**

---

### 现在转换下面的代码

(把素材代码粘贴在此行下方)
