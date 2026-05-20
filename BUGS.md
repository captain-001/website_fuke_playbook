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

## P3-07：Schema `"name"` 字段超 25 字符 push 失败

- **症状**：`shopify theme push` 报 `Invalid schema: name is too long (max 25 characters)`，同时 `templates/index.json` 报 `Section type '...' does not refer to an existing section file`——因为 schema 验证失败，那个 .liquid 文件就不存在了
- **根因**：Shopify 对 section schema 的 `name` 字段有 25 字符的硬上限（实测 2026-05）。`"FL Collections Overview"` 是 23 字符还行，`"Floema · Collections Overview"` 30 字符直接拒
- **修法**：`build-*-sections.mjs` 里组合 schema 时 truncate：`if (name.length > 25) name = name.slice(0, 25)`。命名风格用短缩写前缀（`FL`/`MN`/`FFD`）+ 大写词组
- **教训**：
  1. 25 字符上限对**带项目前缀 + 长 section 名**很容易撞。floema 项目 14 个 section 撞了 5 个
  2. schema 的 `presets[].name` 同样受 25 字符限制
  3. 当看到 templates/index.json 报"section type does not refer to an existing section file"，**先看上面有没有 schema validation error**——通常是 section 编译失败导致它"不存在"

## P3-07b：**Section 文件名（= section type）也有 25 字符硬上限，但 push 时静默拒绝**

- **症状**：修完 P3-07 的 schema name 长度后再 push 显示 `upload complete` 无错误，看起来成功；但访问 storefront preview，首页显示 404 模板内容（不是 index）。再单独 push templates/index.json 时才报 `Section type 'floema-07-workflow-overview' does not refer to an existing section file`
- **根因**：除了 schema `name` 字段，**section 文件名（去 `.liquid` 后缀 = section type）也有 25 字符上限**。`floema-07-workflow-overview` 是 27 字符，**Shopify 服务端静默拒绝，但 CLI 的 push 不报错**。后续 templates/index.json 上传时引用这个 type 才暴露——而 index.json validation 一旦失败，整个首页 fallback 到 404 模板渲染
- **修法**：
  - 把 `build-*-sections.mjs` 加上 section type 长度检查，自动用缩写表替换（`-collections` → `-coll`、`-highlight` → `-hl`、`-overview` → `-ovr`、`-products` → `-prods` 等）截到 ≤25
  - 让 build 脚本顺带写 `templates/index.json`——文件名和 index.json 引用的 type 永远同步，不会因为后续手动 rename 一边而忘了另一边
- **教训**：
  1. **`upload complete` 不等于全部文件落地**。Shopify CLI 对 section 文件名超长是 silent reject。判定标准用 `--json` 输出里的 `"errors"` 字段，不是看人类可读的 "complete" 文字
  2. **诊断 "preview 走 404 模板" 的标准做法**：单独 push `templates/index.json` 看 JSON 输出，它会列出所有 `Section type 'X' does not refer to an existing section file` 错误——这是 section 文件名超长的 smoking gun
  3. **section type 限制不只是文件名**，还反映到 `.liquid` 中通过 `data-section-id`/`section.id`/`section.type` 流转的所有地方。一律压在 25 字符以下
  4. **build 脚本应该同时输出 sections + index.json**——避免 type 名重命名后人工维护 index.json 漏掉某个

## P3-07c：Shopify CLI bulk push 会静默跳过部分新文件，单独推才上传

