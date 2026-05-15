# Bug 目录

按发生顺序记录。每个条目格式：**症状 / 根因 / 修法 / 教训**。
新 bug 修完后请回填到这里。

---

## P1-01：Phase 1 — Nuxt unctx Context 冲突（不可救）

- **症状**：SPA 整站塞进单个 Liquid，预览页报 `beforeEach undefined` 和 unctx Context 冲突，hydration 失败
- **根因**：Shopify 通过 `{{ content_for_header }}` 注入了第三方 app 的 Vue/Nuxt 嵌入式 script（cdn.slpht.com、cdn.nfcube.com 等），跟主页面的 Nuxt 实例抢全局 `unctx` 上下文
- **修法**：放弃 Phase 1，改 Phase 2 拆 Liquid sections
- **教训**：店铺装了 Vue/Nuxt app → SPA 嵌入死路。立项必须先问商家「装了哪些 app」

## P1-02：Liquid 256KB 上限

- **症状**：单个 `.liquid` 文件超 256KB，`shopify theme push` 报错
- **根因**：Phase 1 内联 873KB 的 `__NUXT_DATA__` JSON
- **修法**：拆 4 个 chunk，用 `{%- render -%}`（双侧 trim）拼接
- **教训**：内联大数据时计算文件大小，先拆好再写

## P1-03：`{% render %}` 插换行破坏 JSON

- **症状**：拆 chunk 后 JSON.parse 失败，错误指向 control character
- **根因**：`{% render %}` 默认在 render 周围插入换行符
- **修法**：用 `{%- render -%}` 双侧 trim
- **教训**：拼内联数据一律用 `{%-` `-%}`

---

## P2-01：Hero 文字颜色不是白色

- **症状**：home-header 文字渲染成深灰，应该是白色
- **根因**：`.fg-section { color: var(--fg-color-grey) }` 在 fg-base.css，`.fg-home-header { color: var(--fg-color-white) }` 在 section bundle，加载顺序导致前者赢
- **修法**：用更高 specificity：
  ```css
  .fg-home-header,
  .fg-home-header .base-heading,
  .fg-home-header .base-title,
  .fg-home-header .text,
  .fg-home-header .indicator { color: var(--fg-color-white); }
  ```
- **教训**：不依赖加载顺序赢 specificity；section bundle 跟 fg-base.css 的相对顺序不保证

## P2-02：Hero 标题不在底部

- **症状**：home-header 用 grid，`align-items: flex-end` 标题还是顶部
- **根因**：CSS Grid 里 `align-items` 控制每个 grid 项目在自己 cell 里的对齐，但要整体推到底需要 `align-content`
- **修法**：`align-items: flex-end; align-content: end;` 同时设
- **教训**：grid 单行时 `align-items` 不够，要 `align-content`

## P2-03：所有比例大 1.6 倍

- **症状**：用 rem 单位写完后，所有元素比原站大 1.6 倍
- **根因**：CSS 默认 1rem = 16px，但原站根字号是自定义 `(100vw/1600)*10`（≈ 10px 在 1600 视口）
- **修法**：全部 `Nrem` → `calc(N * var(--fg-font-s))`。写 `scale-rem.mjs` 批量转，77 处
- **教训**：复刻视口适配站点时，先把根 unit 体系对齐再写组件

## P2-04：产品集合 hover 图片不跟鼠标

- **症状**：fg-featured-projects 的项目 hover 图片浮现的位置随鼠标移动，但用户要求**固定位置**
- **根因**：误读需求，加了鼠标跟随
- **修法**：图片 `position: absolute; left: 29.7rem; top: 50%; transform: translateY(-50%)`，去掉 mousemove 监听
- **教训**：阅读需求时 hover 行为可能跟"鼠标位置追踪"分开；不要默认加

## P2-05：浮动 page-header 覆盖整页而非 hero 顶

- **症状**：fg-page-header 的"Get a quote"按钮和 wordmark 不在 hero 顶部，而是黏在视口顶部
- **根因**：用了 `position: fixed; top: 0`
- **修法**：改成 `position: absolute; top: 0`，跟随 hero section 滚动
- **教训**：装饰性顶部条用 absolute（跟随 section），导航条用 fixed（跟随视口）

