# Shopify 网页移植 Playbook

把高端动效网站（Nuxt SSG / Webflow / 静态站）移植成 Shopify 原生 Liquid theme 的流程规范。

**先读 [00-MASTER.md](00-MASTER.md)** —— 入口总览，包含两个真实项目（fluidglass、more-nutrition）的全部踩坑沉淀。

源自：
- 2026-05 fluid.glass → himaxlimited.myshopify.com（`../fluidglass-theme/`，源材料 `../007-fluidglass/`）
- 2026-05-14 fluid-folding-door → himaxlimited.myshopify.com（`../ffd-theme/`，源材料 `../designs/010-fluid-folding-door/`）
- 2026-05-15 morematcha.vercel.app → himaxlimited.myshopify.com（`../more-nutrition-theme/`，源材料 `../designs/013-more-nutrition/`）

---

## 这是写给谁的

- 接手类似项目的开发者
- 用 AI 助手做类似工作的同事 —— **AI 读完这套文档应该能 1:1 复刻**

## 如何使用

1. **新项目立项**：先读 [01-architecture.md](01-architecture.md) 决定 SPA 嵌入 还是 Liquid sections
2. **写样式时被 Dawn 干扰**：查 [02-css-cascade.md](02-css-cascade.md)
3. **把 GSAP 动效搬到 CSS**：查 [03-animations.md](03-animations.md)
4. **配置 / 部署 / schema 问题**：查 [04-shopify-deploy.md](04-shopify-deploy.md)
5. **每完成一个 section / 功能**：必走 [05-checklist.md](05-checklist.md)
6. **写新 section / 把素材代码改成 Liquid**：查 [06-section-conventions.md](06-section-conventions.md)
7. **要把素材库代码丢给外部 AI 转换**：复制 [07-html-to-section-prompt.md](07-html-to-section-prompt.md) 分隔线下的提示词
8. **遇到具体 bug**：先查 [BUGS.md](BUGS.md)，没有的话**解决后回填这个文件**

## 文件清单

| 文件 | 内容 |
|---|---|
| [**00-MASTER.md**](00-MASTER.md) | **入口总览**：架构决策、转换流程、CSS 防御、诊断方法、12 章节连通的完整 playbook |
| [01-architecture.md](01-architecture.md) | 技术选型、文件结构、设计 token、字体 |
| [02-css-cascade.md](02-css-cascade.md) | Dawn/Lumia 全局污染、命名规范、specificity 规则 |
| [03-animations.md](03-animations.md) | transition/animation 取舍、滚动驱动、GSAP↔CSS 映射 |
| [04-shopify-deploy.md](04-shopify-deploy.md) | Liquid 上限、schema vs index.json、push 循环 |
| [05-checklist.md](05-checklist.md) | 每个 feature 必走的 7 步验证 |
| [06-section-conventions.md](06-section-conventions.md) | 把任意 HTML/CSS/JS 素材改写成 section 的内部规范 |
| [07-html-to-section-prompt.md](07-html-to-section-prompt.md) | 给外部 AI 用的自包含转换提示词 |
| [BUGS.md](BUGS.md) | 已修 bug 目录，含症状 → 根因 → 修法 |

## 维护规范

- **每修一个非显而易见的 bug**：往 [BUGS.md](BUGS.md) 加一行
- **每次发现新的 Dawn 干扰**：补到 [02-css-cascade.md](02-css-cascade.md) 的雷区列表
- **写新动效**：先看 [03-animations.md](03-animations.md) 有没有对应模式，没有的话**实现后回填**
- 文档自身的修改请保留 git 历史（即使本仓库不在 git，也用日期标注）

## 元教训 — 任何动效返工 5 次以上时

如果你发现自己反复改某个动效（菜单、carousel、入场……），用户连续 3+ 次反馈"还是不对"——**先停下，去 grep 原站 CSS**：

```bash
# 在原站 CSS 转储里找对应 data-v-XXXX scope
grep -oE 'XXXXX[^}]{0,300}' 007-fluidglass/_nuxt/style.BshK6p_6.css
```

一次性把原版的所有规格（尺寸、位置、transform-origin、动画时长、easing）拿到，比凭截图猜要快 5-10 倍。这条踩了至少 3 次（菜单 5 轮、按钮箭头 6 轮、wordmark 2 轮）。

## 更新历史

- 2026-05-11：初版（Phase 2 完成）
- 2026-05-11：补充 z-index:-1 stacking 坑、双向 toggle 用 transition 不用 animation、滚到底 settle 延迟、模态从某点长出来配方
- 2026-05-14：ffd-theme 部署完成，补充 Lumia ajax-scroll、手控 wheel handler 等 P3 系列 bug
- 2026-05-15：more-nutrition-theme 部署，新增 [00-MASTER.md](00-MASTER.md) 作为总入口；新增踩坑：`request.page_type` 比 `template.name` 可靠、Webflow CSS url() 引用必须扁平化改写、Lumia `html { font-size: 20px }` 让 rem 单位放大 25%、字体 404 退 Arial 导致"字号偏大"假象