- **症状**：本地有 14 个新名字的 sections，跑 `shopify theme push --only "sections/floema-*.liquid"` 显示 `Theme upload complete` 无 errors，但服务端 pull 下来一看只有 9 个，缺 5 个新文件
- **根因**：Shopify CLI（3.94.3 实测）在 bulk push 时基于 checksum 做 diff 决定是否上传，**对某些新文件存在错误判定，认为不需要上传**而跳过——CLI verbose log 里看不到该文件被处理。把同一个文件**单独**用 `--only "sections/foo.liquid"` push，CLI 反而正确上传
- **修法**：
  - 对关键 sections 用 `shopify theme pull --only "sections/<name>.liquid"` 验证服务端真的有
  - 如果服务端缺失，**逐个单独 push** —— `for f in foo bar baz; do shopify theme push --only "sections/$f.liquid"; done`
  - 或者改大动作：删掉这 5 个本地文件 → push 一次（让 CLI 删 remote 旧版本，但因为 local 没了所以 CLI 不会判断 checksum 相同）→ 重建本地文件 → 再 push 一次
- **教训**：
  1. **`upload complete` + 0 errors ≠ 文件落地**。哪怕 JSON 输出干净，仍要 pull 验证关键文件是否在服务端
  2. CLI bulk push 的 checksum diff 算法对"local 新文件 + server 没有"这种 case **有时会跳过**——可能跟之前 push 残留的某种缓存有关。绕过：**对关键文件单独 push 一遍**作为兜底
  3. 这次 floema-v1 在这条上卡了一整轮迭代（push 报成功 → 服务端缺 5 个 → index.json 引用失败 → 整页渲染 404 fallback）。诊断时间 ≈ 30 分钟

## P3-07d：Shopify CLI 状态污染后单文件 push 全部失效，admin 时间戳是唯一真相

- **症状**：删过一批 layout/template 别名后，CLI 报过一次 `"layout/theme.liquid could not be deleted"` 警告；之后**所有 `--only` 单文件 push 不管推什么都报 `Theme upload complete`**，但 admin Themes 列表里 `Last saved` 时间戳**永远停留在最早创建时间**。表现像主题被冻结
- **根因**：Shopify CLI 3.94.3 在某种内部 state（一次 mirror push 删除失败后）会卡在某种 cache 污染态，单文件推送被完全跳过——既不上传、也不报错。"upload complete" 只是 CLI 客户端写出的常量字符串
- **修法**：
  - **判断 push 是否真生效，看 admin Themes 列表的 `Last saved` 时间戳**——这是唯一可靠的服务端真相，比 CLI 输出可信度高得多
  - **CLI 卡死后，唯一突破方式是不带 `--only` 的完整 mirror push**：`shopify theme push --theme=ID --nodelete -s store`
  - 关键观察指标：成功的全量 push 会在 stdout 显示 `Uploading files to remote theme [XX%]` 进度条；如果只看到 `Cleaning your remote theme [100%]` + `Theme upload complete` 而没有 upload 进度条，说明又被 silent-skip 了，要再跑一次
  - 加 `--nodelete` 避免之前那次 `could not be deleted` 状态再次触发
- **教训**：
  1. **永远别相信 `Theme upload complete`**——它是 CLI 客户端常量字符串，跟服务端无关。判定标准只看 admin 时间戳 + view-source 实际 HTML
  2. **诊断 banner 是验证 push 是否真生效最快的方法**：往 theme.liquid 的 head 里 inline 写一段 `<style> body::before { content: "v1234" ... } </style>`，每次改 v 编号，刷 preview 看 banner 编号 — 不出现就是 silent skip
  3. floema-v1 在这条上卡了 3+ 小时，30+ 次单文件 push 全部无效。教训：**部署阶段如果 user 反复说"没看到变化"，第一件事是去 admin 列表看时间戳**，而不是怀疑 CSS / cascade / 浏览器缓存

## P3-07e：CLI checksum cache 死锁 — 修改文件内容后 silent-skip 不上传