## P2-06：按钮箭头交互理解错（迭代 6 次）

- **症状**：用户反复说按钮 hover 动画不对
- **根因**：箭头实际是 3 元素：L 线 + 右端 chevron 尖 + 旋转方块。初始态显示 L+chevron，hover 后只显示 diamond。我一开始只做了 2 元素，反复调整
- **修法**：3 个独立 SVG 子元素分别控制 stroke-dasharray + dashoffset：
  - `.arrow-line` (dasharray 19) hover 时 offset 0 → 19（擦除）
  - `.arrow-chevron` (dasharray 12) 同步擦除
  - `.arrow-diamond` (dasharray 23) offset 23 → 0（画入）
- **教训**：用户给的截图要仔细分辨"由几个图形组成"；面对反复修不到位的情况，直接问"初始态有哪些元素，终态有哪些元素"

## P2-07：Showroom 文字位置乱（10× 单位错）

- **症状**：showroom 滚动时左右两栏文字滑入幅度极大、滑到中心
- **根因**：原站 `gsap.to(el, { x: c*65 })` 是 65 像素（c = width/1600），不是 65rem。我用了 `calc(65 * var(--fg-font-s))` = 650px，**多了 10 倍**
- **修法**：`calc(6.5 * var(--fg-font-s))` ≈ 65px
- **教训**：GSAP 数值是像素；从原站 GSAP 代码搬动画，**数值要 ÷ 10**（在 fg-font-s 体系下）

## P2-08：Assets duo parallax 完全没生效

- **症状**：左图应该随滚动从顶滑到底，实际不动
- **根因**：图片 `loading="lazy"`，JS 测量 `img.clientHeight` 时图片还没加载，返回 0，差值算成 0 跳过
- **修法**：双重保险：
  1. 改 `loading="eager"`
  2. JS 中检测 `media.complete`，未完成则挂 `load` 事件触发重测（用 `dataset.fgWaiting` 标记防重复挂）
- **教训**：测量类图片不能 lazy；或者必须有 `load` 事件回退

## P2-09：Footer 没动画

- **症状**：原站 footer 入场有 container 滑入 + wordmark 缩放 + logo 末段淡入，我的版本完全静止
- **根因**：用了 `data-fg-scroll="enter"` 模式，但 enter 模式是"全程进入到离开"，footer 还没到顶时 progress 已经过半
- **修法**：加新模式 `data-fg-scroll="reveal"`：`progress = clamp((vh - rect.top) / rect.height)` —— 从底进入到自身完全在视口时 = 0 → 1
- **教训**：滚动驱动的不同节奏需要不同的"起止区间"；写 progress 工具时多设计几种模式

## P2-10：Footer wordmark 字体位置不对 + 字过大

- **症状**：用 plain text "Fluid Glass" 渲染水印，位置和原站不一样；尺寸 152.3rem 撑满视口看着大
- **根因 1**：原站用 SVG mask 把"Fluid Glass"的精确字形 mask 出来，跟我们的 Aeonik 字体 baseline / descender 有几像素差
- **修法 1**：直接 inline 原站的 SVG 路径（从原 CSS data URI 拷贝），完全脱离字体度量
- **根因 2**：用户觉得视觉上太大（虽然跟原站尺寸一样）
- **修法 2**：把 `152.3rem × 25.4rem` 缩到 `78rem × 13rem`（保持 6:1 宽高比）
- **教训**：用 SVG 路径替代字体渲染做精确位置；尺寸"对齐原站"不一定等于"用户觉得对"

## P2-11：菜单 X 关闭按钮压住 CTA

- **症状**：菜单打开时底部 X 按钮跟 GET A QUOTE 按钮重叠
- **根因**：用 `position: fixed; bottom: 4rem` 定位 X，跟用 `bottom: 14rem` 定位的 panel 底部撞了
- **修法**：菜单容器改 flex stack：`display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 3rem`，panel 和 X 都是普通 flex item 自然不重叠
- **教训**：多个 fixed/absolute 元素要靠人工算距离避免重叠；用 flex stack 自动布局更稳

## P2-12：浮动 nav HOME 文字偏左

