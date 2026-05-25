# Shopify 网页移植 Playbook

把高端动效网站（Nuxt SSG / Webflow / Next.js / 静态站）移植成 Shopify 原生 Liquid theme 的流程规范。

**先读 [00-MASTER.md](00-MASTER.md)** —— 入口总览，包含 6 个真实项目的全部踩坑沉淀。

源自：
- 2026-05-11 fluid.glass → himaxlimited.myshopify.com（`../fluidglass-theme/`，源材料 `../007-fluidglass/`）
- 2026-05-14 fluid-folding-door → himaxlimited.myshopify.com（`../ffd-theme/`，源材料 `../designs/010-fluid-folding-door/`）
- 2026-05-15 morematcha.vercel.app → himaxlimited.myshopify.com（`../more-nutrition-theme/`，源材料 `../designs/013-more-nutrition/`）
- 2026-05-18 www.floema.com → himaxlimited.myshopify.com（`../floema-theme/`）
- 2026-05-20 viture-beast → himaxlimited.myshopify.com（`../viture-beast-theme/`）
- 2026-05-25 cravburgers.shop → himaxlimited.myshopify.com（`../crav-theme/`，源材料 `C:/Users/EDY/Documents/sc2/designs/021-cravburgers/`）

---

## ⚠️ 归档铁律