- **症状**：改了 `assets/foo.css` 添加了 `!important`，跑 `shopify theme push --only "assets/foo.css"` 显示 `Theme upload complete` 但没有 `Uploading [XX%]` 进度条；连改 v 编号注释、连推 5 次都不行。**服务端文件永远停在之前某个版本**
- **根因**：Shopify CLI 3.94.3 维护一个本地 checksum cache。某种污染状态下，cache 里的 checksum 错误地等于 server 上的某个旧 checksum，CLI 的 diff 算法认为"本地跟 server 一致 → 不需要上传"。改了文件内容 → 本地实际 checksum 变了，但 cache 没刷新（具体污染触发条件不清楚，跟之前 `could not be deleted` 状态相关）
- **修法**（按激进程度排序）：
  1. **重命名文件** —— `mv assets/foo.css assets/foo2.css` + 改 theme.liquid 引用 → CLI 必须把新文件名当新增上传。**几乎 100% 有效**
  2. **删本地 + push（不带 --nodelete）让 CLI 同步 delete，再放回 + push** —— 强制 reset checksum
  3. **在 admin 的 Edit Code UI 里编辑一次保存** —— 服务端 checksum 改变，CLI 下次 push 会重新比对
- **教训**：
  1. 判断 push 是否真生效的**唯一可靠指标是 `Uploading [XX%]` 进度条**。没看到进度条就一定是 silent skip
  2. CLI 报告"Theme upload complete"完全没有真实意义（这个字符串在 silent-skip 路径上也会输出）
  3. 改文件内容（包括加 `!important`、加注释、改版本号）**不一定**能让 CLI 重新上传。要靠**改文件名 / 改路径**才能 100% 突破
  4. 这条踩在 floema-v1 collections-cta clip-path 修复时——连推 5 次都没生效，最后靠 rename 突破

## P3-08：嵌套 wrapper `<div class="floema-page">` 在 Lumia/GemPages 母版上不可靠

- **症状**：在 layout/theme.liquid 里写 `{% if request.page_type == 'index' %}<div class="page-wrapper floema-page">{% endif %}` 包裹 sections，inline banner 验证显示同一请求里 `request.page_type` 在头部判定为 'index'（body class 也加上了 `floema-index`），但 **wrapper 那行 `if` 输出的 div **居然没出现在 DOM 里**，DOM 用 inspector 看不到 `.floema-page` class 元素
- **根因**：Lumia/GemPages 母版的 `<div class="page-content">` 容器和 `{% if request.design_mode %}{% section 'megamenu' %}{% endif %}` 之类的 Liquid 块会注入大量预渲染 HTML，**其中某个 section 输出了未闭合 `<div>` 或破坏性嵌套**，导致后面 Liquid 模板从字符串拼接角度看是对的、但实际写入 HTML 文档时被解析器吞掉
- **修法**：**不要写嵌套子 wrapper**，直接把 `floema-page` class **合并到永远存在的 `<div class="page-content">` 上**：
  ```liquid
  <div class="page-content{% if request.page_type == 'index' %} floema-page page-wrapper{% endif %}" id="pageContent" role="main">
    {{ content_for_layout }}
  </div>
  ```
  这样：(a) wrapper 不依赖嵌套 if 块输出；(b) `page-content` 是母版根容器，永远在 DOM 里；(c) 一次性给同一节点加 floema-page 作用域
- **教训**：
  1. **Lumia/GemPages 派生主题里写嵌套 wrapper 是脆弱的**——母版 Liquid 注入的 HTML 不可控
  2. 共享 wrapper 时**复用现有母版 div 加 class**（merge）比新开 wrapper div（nest）稳得多
  3. 验证 wrapper 是否真的渲染：用 inline `<style> .your-wrapper::before { content: "marker" ... }</style>` 加在 head，刷新 preview 看 marker 是否出现 — marker 不出现就是 wrapper 不在 DOM

## P3-16：导航 / overlay sections 在 Shopify wrapper 里占了 layout 空间 → footer 下方一大块空白