- **症状**：浮动 nav 上的 "HOME" 文字视觉上不在 pill 中心
- **根因**：用 flex 三槽位（logo + flex:1 title + burger），title 在**剩余空间**里居中，但 logo 5rem + burger 5rem 不等于 title 槽位中心是 pill 几何中心
- **修法**：side 槽 `position: absolute; left/right: 0`，title `position: absolute; inset: 0` + flex 居中
- **教训**：要 pill 几何中心居中，用 absolute + inset:0 + flex，不要靠 flex 三槽

## P2-13：Reviews 箭头按钮变大 + 颜色不对

- **症状**：reviews 的 prev/next 按钮渲染成大方块，边框颜色不对
- **根因**：button class 叫 `button`，撞 Dawn 全局 `.button` 样式（padding / min-height / box-shadow / 字体一堆都注入了）
- **修法**：
  1. 改名 `arrow-btn`
  2. 显式重置：`appearance: none; padding: 0; margin: 0; min-height: 0; min-width: 0; border-radius: 0; box-shadow: none; font: inherit; letter-spacing: 0`
- **教训**：任何裸的全名 class（button / icon / media / field / title）都是 Dawn 雷区，前缀 `fg-`

## P2-14：Reviews 切换没重播动画

- **症状**：第一条评论的引语逐行升起 ok，点 next 后第二条没动画，直接显示
- **根因 1**：用了 CSS `transition` 实现。同一帧 remove + add `is-revealed` 会被合并成"无属性变化"，transition 不触发
- **修法 1**：改用 `@keyframes` animation，移除 class → 动画卸下、元素瞬回 110%；加回 class → 新动画从头跑
- **根因 2**：手动 carousel JS 没做"标准重启三件套"
- **修法 2**：`remove → void offsetWidth → add`
- **教训**：需要重放的视觉用 `animation`；transition 只适合一次性过渡

## P2-15：整页背景是白色不是米色

- **症状**：body 背景应该是 #f3f0ec 米色，实际是白色
- **根因**：Dawn 的 `body.color-background-1 { background-color: rgb(var(--color-background)) }` specificity 0,1,0 压过我的 `body { background: cream }` specificity 0,0,1
- **修法**：`body, body.gradient { background: var(--fg-color-cream) !important; background-color: var(--fg-color-cream) !important }`，两个属性 + 显式 .gradient class + !important
- **教训**：body 背景永远 `!important`，Dawn 配色方案不可绕开

## P2-16：所有 base-heading 入场动画只有第一个有

- **症状**：hero 大标题逐行升起，但下面 5 个 section 的标题都是静态
- **根因**：JS 用 `requestAnimationFrame` 一次性给所有 `[data-fg-reveal-lines]` 加 `is-revealed`，下面的标题用户还没滚到时就播完了
- **修法**：改用 IntersectionObserver（threshold 0.1），每个标题滚到 10% 可见才触发
- **教训**：入场动画一律 IO 触发，不要一次性广播

## P2-17：行拆分错位（字体加载前算）

- **症状**：偶发，标题逐行升起的换行位置跟最终显示不一致
- **根因**：JS 拆行时字体还在加载，按系统字体宽度算的换行点
- **修法**：拆行前等 `document.fonts.ready`
- **教训**：任何依赖文字宽度测量的 JS 必须等字体加载

## P2-18：Sticky 失效（overflow:hidden 干扰）

- **症状**：showroom 改成 200svh sticky 后，inner container 不 pin
- **根因**：section 上有 `overflow: hidden`，sticky 找到这个祖先就停了，不再 pin 到视口
- **修法**：把 `overflow: hidden` 改成 `overflow: clip` —— clip 不创建滚动容器，sticky 跳过它继续找祖先
- **教训**：需要 sticky 的祖先链 + 横向裁剪需求 → 用 clip 而非 hidden

## P2-19：Storyblok CDN URL 转换后 404

- **症状**：从原站抓的图片 URL 形如 `https://a.storyblok.com/.../filters_format(jpg)/...`，请求返回 404
- **根因**：Storyblok 图片变换 URL 被该 space 禁用了
- **修法**：用裸 URL（去掉 `/m/.../filters_*` 后缀），下载到 `assets/` 本地引用
- **教训**：第三方图床别依赖原 URL，下载到本地最稳

