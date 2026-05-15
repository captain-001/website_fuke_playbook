# 04 — Shopify Liquid / 部署 / Schema

## 1. CLI 工作流

### 1.1 首次接入

```bash
# 在主题目录下
cd /path/to/fluidglass-theme/

# 浏览器登录
shopify theme dev --store=himaxlimited.myshopify.com
# 或：shopify auth login

# 推送到未发布主题（首次会问名字）
shopify theme push --unpublished
```

### 1.2 后续迭代

```bash
# 推送到指定主题（不要每次都创建新的）
shopify theme push --theme=FluidGlass-Phase2 --json
```

`--json` 会输出 `preview_url`：
```
https://himaxlimited.myshopify.com?preview_theme_id=154742259885
```

把这个 URL 发给用户测试。

### 1.3 PowerShell 注意事项

- `--nodelete=$false` PowerShell 会展开成 `--nodelete=False`，CLI 报错。**别写这个 flag**，需要不删时用 `--nodelete`，要删（默认）就不带
- `Compress-Archive` 写 zip 时分隔符是反斜杠，Shopify 不识别，**不要手工打包上传**，永远用 `shopify theme push`

## 2. Liquid 文件大小上限

**单个 Liquid 文件 ≤ 256KB**。

Phase 1 内联 873KB 的 Nuxt `__NUXT_DATA__` JSON 时撞墙，拆 4 个 chunk 再用 `{%- render -%}` 拼。

### 拼接 chunk 时的关键

`{% render %}` 默认会在 render 周围插入换行符，破坏内联 JSON。**必须用 `{%- render -%}`**，双侧 trim 空白：

```liquid
<!-- ✗ 会插入换行，破坏 JSON.parse -->
{% render 'data-chunk-1' %}{% render 'data-chunk-2' %}

<!-- ✓ trim 空白，JSON 连续 -->
{%- render 'data-chunk-1' -%}{%- render 'data-chunk-2' -%}
```

Phase 2 不再需要这种内联大数据，但**任何 inline JSON / SVG path 数据** 拼接都遵守同样规则。

## 3. Section schema 设计

### 3.1 settings + blocks + presets 三件套

```liquid
{% schema %}
{
  "name": "FG Banner CTA",
  "tag": "section",
  "class": "fg-banner-cta-section",
  "settings": [
    { "type": "text",     "id": "section_title", "label": "Section label", "default": "..." },
    { "type": "textarea", "id": "heading",       "label": "Heading", "default": "..." },
    { "type": "image_picker", "id": "background_image", "label": "Background image" },
    { "type": "text",         "id": "fallback_image_filename", "label": "Bundled fallback filename",
      "info": "Filename inside theme /assets/", "default": "fg-banner-cta.jpg" }
  ],
  "blocks": [
    {
      "type": "project",
      "name": "Project",
      "settings": [
        { "type": "text", "id": "title", "label": "Project name" },
        { "type": "url",  "id": "link",  "label": "Link" },
        { "type": "image_picker", "id": "image", "label": "Image (overrides bundled fallback)" },
        { "type": "text",         "id": "fallback_image_filename", "label": "Bundled fallback filename" }
      ]
    }
  ],
  "presets": [
    {
      "name": "FG Banner CTA",
      "blocks": [
        { "type": "project", "settings": { "title": "Sample 1", "fallback_image_filename": "fg-project-1.jpg" } }
      ]
    }
  ]
}
{% endschema %}
```

### 3.2 fallback_image_filename 模式

每个有图片的 block 都提供两个字段：
- `image` (image_picker)：商家在 Theme Editor 上传，优先使用
- `fallback_image_filename` (text)：bundled 资源文件名，开箱即用的兜底

```liquid
{%- if b.image != blank -%}
  {{ b.image | image_url: width: 1200 | image_tag: class: 'base-image', loading: 'lazy', alt: b.title }}
{%- elsif b.fallback_image_filename != blank -%}
  <img class="base-image" loading="lazy" src="{{ b.fallback_image_filename | asset_url }}" alt="{{ b.title }}">
{%- endif -%}
```

理由：开发期商家还没上传图，但页面要看起来完整；上线后商家通过 Theme Editor 替换。

### 3.3 schema default vs templates/index.json

⚠️ 改了 schema 的 `default` 字段，**对已存在的 section 实例不生效**。已存在实例的设置存在 `templates/*.json`：

```json
{
  "sections": {
    "fg_banner_showroom": {
      "type": "fg-banner-showroom",
      "settings": {
        "use_bundled_video": true,
        "fallback_image_filename": "fg-banner-showroom.jpg"
      }
    }
  }
}
```