- **症状**：Floema 14 个 sections 里 10-14 是 modal/drawer/grid/loading/curtain overlay（源站全是 `position:fixed` 浮层）。我把内部元素 `display:none` 之后 footer 下方仍多出大约 5 个 section 高的空白
- **根因**：Shopify 渲染每个 section 时外面包一个 `<div class="shopify-section--XX">` block-level wrapper，inner element 隐藏不影响 wrapper 本身。Lumia 母版给所有 section wrapper 加了 `min-height` / `padding` reset，每个 section 即使内容空白也至少占 100px+
- **修法**：直接在 wrapper 上 `display:contents !important`，让 wrapper 在布局中消失但子元素仍渲染（fixed overlay 仍工作）：
  ```css
  body.floema-index .shopify-section--floema-10-option-modal,
  body.floema-index .shopify-section--floema-11-contact-modal,
  ...
  { display: contents !important; }
  ```
- **教训**：
  1. **Shopify section wrapper 是 layout-affecting element**，永远要考虑它占多少 padding/margin/min-height
  2. 移植任何 modal/drawer/curtain 类 overlay section 时，**默认就给 wrapper 加 `display:contents`**
  3. 检查办法：DevTools Elements 里 hover 每个 `shopify-section--*` 看高亮蓝边的高度

---

## P3-15：自制 smooth-scroll 不同步 scrollbar drag → 拖一下 scrollbar 又跳回去

- **症状**：Floema 用 GSAP ScrollTrigger 配自制 lerp wheel handler，拖右侧 scrollbar 到新位置释放后页面瞬间滑回原点
- **根因**：自制 handler 维护内部 `target` + `current` 状态，只 hook `wheel` event。scrollbar 拖动直接改 `window.scrollY` 不走 wheel，下一次 wheel 计算 `target = oldTarget + delta`（旧 target 是被拖前的位置）→ 计算出来的目标位置回到旧位
- **修法**：rAF tick 里检查 `window.scrollY` vs `lastAppliedY`（我们 scrollTo 设的最后值），不一致就 resync `target=current=sy`：
  ```js
  if (Math.abs(sy - lastAppliedY) > 2) { target = sy; current = sy; lastAppliedY = sy; }
  ```
  wheel handler 计算时也用真实 `window.scrollY` 当 base 而不是 stale target
- **教训**：
  1. **自制 smooth-scroll 必须考虑所有 scroll 来源**：wheel / scrollbar drag / anchor jump / ScrollTrigger.scrollTo / DevTools / 键盘 PageUp/Down
  2. 维护 `lastAppliedY` 哨兵，每次 tick 比一遍 → 简单粗暴但能处理所有外部 scroll
  3. Lenis/Locomotive 都内置这个 resync，自制时容易漏

---

## P3-14：Vue lazy-imported component 把真实动画机制藏在另一个 chunk

- **症状**：Floema intro section liquid 里只看到 `<div class="floating-images"><span></span></div>` 空壳 + source CSS 写着 `.slider-image { animation: orbit 60s linear infinite; offset-path: ellipse(...) }`。按 CSS 复刻了 3 版（v1/v2/v3）都被用户说不对
- **根因**：Vue 主组件用了 `const r = defineAsyncComponent(() => import('./C4rlixGp.js'))` lazy-load 一个子组件，主 chunk 里看不到子组件 setup。`C4rlixGp.js` 才是真正创建 `<canvas>` + `new Worker(new URL('Images.worker-XXX.js', ...))`，Three.js scene 在 worker 里跑。CSS 里的 `.slider-image { animation: orbit }` 是组件未挂载时的 fallback 死代码
- **修法**：
  1. 在主 JS chunk 里 grep 所有 `me\(\(\)=>import\("([^"]+)"\)` 把 lazy import 路径列出来
  2. 对每个 chunk 看是否包含 `<canvas>`/`new Worker`/Three.js 关键词，确认是 3D scene 而不是普通组件
  3. Worker 文件**末尾 ~100 行**才是用户场景代码（开头 4000+ 行是 Three.js 库 bundle），`tail -c 8000` 提取