## P2-20：默认值闪烁

- **症状**：showroom column translateX 在 JS 加载前显示在自然位置（0），JS 加载后瞬间跳到 6.5rem 偏移
- **根因**：CSS `var(--fg-progress, 1)` 的 fallback 选了 1（=入场完成态），JS 还没把 progress 设成 0（=入场初态）时显示完成态
- **修法**：fallback 改成 0（=初态），JS 加载前后视觉一致
- **教训**：`var(..., fallback)` 的 fallback **必须等于"section 还在视口外的初始状态"对应的 progress 值**

---

## P2-21：Hero 滚动时文字不上浮（缺视差）

- **症状**：滚动 hero 时，文字和背景图按相同速度移动 = 没有视差感
- **根因**：复刻时漏了原站的 `gsap.fromTo(asset, { yPercent: 0 }, { yPercent: 50 })`，asset 没做相对位移
- **修法**：
  1. 给 `fg-scroll-progress.js` 加 `leave` 模式（`-rect.top / rect.height`，top top → bottom top 区间）
  2. `<section data-fg-scroll="leave">`
  3. `.asset { transform: translateY(calc(var(--fg-progress, 0) * 50%)) }`
  4. `.indicator { opacity: max(0, calc(1 - var(--fg-progress, 0) * 10)) }` 顺便加上首 10% 淡出
- **教训**：从原站照搬 GSAP 时检查每条 `.fromTo(...)`，不要只看显眼的那条；scrollTrigger 的 start/end 决定要选哪个 progress 模式

## P2-22：菜单整体重做（5 轮迭代）

- **症状**：用户对菜单连续反馈 5 次：
  1. 现在菜单格式不对（最初是浅色右侧抽屉，应该是中央暗色 modal）
  2. 菜单比原版大 + 字体大
  3. X 关闭按钮跟原 HOME 浮动 nav 不在同一位置（被 flex stack 推上去了）
  4. 菜单打开不是直接出现，要有"从一个小长方形展开变大"的动画
  5. 暗色长方形直接不显示了
- **根因**：
  1-3. 我一开始没仔细看原站 CSS，凭印象造。第 4 轮才去 grep `data-v-83565617`（菜单 scope）拿到原站 `.menu` 的真实样式：`bottom: 10rem`（锚底）、`.menu .background` 独立子层 `transform-origin: center bottom`
  4. 第 5 轮按原站结构拆 `<div class="fg-menu-bg">` 当独立背景层 + `z-index: -1` + 父 `isolation: isolate` —— 部分浏览器在 transform + overflow + isolation 组合下把负 z-index 子元素整体裁掉
- **修法**：直接把背景放 `.fg-menu-panel` 上，整个面板 `scale(0) → scale(1)` + `transform-origin: center bottom`。删掉独立的 `.fg-menu-bg` div 和 `isolation: isolate`。内容跟着面板一起缩放，原站效果一致
- **教训**：
  1. **复刻动效前先 grep 原站对应 data-v scope 的 CSS**，不要凭截图猜测。一次性把样式拿全比反复迭代快得多
  2. **能不分层就不分层**：`z-index: -1` 在 transform/isolation/overflow 父级里有跨浏览器不一致渲染坑，直接把背景放容器上更稳（详见 [02-css-cascade.md §10](../fluidglass-playbook/02-css-cascade.md)）
  3. **双向切换的动画用 transition 不用 animation**：animation 移除 class 会瞬间 snap 回基础态，关闭看起来生硬；transition 自带反向过渡（详见 [03-animations.md §11](../fluidglass-playbook/03-animations.md)）
  4. **asymmetric delay 技巧**：`transition-delay` 写在 `.is-open` 选择器里，开有 delay、关无 delay

## P2-24：滚动驱动动画卡顿（鼠标滚轮一卡一卡）

- **症状**：第一轮 — hero 滚动视差、product-collection block 平移等 scroll-driven 动画看起来跟随 mouse wheel 步进，不丝滑
- **修法 v1**：`fg-scroll-progress.js` 在 rAF 循环里 lerp progress：
  ```js
  current += (target - current) * 0.18;
  ```