**改动必须同步**：schema default 改了，记得把 templates/index.json 里的对应值也改。否则只有"新建 section"才用新 default。

## 4. 模板分发策略

```
templates/
├── index.json              # 自建 10 个 fg-* section
├── product.json            # 保留 Dawn 默认（不动）
├── collection.json         # 保留 Dawn 默认（不动）
├── cart.json               # 保留 Dawn 默认（不动）
└── ...                     # 其他都保留 Dawn
```

只在 index 上"反客为主"，其他模板让 Dawn 正常工作。这样购物链路、结账、产品页等 Shopify 核心功能不受影响。

## 5. layout/theme.liquid 改动清单

最小改动量原则。我们只在 `theme.liquid` 做了这些：

```liquid
<!-- 隐藏 Dawn 头：index 上不渲染，其他模板正常 -->
{%- unless template.name == 'index' -%}
  {% sections 'header-group' %}
{%- endunless -%}

<main id="MainContent" class="content-for-layout focus-none" role="main" tabindex="-1">
  {{ content_for_layout }}
</main>

<!-- 隐藏 Dawn 尾：同上 -->
{%- unless template.name == 'index' -%}
  {% sections 'footer-group' %}
{%- endunless -%}

<!-- 全局浮动 nav + 菜单，每页都渲染 -->
{{ 'fg-base.css' | asset_url | stylesheet_tag }}
{%- render 'fg-floating-nav' -%}
```

**不要动**：fonts preload、Shopify scripts、cart 抽屉等 Dawn 自带逻辑。

## 6. assets/ 命名

- 所有自创资产文件加 `fg-` 前缀（`fg-base.css`, `fg-scroll-progress.js`, `fg-hero-bg.mp4`, `fg-project-doors.jpg`）
- 字体保留原名（带 hash 后缀），方便从原站拷贝
- 不要覆盖 Dawn 自带的 `base.css` / `global.js` / `animations.js` 等

## 7. 引用资源

CSS 引用：
```liquid
{{ 'fg-base.css' | asset_url | stylesheet_tag }}
```

JS 引用（defer 必加）：
```liquid
<script src="{{ 'fg-scroll-progress.js' | asset_url }}" defer></script>
```

⚠️ **不要在多个 section 重复引同一个 JS 而不做幂等保护**。Shopify 把同名 src 的 `<script>` 标签全部执行，会重复 init。我们的 JS 用：
```js
(function () {
  if (window.__fgScrollInit) return;
  window.__fgScrollInit = true;
  // ... 实际逻辑
})();
```

每个独立子系统（cursor、scroll-progress、reveal）各自挂一个 `__fg{Name}Init` 标志。

## 8. Theme Editor 跟 Pure preview 行为差异

某些 JS 在 Theme Editor 里被 Shopify 框架包装后行为不同：
- **IntersectionObserver** 可能延迟触发
- **mousemove / scroll** 事件可能被吃掉
- **section reload** 会重跑 init，确保幂等

测动效**优先用 pure preview URL** (`?preview_theme_id=N`)，不要在 editor 里测。

## 9. 视频 / 大文件

- mp4 直接放 `assets/`，用 `asset_url` 引
- 文件大小限制：Shopify 单文件 20MB（实测），超了上传失败
- 我们的 `fg-hero-bg.mp4` 8.3MB，`fg-showroom-bg.mp4` 6.1MB，安全
- 商家可以在 Theme Editor 用 `text` 字段填 CDN URL 覆盖 bundled mp4

## 10. 502 / 推送失败

`shopify theme push` 偶尔返回 502。**重试一次**通常解决。如果 3 次都失败：
1. 检查文件是否超 256KB
2. 检查 Liquid 语法（用 `shopify theme check`）
3. 检查网络

## 11. 常用文件位置速查

| 想改什么 | 改哪 |
|---|---|
| 全站米色背景 | `assets/fg-base.css` body 规则 |
| 字体 / token / 共享按钮样式 | `assets/fg-base.css` |
| 滚动驱动逻辑 / cursor / line-reveal | `assets/fg-scroll-progress.js` |
| 浮动 nav + 菜单 | `snippets/fg-floating-nav.liquid` |
| 某个 section 的内容 / 样式 | `sections/fg-*.liquid` |
| 首页 section 顺序 / 默认值 | `templates/index.json` |
| 隐藏 Dawn UI / 全站 script 注入点 | `layout/theme.liquid` |