- **教训**：
  1. **看到 `__vite__mapDeps([...])` 或 `defineAsyncComponent` → 必须追那个 chunk**
  2. **Vue SFC 编译会把组件 scoped styles 全塞 `<head>` 里，组件本身没挂载时这些 CSS 就是死代码**。看到 `data-v-XXXXX` scope 的 CSS 选择器一定要去 inspect DOM 验证元素是否真的存在
  3. **`new Worker(new URL(...))` 是 3D 场景的高频架构**，移植到 Lumia 上简化成主线程 Three.js 即可（worker 主要省主线程性能，主线程也能跑）

---

## P3-13：3D scene 用 packed RGBA 纹理 + shader 采样，不是粒子系统

- **症状**：Floema collections-overview 的 `.shadows` div 第一眼以为是粒子叶子在飘，写了 60 片 ShapeGeometry 墨绿叶子 + 鼠标视差，用户截图原版后说"是软的圆形云朵不是锋利叶子"
- **根因**：源站用了 **packed_texture_3.png**（一张 PNG，4 个通道分别打包 4 种植物轮廓 silhouette）+ **noiseTexture.png** + 自定义 GLSL shader。Fragment shader 按 `uLayer` uniform 采样不同通道：
  ```glsl
  float tex = texture2D(uTexture, uv).r;           // layer 0
  if (uLayer == 1.0) tex = texture2D(uTexture, uv).g; // layer 1 — collections
  if (uLayer == 3.0) tex = 1.0 - texture2D(uTexture, uv).w; // layer 3 — madeToLast
  ```
  然后用 noiseTexture 扰动 alpha 制造软边、用 smoothstep mask 做圆形 vignette
- **修法**：
  1. PowerShell 下载 `/3d/shadows/packed_texture_3.png` + `noiseTexture.png`，上传成 Shopify assets
  2. ShaderMaterial 把源 vertex + fragment shader 整段抄过来（不要重写）
  3. 按 section 配置 uniforms：collections layer 1 / rotation π·1.9 / offset(-0.15,-0.15) / alpha 0.9 / cover:true
- **教训**：
  1. **看到 `texture2D(uTexture, uv).g` 这种通道采样 → 一定有预制的 packed texture**，找 `/3d/` `/assets/` `/textures/` 目录看有没有
  2. 用户描述「软」「云」「模糊」「不锋利」→ 必定是 shader-based effect 而不是粒子，重写思路：抄 shader + 抄纹理
  3. shader uniform 配置（rotation/offset/alpha/layer）按 section 不同，必须读源码 `defaults map` 里的 per-type 配置全抄

---

## P3-12：Liquid → JS 的 asset URL 桥接需要 `<script type="application/json">`

- **症状**：JS 需要拿到 41 张本地化图片的 Shopify CDN URL（绕 Sanity CORS），但 `asset_url` filter 只能在 .liquid 里渲染
- **根因**：JS 文件是静态 asset，不经 Liquid 渲染；硬编码 `cdn.shopify.com/s/files/1/{shop_id}/...` 路径会在主题切换时断
- **修法**：section liquid 里塞一个 JSON script tag，用 Liquid for 循环渲染：
  ```liquid
  <script type="application/json" id="floema-intro-image-urls">[
  {%- for f in img_array -%}
    "{{ f | asset_url }}"{%- unless forloop.last -%},{%- endunless -%}
  {%- endfor -%}
  ]</script>
  ```
  JS 端 `JSON.parse(document.getElementById('floema-intro-image-urls').textContent)` 读取。也可以用 `window.__floemaAssetUrls = {...}` 全局对象（少量 URL 时更简洁）
- **教训**：
  1. **Shopify CDN 路径包含 store ID，主题切换会变，永远不要在 JS 里硬编码**
  2. JSON script tag 是最稳的 Liquid→JS 数据桥，比 data-attribute 更适合大数组
  3. theme.liquid 里 `window.__namespace = { foo: "{{ ... }}", bar: "{{ ... }}" }` 比每个 section 都塞 JSON tag 更适合全局共享

---

## P3-11：跨域 CDN 图片 Three.js 加载失败 → 必须本地化为 Shopify asset