- **反馈 #1**："还是一卡一卡 很僵硬 原版好像是缓慢减速的那种感觉"
- **修法 v2**：两层平滑
  1. progress lerp 调到 0.08
  2. 追加 ~40 行 Lenis 等价实现：拦 wheel → preventDefault → 累加到虚拟 target → rAF 用 `window.scrollTo` 把 scrollY 平滑追上去
- **反馈 #2**："缓慢滚动差不多了 但是滑动快时还是有问题 原版鼠标滑动快时也是缓慢上升下降的"
- **修法 v3**：
  1. `SCROLL_LERP` 从 0.1 降到 0.06（减速尾从 ~500ms 拉到 ~1.5s）
  2. 加 `normalizeDelta(e)` 处理 `deltaMode === 1` (lines) / `2` (pages)：Firefox 用户尤其需要，否则报 lines mode 时每次只走几 px
- **教训**：
  1. **scroll-driven 丝滑实际是两层叠加**：scroll 位置本身要平滑（Lenis）+ progress 派生再平滑（lerp）。只做一层反差更明显
  2. **快速滚动的"长尾"感觉来自低 SCROLL_LERP**：每帧只追 6% 差值，target 大幅领先时需要很多帧才追上 = 长滑行
  3. **`normalizeDelta` 不要漏**：Firefox 用 `deltaMode: 1` (lines)，原始 `e.deltaY` 值很小，不归一化就基本不动
  4. **拦 wheel 必须 `passive: false` + `preventDefault`**；只拦 wheel 不要拦 touch/keyboard
  5. **resync from `window.scrollY`** 是关键：用户 idle 时按键盘 / 拖 scrollbar / 点锚点改了 scrollY，下一次 wheel 之前必须重新读一次同步回来
  6. **SCROLL_LERP 调参对照表**：0.10 紧（500ms）/ 0.08 中（700ms）/ 0.06 长尾（1.5s）/ 0.04 太飘

## P2-23：滚到底菜单触发太早

- **症状**：用户滚到页面底部时菜单立刻弹出，感觉"还在滚就跳出来"
- **根因**：scroll 事件每次触发都检查 `atBottom()`，一进入阈值就 open。但用户还在快速滚动中，没"稳定"到底
- **修法**：加 250ms settle 定时器。`atBottom()` 触发后启动 timer，250ms 后再次确认仍在底部才真正 open；中途离开则取消
- **教训**：scroll 触发的 UI 加个小 settle 延迟（~200-300ms）能避免边滚边触发的体验问题；不要在每次 scroll 事件直接做关键动作

---

## P3-02：rem token scoping 写错导致全站字号大 2-3 倍

- **症状**：preview 上首屏的标题、wordmark、按钮全部比原站大很多，"your home with folding doors" 这种短句每个词都换行
- **根因**：`--ffd-font-s = (100vw/1600)*10` 定义在 `.ffd-page` 这个子元素上，但 `rem` 单位**始终锚定 `<html>` 元素的 font-size**，不看父级。Dawn 设了 `html { font-size: ~16px }`，所以 4rem 渲染成 64px 而不是预期的 40px
- **修法路径 A**：把 token 提到 `:root`，加 `html { font-size: var(--ffd-font-s) !important }`。**前提是 ffd-base.css 只在 index 模板加载**（`{%- if template.name == 'index' -%}` 包 link 标签），这样不影响其他模板
- **修法路径 B**（fluidglass 旧版采用）：把所有 `Nrem` 全部改写成 `calc(N * var(--fg-font-s))`，配 `scale-rem.mjs` 批量转 77 处
- **教训**：写法上更倾向路径 A —— 一行 `html { font-size }` 解决全部 rem 单位映射，比扫 77 处省事。但前提是基础 CSS 严格只加载到目标模板，否则会影响 Dawn 默认页面。fluidglass 时代没用路径 A 是因为整站 SPA 嵌入；section 化以后路径 A 更优。

---

## P3-06：base-button 的 CSS 加了 `.ffd-page` 前缀,菜单里完全没接到样式