**所有移植项目的踩坑总结、教训、动效 spec、新发现的母版陷阱 → 一律写到这个 `playbook/` 文件夹**，不要散落到其他位置。详 [00-MASTER.md "文件归档铁律"](00-MASTER.md#️-文件归档铁律2026-05-25-加入) 章节。

---

## 这是写给谁的

- 接手类似项目的开发者
- 用 AI 助手做类似工作的同事 —— **AI 读完这套文档应该能 1:1 复刻**

## 如何使用

1. **新项目立项**：先读 [01-architecture.md](01-architecture.md) 决定 SPA 嵌入 还是 Liquid sections
2. **源站是 Next.js 15 + Tailwind v4**：必读 [08-nextjs-porting.md](08-nextjs-porting.md)（@layer unwrap、next/font CSS vars、GSAP 付费 plugin vanilla 替代、源动效完整 spec）
3. **写样式时被 Dawn/Lumia 干扰**：查 [02-css-cascade.md](02-css-cascade.md)
4. **把 GSAP 动效搬到 CSS / vanilla JS**：查 [03-animations.md](03-animations.md) + [08-nextjs-porting.md §6 §8](08-nextjs-porting.md#6-gsap-付费-plugin-vanilla-替代)
5. **配置 / 部署 / schema 问题**：查 [04-shopify-deploy.md](04-shopify-deploy.md)
6. **每完成一个 section / 功能**：必走 [05-checklist.md](05-checklist.md)
7. **写新 section / 把素材代码改成 Liquid**：查 [06-section-conventions.md](06-section-conventions.md)
8. **要把素材库代码丢给外部 AI 转换**：复制 [07-html-to-section-prompt.md](07-html-to-section-prompt.md) 分隔线下的提示词
9. **Push 后必做 Playwright 自验**：流程见 [09-playwright-verify.md](09-playwright-verify.md)
10. **遇到具体 bug**：先查 [BUGS.md](BUGS.md)（P1/P2/P3/P4 系列），没有的话**解决后回填这个文件**

## 文件清单

| 文件 | 内容 |
|---|---|
| [**00-MASTER.md**](00-MASTER.md) | **入口总览**：架构决策、转换流程、CSS 防御、诊断方法、12 章节连通的完整 playbook |
| [01-architecture.md](01-architecture.md) | 技术选型、文件结构、设计 token、字体 |
| [02-css-cascade.md](02-css-cascade.md) | Dawn/Lumia 全局污染、命名规范、specificity 规则（含 Tailwind v4 @layer 警告） |
| [03-animations.md](03-animations.md) | transition/animation 取舍、滚动驱动、GSAP↔CSS 映射 |
| [04-shopify-deploy.md](04-shopify-deploy.md) | Liquid 上限、schema vs index.json、push 循环 |
| [05-checklist.md](05-checklist.md) | 每个 feature 必走的 7 步验证 |
| [06-section-conventions.md](06-section-conventions.md) | 把任意 HTML/CSS/JS 素材改写成 section 的内部规范 |
| [07-html-to-section-prompt.md](07-html-to-section-prompt.md) | 给外部 AI 用的自包含转换提示词 |
| [**08-nextjs-porting.md**](08-nextjs-porting.md) | **Next.js 15 + Tailwind v4 移植专题**（CRAV 项目沉淀）：@layer unwrap、next/font CSS vars、GSAP 付费 plugin vanilla 替代、8 类源动效完整 spec |
| [**09-playwright-verify.md**](09-playwright-verify.md) | **Playwright 自验方法论**：CDP `CSS.getMatchedStylesForNode`、逐 section scroll+screenshot、hover/scroll 动效验证 |
| [BUGS.md](BUGS.md) | 已修 bug 目录，含症状 → 根因 → 修法 → 教训（P1/P2/P3/P4） |

## 维护规范

- **每修一个非显而易见的 bug**：往 [BUGS.md](BUGS.md) 加一行（P1-XX/P2-XX/P3-XX/P4-XX 编号递增）
- **每次发现新的 Dawn/Lumia 干扰**：补到 [02-css-cascade.md](02-css-cascade.md) 的雷区列表
- **写新动效**：先看 [03-animations.md](03-animations.md) 跟 [08-nextjs-porting.md §8](08-nextjs-porting.md#8-source-动效完整-spec-速查crav-实战) 有没有对应模式，没有的话**实现后回填**
- **新部署完成后**：必跑 [09-playwright-verify.md §9](09-playwright-verify.md#9-必查-checklistpush-后) checklist，过了再向用户报告
- 文档自身的修改请保留 git 历史（即使本仓库不在 git，也用日期标注）

## 元教训

### 1. 任何动效返工 5 次以上时

如果你发现自己反复改某个动效（菜单、carousel、入场……），用户连续 3+ 次反馈"还是不对"——**先停下，去 grep 原站 CSS / JS chunk**：

```bash
# 在原站 CSS 转储里找对应 data-v-XXXX scope（Nuxt SSG）
grep -oE 'XXXXX[^}]{0,300}' 007-fluidglass/_nuxt/style.BshK6p_6.css

# 在 Next.js chunk 里找动效 spec
grep -lE 'data-pop|cursor|MotionPath|InertiaPlugin' _next/static/chunks/*.js
```

一次性把原版的所有规格（尺寸、位置、transform-origin、duration、ease、stagger、threshold）拿到，比凭截图猜要快 5-10 倍。这条踩了至少 4 次（菜单 5 轮、按钮箭头 6 轮、wordmark 2 轮、viture-beast useMask/useAnimation 7 轮）。

### 2. 自家防御层 CSS 是反向 sabotage 高发地

写完 `{prefix}-base.css` 必审：是否在 body/wrapper 加了 inheritable property 的 `!important`？是否套了 `.{prefix}-page` 前缀给通用元素（h*/p/img）加规则？详 [08-nextjs-porting.md §9](08-nextjs-porting.md#9-反向-cascade-self-sabotage-检查清单)。

### 3. Push 完先 Playwright 自验，不要靠用户截图

详 [09-playwright-verify.md](09-playwright-verify.md)。用户反馈截图错误 ≥ 1 次 = 你没自验。CDP probe + screenshot 是部署的最后一步，不是可选步骤。

## 更新历史

- 2026-05-11：初版（Phase 2 完成）
- 2026-05-11：补充 z-index:-1 stacking 坑、双向 toggle 用 transition 不用 animation、滚到底 settle 延迟、模态从某点长出来配方
- 2026-05-14：ffd-theme 部署完成，补充 Lumia ajax-scroll、手控 wheel handler 等 P3 系列 bug
- 2026-05-15：more-nutrition-theme 部署，新增 [00-MASTER.md](00-MASTER.md) 作为总入口；新增踩坑：`request.page_type` 比 `template.name` 可靠、Webflow CSS url() 引用必须扁平化改写、Lumia `html { font-size: 20px }` 让 rem 单位放大 25%、字体 404 退 Arial 导致"字号偏大"假象
- 2026-05-18：floema-theme 部署，补充 Schema name >25 silent reject / 文件名 >25 silent reject / CLI checksum cache 死锁 / SFC scoped styles 扁平化等 P3-07/P3-08/P3-09 系列
- 2026-05-25：crav-theme 部署，补充 P4 系列（Tailwind v4 @layer cascade / self-defense !important leak / .crav-page wrapper prefix self-sabotage / .page-content inherited override / Next.js _next/image alpha loss / char-split whitespace gap / GSAP paid plugin vanilla 替代 / cursor rope timing / Playwright self-verify）；新增 [08-nextjs-porting.md](08-nextjs-porting.md) + [09-playwright-verify.md](09-playwright-verify.md) 两章；订立**"归档铁律"** —— 总结一律入 playbook 不散落