- **症状**：Floema intro tunnel 用源站同款 Sanity CDN URL（`cdn.sanity.io/images/535lnz3g/.../*.png`）作为 Three.js TextureLoader 输入，41 张图全部加载失败，场景只剩彩色 placeholder 方块在飞
- **根因**：Sanity CDN 对自家 hosted 域名（如 `www.floema.com`）发 CORS 头，但对 `himaxlimited.myshopify.com` 这种第三方域名要么不发 `Access-Control-Allow-Origin: *` 要么发了但 referer 校验失败。WebGL 上传纹理必须 CORS-clean 才能用，普通 `<img>` 标签可以容忍 tainted texture，但 Three.js 不行
- **修法**：
  1. PowerShell 写下载脚本 `Invoke-WebRequest` 批量拉源站 CDN 图到本地 `assets/` 目录
  2. Liquid section 里用 `{%- for f in img_array -%}"{{ f | asset_url }}"{%- endfor -%}` 渲染 JSON 数组到 `<script type="application/json" id="..."`
  3. JS 端 `JSON.parse(document.getElementById('...').textContent)` 拿到同源 Shopify CDN URL，TextureLoader 加载零阻力
- **教训**：
  1. **任何 3rd-party CDN 图片 + Three.js / WebGL canvas 一律要本地化**，不要奢望 CORS 头会自动通过
  2. 排查时先写 placeholder material（彩色方块）让 scene 永远有内容 —— 看到方块飞起来 = Three.js 工程对，CORS 才是问题
  3. Liquid `asset_url` 只能在 .liquid 文件里用，所以「JS 读 Liquid 计算的 URL」的桥梁就是塞一个 `<script type="application/json">` 标签
  4. 别在 JS 里硬编码 Shopify CDN 路径（`cdn.shopify.com/s/files/1/.../assets/xx.png`）——主题 ID 切换时会断
  5. Playbook 已加 §6.5「Three.js 外站资源必须先本地化」清单项（[03-animations.md](03-animations.md)）

---

## P3-10：CSS hint 与真实动画机制完全不符 — Vue SFC CSS 可能是死代码

- **症状**：Floema intro section 的 source CSS 里有 `.slider-image { animation: orbit 60s linear infinite; offset-path: ellipse(...); }` 看起来像绕椭圆旋转的特效。按这个写了 v1/v2/v3 三版 JS（注入 `<img class="slider-image">` + animation-delay 错峰），用户三次都说"不对"。最后用户截图原版是 **dense ring of images** 浮现 + 飞过
- **根因**：CSS 是 **死代码**（fallback 或老版本残留）。真实机制在 lazy-loaded Vue 子组件 `C4rlixGp.js` 里——它创建 `<canvas>` + Web Worker (`Images.worker-BWdQtf8a.js`)，用 Three.js 渲染 **3D Z-tunnel of textured planes**（46 个平面沿 Z 轴飞向相机，fibonacci 螺旋分布在半径 10 的环上）。CSS 里的 `.slider-image` 选择器在真实 DOM 中根本不存在
- **修法**：
  1. 删掉 v1/v2/v3 的 CSS orbit 实现
  2. 读源码 worker 文件尾部（前面 4000+ 行是 Three.js 库代码，最后 ~100 行才是用户场景）找到 `bn` constants 和 setup callback
  3. 把 Three.js scene 整套 port 过来：fibonacci lane、Z spawn、SPEED_START→SPEED_IDLE 衰减、scroll feedback、fog