- **症状**：菜单打开后，里面的 "Get a quote" 按钮的箭头 SVG **撑满整个面板**（无 width/height 约束），按钮文字也变成 "Get a quoteGet a quote"（双行 label 都裸露）
- **根因**：base-button 的所有 CSS 规则我写成了 `.ffd-page .base-button { ... }`，但是 floating-nav / menu / cookies popup 这些组件都是在 `layout/theme.liquid` 里**直接挂在 `<body>` 下面**的（不属于任何 section，渲染在 main 之外）。它们不在 `.ffd-page` 这个 wrapper 内 → 我的选择器全部 miss → 这些组件里的 base-button 一条样式都没接到,SVG 自然按 viewBox + 容器尺寸 stretch 到巨大
- **修法**：所有共享组件级（base-button / base-title / line-mask 等）的 CSS 选择器**写成 bare class**（`.base-button` 不要 `.ffd-page .base-button`）。Dawn 不占用 `base-*` 前缀，安全
- **教训**：
  1. 共享样式（base-button、cursor、splash 等）跟 section 级样式分清楚。**section 级样式可以 `.ffd-page` 前缀作用域，共享组件不能** —— 因为共享组件经常浮在 body 上（modal / cursor / cookies / floating nav）
  2. 写 ffd-base.css 的时候要思考"这个 class 会出现在 .ffd-page 外吗"，会的话就 bare 选择器
  3. 也要给 SVG 加 `flex-shrink: 0`，否则在 flex 容器里被压扁照样变形

---

## P3-05：Lenis 在 Lumia 母版上被静默吃掉,要换手控 smooth scroll

- **症状**：调了一晚上 Lenis（duration 1.2 → 2.4 → lerp 0.04 → 0.03，wheelMultiplier 1 → 0.5 → 0.25），用户反馈跨多个版本「没区别 / 还是一下子飞过去」。同样的 Lenis 代码 + 同样的 store 在 fluidglass-v2 上能工作
- **根因**：Lumia 派生 theme 在 `theme.liquid` / `assets/global.js` / `assets/animations.js` 等地方早早绑了 `wheel` / `scroll` 监听器。Lenis 1.x 默认靠 `addEventListener('wheel', ..., {passive:false}) + preventDefault()` 来劫持滚动，但如果母版的 listener 跑在 capture 阶段或没让出事件，Lenis 的 preventDefault 形同虚设 —— 你看到的是 native 滚动，再多的 lerp / duration 都是空配
- **修法**：完全弃用 Lenis，手写一个 ~40 行 smooth-scroll。三层结构：
  1. `wheel` 监听器 `passive: false`，**每次 e.preventDefault() 抢断 native**
  2. clamp `deltaY` 到 ~120px 上限 +  `wheelMultiplier ~0.8` → 单 tick 累加到 `target`
  3. rAF tick: `current += (target - current) * 0.1`，`window.scrollTo(0, current)`
  - 关键:`window.scrollTo()` 自动触发 native scroll 事件,GSAP ScrollTrigger 走原生路径，不再依赖 Lenis 的 `lenis.on('scroll', ...)`
- **额外坑**：途中我犯了一个错 —— 加了「per-frame 12px 硬速度上限」让用户狂滚也只能慢慢漂。用户反馈"对了但太慢"，因为这破坏了 native 速度感。**正确做法**:不要硬封速度,只用 `lerp` 做平滑(0.1 = ~500ms 收尾)。"特定段慢"应该由那段 GSAP scrub 动画的范围/距离自然产生（footer 100svh 的 scrub 让 translateY/scale 看起来很慢，跟全局速度无关）
- **教训**：
  1. **接管 Shopify 母版的 wheel 必须自己 preventDefault**。第三方 lib（Lenis / Locomotive / 等）的事件劫持在被装了 wheel listener 的 theme 上会静默失败
  2. **"smooth"和"slow"是两件事**。smooth = lerp 平滑收尾；slow = scrub 动画跨度长。混在一起调（硬封 velocity）只会两头不到岸
  3. **调参顺序**：先 `LERP`（决定丝滑感）→ `WHEEL_MULTIPLIER`（决定 target 累积速度）→ 最后 `MAX_DELTA`（只防高 DPI 鼠标飞屏）。**绝对不要加硬速度上限**
  4. 这条坑了 6 轮迭代（Lenis 4 个版本 + 自家 lerp + 加上限再去掉）才到位

---

## P3-04：Lumia/Dawn 主题的 ajax-scroll 把 sections 切成占位符,GSAP ScrollTrigger 测不到

- **症状**：所有滚动驱动动画(parallax、SVG 路径绘制、footer reveal)在 preview 上完全不触发。`--ffd-progress` 永远停在 0/未设。但页面 *能* 正常滚动看到所有 section
- **根因**：继承的 Lumia 主题 `layout/theme.liquid` 在 `template == 'index'` 模板上启用了 ajax-scroll 模式 —— 第三个 section 起被替换成占位符 `<div data-ajax-section-scroll="..." class="shopify-section show-on-scroll"></div>`,真实内容通过 ajax 在用户滚到附近时才加载。GSAP ScrollTrigger 在初始化时测不到不存在的元素,创建出来的 trigger 范围全是 0,onUpdate 永不触发
- **修法**：把 `theme.liquid` 里 `{%- if settings.use_ajax_scroll and template == 'index' and request.design_mode == false -%}` 的条件加一句 `and template.name != 'index'` 强制把它变成 false,走 else 分支 `{{ content_for_layout }}` 让所有 section 一次性渲染
- **教训**:
  1. 接管 index 时不只要 unless 掉 header/footer/breadcrumbs,**整个 layout/theme.liquid 都要扫一遍 ajax-scroll、`section-negative-margin`、`data-ajax-*` 这类延迟加载 hack**
  2. ScrollTrigger 的 trigger 元素必须在初始化时就存在于 DOM,否则它的 start/end 位置计算成 0,即使后来插入也不会自动 refresh(除非显式调 `ScrollTrigger.refresh()`)
  3. 这条踩在 ffd-folding-door 第 5 次推送 —— 之前换过 4 套不同的 scroll 进度实现(简单 scroll listener、rAF lerp、GSAP enter、GSAP+Lenis)都不工作,根因都不在 JS 而在 Liquid 母版

---

## P3-03：Dawn breadcrumbs 钻进自定义页面

- **症状**：自定义 hero 内出现一个白色矩形遮住部分内容
- **根因**：`layout/theme.liquid` 里 `{% render 'breadcrumbs' %}` 不在任何条件块里，所有模板都会渲染。Dawn 的 breadcrumbs 默认就是个浅色 pill
- **修法**：套 `{%- unless template.name == 'index' -%}{% render 'breadcrumbs' %}{%- endunless -%}`
- **教训**：接管 index 模板时除了 `header-group` / `footer-group` 还要扫 `layout/theme.liquid` 里所有零散的 `{% render '...' %}` / `{% section '...' %}`，把不需要的全部 unless 掉。常见嫌疑：`breadcrumbs`、`announcement-bar`、`navigation-sub`、`promo-products`、`megamenu`、`before-body-end`、`newsletter`/`footer` form

---

## P3-01：Section schema `"tag": null` 推送报错

- **症状**：Shopify CLI 推送时报 `Invalid schema: tag must be a string`，导致 templates/index.json 引用该 section 时报 `Section type 'foo' does not refer to an existing section file`
- **根因**：想让 section 不输出默认的 `<div class="shopify-section">` 包裹层，在 schema 里写了 `"tag": null`。Shopify 不接受 null，只接受字符串（如 `"div"`、`"section"`、`"article"`）。section 必须有包裹标签，无法去掉
- **修法**：不要尝试用 section 拆开 / 关闭外层 wrapper（`<div class="page"><main>...</main></div>`）。改成在 `layout/theme.liquid` 里 `{%- if template.name == 'index' -%}<div class="page">...{%- endif -%}` 直接包裹 `{{ content_for_layout }}`
- **教训**：section wrapper tag 不可省略；需要纯逻辑包装层时**用 layout 而不是 section**。这条踩在 ffd-folding-door 首次推送

---

## 模板

新增 bug 时复制下面这段：

```markdown
## P2-XX：[一句话症状]

- **症状**：...
- **根因**：...
- **修法**：...
- **教训**：...
```