- **教训**：
  1. **不要只看 source CSS 推 animation 机制**。Vue/Nuxt SFC 编译会把组件 scoped styles 全部塞 head，但 **JS 里的 dynamic `<canvas>` rendering 不会经过 CSS**
  2. 看到 `data-component=intro` + `floating-images` 这种容器后，先 grep 同名 JS 文件（`grep -lE 'floating-images|intro' _nuxt/*.js`）确认是 CSS-only 还是 JS-driven
  3. Vue lazy-import (`defineAsyncComponent` / `() => import('./X.js')`) 经常把重型 3D scene 藏在独立 chunk 里。看到 `__vite__mapDeps` 或 `() => me(() => import(...))` 就要追那个文件
  4. Web Worker (`new Worker(new URL(...))`) 是高性能 3D scene 的常见模式 — Lumia/Dawn 上 port 时简化成主线程 Three.js 即可
  5. 任何怀疑都先看 `_payload.json` + `_nuxt/*.js`，不要凭 CSS 猜机制

---

## P3-09：动态注入数据偷工减料 → 用户立刻发现（41 张 → 写了 16 张）

- **症状**：源站 intro section 用 41 张 product PNG 沿椭圆轨道环绕标题（Vue 运行时注入），我自作主张「精选 16 张」节省 token，用户截图后说"我要的是完全与原站一致 不要有偷工减料的步骤"
- **根因**：源站 Vue 运行时数据（图片/卡片/列表/任何 `v-for` 渲染的集合）扁平化时容易被当成"装饰性的多"，让人想"少几个看着也差不多"。但用户看的是 **数量是否对得上原版**，不是看着是否 OK。少了立刻露馅
- **修法**：
  1. 任何从 source `_payload.json` / `__NUXT_DATA__` 提取的数据集合（图片数组、collection list、product list、leaf count、wireframe count），**必须穷举不抽样**
  2. 提取时把 grep 结果的行数对一遍 source 静态 HTML / payload 里声明的数量
  3. 复刻 Three.js / Canvas 这种"看起来连续"的视觉效果时也一样——源站若声明 50 leaves，写 50 不写 30
- **教训**：
  1. **1:1 复刻 = 数量、顺序、字段都要对得上**。不要在数据量级上 "judgment call"，那是 source decisions、不是我的优化空间
  2. 数据节流只在 _bundle size_ 实际成为问题（>1MB CDN cost）时才做，且要写明跟原版的差异
  3. 任何 `const FOO_COUNT = N` 常量在文件头部加注释 `// source: 41 (verified from _payload.json intro array)`，让 review 时一眼能对账

---

## P3-08：Vue3 SFC scoped styles 漏抽 → 视觉骨架全丢

- **症状**：把 Nuxt SSG 站的 5 个外部 CSS（entry / ArticleCard / RevealImage / Footer / LowerTagline）扁平化后部署，页面看起来全无样式——文字是无衬线默认字体，没有任何 layout、颜色、间距
- **根因**：Vue3 SFC 编译时每个组件的 `<style scoped>` 会变成 inline `<style>` 块挂在 `<head>` 里（不打包到外部 CSS），靠 `data-v-XXXXXX` attribute selector 跟组件 DOM 匹配。Floema 站 head 里有 **68** 个这样的 inline style block，加起来 ~255KB。如果只扁平化外部 CSS，组件级样式全部丢失
- **修法**：写 `extract-{name}-inline-styles.mjs`，遍历 head 里所有 `<style>` 块的 innerText 合并成单个 `{prefix}-inline-styles.css`，加入 theme.liquid 的 CSS 加载列表（顺序：inline-styles 在最前 → 外部 5 个 CSS → base.css）。还要扫 inline-styles 里的 `url()` 引用一起重写（Floema 案例里有 @font-face 引用字体）
- **教训**：
  1. **Nuxt SSG 站 ≠ 全部 CSS 在外部文件**。Vue3 SFC 模式默认把 scoped styles 内联到 head。如果只跟着 `<link rel="stylesheet">` 抓 CSS，会漏 60-80%
  2. 立项第一件事除了 grep 外部 CSS，还要 grep `<style>` block 数量。`[regex]::Matches($html, '<style[^>]*>') | Measure-Object` 一行搞定
  3. inline-styles 也可能含 url() 引用（@font-face、background-image），扁平化 url 重写不能漏

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
