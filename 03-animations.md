# 03 — 动画 / 交互

> 复刻 GSAP 风格站点（特别是 Next.js + Tailwind v4）时，**先看** [08-nextjs-porting.md §8](08-nextjs-porting.md#8-source-动效完整-spec-速查crav-实战)。CRAV 项目沉淀了 8 类典型动效的完整 spec（pop 文字 / 贴纸 peel / jelly bar / 地图飞机 / cursor rope / 眼睛跟鼠标 / 杂耍球 / smooth scroll），从源 chunk 反推出来的参数（duration、ease、stagger、threshold、apex、spin 等）都能直接拿来用。
> 
> GSAP 付费 plugin（InertiaPlugin / MotionPathPlugin / DrawSVGPlugin / MorphSVGPlugin / 老 SplitText）的 **vanilla 替代** 见 [08-nextjs-porting.md §6](08-nextjs-porting.md#6-gsap-付费-plugin-vanilla-替代)。

## 1. 核心心法

| 场景 | 用什么 |
|---|---|
| 一次性入场（hero heading 加载淡入） | CSS `@keyframes animation` |
| 持续交互（hover、carousel 切换）需要可重放 | CSS `@keyframes animation` + remove/reflow/re-add 三件套 |
| 滚动驱动（scrub 风格的视差 / 缩放 / 描边） | `--fg-progress: 0..1` CSS 变量 + `calc()`，或 GSAP ScrollTrigger scrub |
| 状态平滑过渡（颜色、不透明度的 hover）| CSS `transition` |
| 鼠标跟随 / 沿路径动画 | vanilla GSAP，详 [08-nextjs-porting.md §6 §8](08-nextjs-porting.md#6-gsap-付费-plugin-vanilla-替代) |

## 2. Transition vs Animation —— 选错就重放不了

**transition 的致命弱点**：基于属性差异计算。同一帧内 remove + add class 会被浏览器合并成"无变化"，transition 不触发。即使 `void offsetWidth` 强制 reflow，元素只移动 ~1%，肉眼不可见。

**animation 没有这个问题**：移除 class → animation 整个卸下、元素瞬间回到基础态；reflow；加回 class → 全新 animation 实例从头跑。

### 标准重启三件套（必背）

```js
el.classList.remove('is-revealed');
void el.offsetWidth;           // 强制 reflow，让浏览器把"动画卸下"登记
el.classList.add('is-revealed');
```

### 反例 vs 正例

```css
/* ✗ 反例：用 transition，carousel 切换不重播 */
.fg-line {
  transform: translateY(110%);
  transition: transform 1.5s var(--fg-ease-out-cubic);
}
.is-revealed .fg-line { transform: translateY(0); }

/* ✓ 正例：用 animation，carousel 切换可重播 */
.fg-line { transform: translateY(110%); }
@keyframes fg-line-rise {
  from { transform: translateY(110%); }
  to   { transform: translateY(0); }
}
.is-revealed .fg-line {
  animation: fg-line-rise 1.5s var(--fg-ease-out-cubic) both;
}
```

## 3. 滚动驱动动画 `--fg-progress`

集中在 `assets/fg-scroll-progress.js`，给标了 `[data-fg-scroll]` 的元素挂一个 0..1 的 progress CSS 变量。

### 四种模式

| 模式 | 公式 | progress 0 时刻 | progress 1 时刻 | 用于 |
|---|---|---|---|---|
| `enter` (默认) | `(vh - rect.top) / (vh + rect.height)` | top 撞视口底 | bottom 撞视口顶 | 元素从底进入到从顶离开（全程） |
| `reveal` | `(vh - rect.top) / rect.height` | top 撞视口底 | bottom 撞视口底 | 入场即停止（footer 入场） |
| `leave` | `-rect.top / rect.height` | top 撞视口顶 | bottom 撞视口顶 | 离场专用（hero 视差） |
| `pin` | `-rect.top / (rect.height - vh)` | top 撞视口顶 | bottom 撞视口底 | 200svh sticky 整段 pin |

用法：
```liquid
<section class="fg-banner-cta" data-fg-scroll="enter">...</section>
<section class="fg-footer" data-fg-scroll="reveal">...</section>
<section class="fg-banner-showroom" data-fg-scroll="pin">...</section>
```

CSS 里直接用 `var(--fg-progress, 0)`：
```css
.fg-banner-showroom .background {
  transform: scale(calc(0.55 + var(--fg-progress, 0) * 0.45));
  opacity: calc(0.7 - var(--fg-progress, 0) * 0.4);
}
```

**默认值原则**：`var(--fg-progress, X)` 的 fallback X **必须等于"section 还在视口外的初始状态对应的 progress 值"**。enter / reveal / pin 三种模式 progress=0 时都是"section 还在视口下方"，所以 fallback 一律写 `0`，避免 JS 加载前后闪烁。

### Lerp 平滑（关键）

原生鼠标滚轮一次 tick 跳 80-100px，scroll-driven 动画如果直接跟着 `getBoundingClientRect().top` 算 progress，写到 transform 上看起来一卡一卡的。

`fg-scroll-progress.js` 在 rAF 循环里做 lerp 平滑：

```js
var LERP = 0.18;  // 每帧追上 18% 的目标值
function tick() {
  for (each section) {
    target = computeTarget(s);
    if (typeof current !== 'number') current = target;
    current += (target - current) * LERP;
    setCSS('--fg-progress', current);
  }
  if (任何 section 还没追到 target) requestAnimationFrame(tick);
}
```

效果：滚一下，progress 用 8-10 帧（约 130ms）平滑追到目标值，transform 跟着平滑过渡。等价于自制的轻量 Lenis，不需要引入第三方库。

调节：
- `LERP = 0.18` 适合中等丝滑感
- `LERP = 0.3-0.5` 接近原生（响应更紧、平滑感弱）
- `LERP = 0.08-0.12` 非常丝滑但有明显跟随延迟

**为什么不能用 CSS `transition: transform 0.1s linear`**：scroll-driven 的 transform 值在每个 scroll 事件都重新设置，transition 会被反复打断重启，结果反而抖。Lerp 在 JS 层把 progress 平滑后再写给 CSS，CSS 只负责响应静态值——transition 不需要。

### 仅 progress lerp 不够 — 需要 smooth scroll 配合

只 lerp progress 能让 transform 看着丝滑，但**页面文字本身**还是跟原生鼠标滚轮的 80-100px tick 一起跳。视觉上：transform 缓慢移动，文字一卡一卡，反差感反而更僵硬。

原站用 Lenis（~7KB 库）拦截 wheel、维护虚拟 scroll 位置、用 lerp 把 `window.scrollTo` 平滑过去。整个页面（包括文字）跟着虚拟 scroll 平滑移动。

`fg-scroll-progress.js` 末尾内置了 ~40 行的 Lenis 等价实现：

```js
var current = window.scrollY;
var target = current;
var rafId = null;
var SCROLL_LERP = 0.06;  // 1.2-1.5s deceleration tail; tune per taste

function normalizeDelta(e) {
  if (e.deltaMode === 1) return e.deltaY * 16;            // Firefox lines mode
  if (e.deltaMode === 2) return e.deltaY * window.innerHeight;  // pages mode
  return e.deltaY;                                         // default px mode
}

function loop() {
  var d = target - current;
  if (Math.abs(d) < 0.4) { window.scrollTo(0, target); rafId = null; return; }
  current += d * SCROLL_LERP;
  window.scrollTo(0, current);
  rafId = requestAnimationFrame(loop);
}

document.addEventListener('wheel', function (e) {
  if (e.ctrlKey) return;  // 不要破坏 pinch zoom
  e.preventDefault();
  if (rafId === null) {
    // resync — native scroll (键盘/锚点/scrollbar drag) 可能改了 scrollY
    current = window.scrollY;
    target = current;
  }
  target = clamp(target + normalizeDelta(e));
  if (rafId === null) rafId = requestAnimationFrame(loop);
}, { passive: false });
```

要点：
1. **只拦 wheel**，touch / 键盘 / scrollbar 走原生（mobile 有自带 momentum，键盘/scrollbar 用户不会怪它没动画）
2. **每次 wheel 之前 resync**：原生 scroll 改了 scrollY 时把 current/target 拉回来，避免下一次 wheel 从过期位置加 delta
3. **触屏跳过**：`@media (hover: hover) and (pointer: fine)` 检查，触屏设备直接 return
4. **prefers-reduced-motion 也跳过**，照顾无障碍偏好
5. **`normalizeDelta` 处理 `deltaMode`**：Firefox 的鼠标滚轮可能报 lines（×16）或 pages（×viewport）而不是 pixels。不处理的话 Firefox 用户每次滚轮只走几 px，几乎不动
6. **跟 progress lerp 双层叠加**：scroll 位置被 SCROLL_LERP（0.06）平滑 → 派生的 progress 再被 LERP（0.08）平滑一层。视觉很丝滑

**SCROLL_LERP 调参指南**：
- `0.10`（Lenis 默认）：~500ms 减速，感觉紧
- `0.08`：~700ms 减速
- `0.06`：~1.2-1.5s 减速，长尾，适合高端展示站
- `0.04`：~2s+ 减速，过于飘，不推荐
- 快速滚一下后页面滑行的时长 = decel 时长。用户感受"原版滑动快时也是缓慢上升下降"= 想要更长的尾巴

副作用：`scroll-behavior: smooth` 这种 CSS 属性会跟我们的 JS 平滑冲突，**不要**用。如果有需要程序滚到锚点的地方，调用 `window.scrollTo({ top, behavior: 'auto' })`，让我们的 wheel handler 自动 resync。

### 嵌套子元素的视差测量

如果要按子元素实际高度差做平移（assets-duo 左图比右图矮的 parallax）：

```liquid
<div data-fg-parallax-track>      <!-- 容器 -->
  <div data-fg-parallax>...img... <!-- 较矮项目，JS 会算出高度差 -->
  <div data-fg-parallax>...img... <!-- 较高项目，diff < 4 自动跳过 -->
</div>
```

CSS：
```css
.fg-assets-duo [data-fg-parallax] {
  transform: translateY(var(--fg-parallax-y, 0px));
}
```

**注意**：测量类图片必须 `loading="eager"` 或挂 `load` 事件再测，懒加载图片 `clientHeight = 0` 会算错。

## 4. 行拆分 reveal（hero heading + 每个 section 大标题）

带 `data-fg-reveal-lines` 的元素会被 JS 按渲染后的换行拆成 `<span class="fg-line-mask"><span class="fg-line">行文本</span></span>`，每行 mask 用 overflow:hidden 当遮罩，内层用 `@keyframes fg-line-rise` 从 `translateY(110%)` 升到 0，按 `--fg-line-i` 错开 0.1s。

```liquid
<h2 class="base-heading" data-fg-reveal-lines>{{ s.heading }}</h2>
```

**触发方式**：默认用 IntersectionObserver（threshold 0.1），每个元素滚到 10% 可见时单独触发。**不要**改成"加载时一次性触发所有"——下方的标题用户还没看见就播完了。

**字体等待**：拆行**必须**在 `document.fonts.ready` 之后，否则用回退字体测，加载完 Aeonik 后换行位置错位。

**Carousel 切换重触发**（reviews 这种）：用上面 §2 的三件套，对新 active 的 blockquote 执行 `remove → void offsetWidth → add`。

## 5. Hover 重放型动画（reviews 箭头按钮）

需求：鼠标移入 → 箭头朝指向方向"飞出"+ 新箭头从相反方向飞回。

**纯 CSS** 用一段 keyframes + 中间瞬切：

```css
@keyframes fg-arrow-loop {
  0%   { transform: translateX(0%);   animation-timing-function: cubic-bezier(0.55, 0.055, 0.675, 0.19); }
  50%  { transform: translateX(200%); animation-timing-function: steps(1, end); }
  51%  { transform: translateX(-200%); animation-timing-function: cubic-bezier(0.215, 0.61, 0.355, 1); }
  100% { transform: translateX(0%); }
}
@media (hover: hover) {
  .fg-reviews .arrow-nav .arrow-btn:hover .nav-svg {
    animation: fg-arrow-loop 1s both;
  }
}
```

`steps(1, end)` 实现 50% → 51% 的瞬切（看上去是"在屏幕边缘的瞬切"）。

`prev` 按钮父级 `transform: rotate(180deg)`，子 SVG 的 translateX 视觉自动反向——两个按钮都"朝指向方向飞出"。

## 6. 自定义 cursor 系统

全局单一 `.fg-cursor` 元素（在 `body` 末尾或 layout 里渲染一次），跟随 `mousemove` 的 `transform: translate(x, y)`。元素加 `data-fg-cursor="Label"` 触发 `mouseenter` 显示、`mouseleave` 隐藏。

```html
<div class="fg-cursor" aria-hidden="true">View</div>
```

```css
.fg-cursor {
  position: fixed;
  top: 0; left: 0;
  width: 10rem; height: 4.4rem;
  margin: 2rem 0 0 2rem;  /* 让 cursor 卡片出现在鼠标 tip 的右下角 */
  background: linear-gradient(180deg, #ffffff26, #fff3);
  backdrop-filter: blur(2rem);
  pointer-events: none;
  opacity: 0;
  z-index: 99;
}
.fg-cursor.is-visible { opacity: 1; }
@media (max-width: 600px), (hover: none) { .fg-cursor { display: none } }
```

```js
document.addEventListener('mousemove', function (e) {
  cursor.style.transform = 'translate(' + e.clientX + 'px, ' + e.clientY + 'px)';
});
document.querySelectorAll('[data-fg-cursor]').forEach(function (el) {
  el.addEventListener('mouseenter', function () {
    cursor.textContent = el.getAttribute('data-fg-cursor');
    cursor.classList.add('is-visible');
  });
  el.addEventListener('mouseleave', function () { cursor.classList.remove('is-visible') });
});
```

**关键点**：
- 用 `transform: translate(x, y)` 而非 `left/top`，性能差一个数量级
- `pointer-events: none` 不挡点击
- `(hover: hover)` 媒体查询禁用触屏设备

## 7. GSAP → CSS 翻译表

| GSAP | CSS / JS 等价 | 备注 |
|---|---|---|
| `gsap.to(el, { x: 100 })` | `transform: translateX(100px)` | x 是**像素** |
| `gsap.to(el, { x: c*65 }), c = w/1600` | `translateX(calc(6.5 * var(--fg-font-s)))` | **不是 65**，是 6.5 |
| `gsap.to(el, { xPercent: 200 })` | `transform: translateX(200%)` | % |
| `gsap.to(el, { yPercent: -50 })` | `transform: translateY(-50%)` | |
| `gsap.to(el, { scale: 1.1 })` | `transform: scale(1.1)` | |
| `gsap.to(el, { autoAlpha: 0 })` | `opacity: 0; visibility: hidden` | |
| `power3.in` | `cubic-bezier(0.55, 0.055, 0.675, 0.19)` | 也叫 easeInCubic |
| `power3.out` | `cubic-bezier(0.215, 0.61, 0.355, 1)` | easeOutCubic |
| `power3.inOut` | `cubic-bezier(0.645, 0.045, 0.355, 1)` | |
| `power4.in` | `cubic-bezier(0.755, 0.05, 0.855, 0.06)` | |
| `ease: "none"` | `linear` | |
| `scrollTrigger: { scrub: true, start, end }` | `--fg-progress` (本文档 §3) | |
| `scrollTrigger: { onEnter }` | IntersectionObserver | |
| `drawSVG: "0% 50%"` | `stroke-dasharray` + `stroke-dashoffset` + `pathLength="1"` | 详见 §8 |
| `stagger: 0.1` | `animation-delay: calc(var(--i) * 0.1s)` + JS 设置 `--i` | |

## 8. SVG 描边动画（banner-cta 装饰线）

GSAP plugin `drawSVG` 用 CSS 替代：

```html
<svg ...>
  <path d="..." pathLength="1" stroke-dasharray="1 1" />
</svg>
```

```css
.banner-cta-svg path {
  stroke-dashoffset: calc(2 * var(--fg-progress, 0) - 1);
}
```

- `pathLength="1"` 把路径长度归一化到 1
- `stroke-dasharray="1 1"` = 一段 dash + 一段 gap，各占整条路径
- `stroke-dashoffset: -1` → dash 完全推到 gap 位置 → 不可见
- `stroke-dashoffset: 0` → dash 正好填满 → 完全可见
- `stroke-dashoffset: 1` → 从另一端推出 → 又不可见

progress 0 → 0.5 → 1 映射到 dashoffset `-1 → 0 → 1`：进入视口时画入、离开时画出。

## 9. Sticky parallax (200svh) 配方（banner-showroom 同款）

```liquid
<section class="fg-banner-showroom" data-fg-scroll="pin">
  <h2 class="base-title">{{ s.section_title }}</h2>
  <div class="container">                <!-- sticky 容器 -->
    <div class="background">             <!-- 缩放/淡入 的图层 -->
      <img class="base-image" ...>
    </div>
    <div class="content">                <!-- 左右两栏 -->
      <div class="col-one">heading</div>
      <div class="col-two">address + cta</div>
    </div>
  </div>
</section>
```

```css
.fg-banner-showroom { overflow: clip; }  /* 不能 hidden */
@media (min-width: 601px) {
  .fg-banner-showroom { height: 200svh; padding-top: 4rem; }
  .fg-banner-showroom .container {
    position: sticky; top: 0; height: 100svh;
    display: flex; align-items: center;
  }
  .fg-banner-showroom .background {
    transform: scale(calc(0.55 + var(--fg-progress, 0) * 0.45));
    opacity: calc(0.7 - var(--fg-progress, 0) * 0.4);
  }
  .fg-banner-showroom .col-one {
    transform: translateX(calc((1 - var(--fg-progress, 0)) * 6.5 * var(--fg-font-s)));
  }
  .fg-banner-showroom .col-two {
    transform: translateX(calc((1 - var(--fg-progress, 0)) * -6.5 * var(--fg-font-s)));
  }
}
```

要点：
1. section 高度 200svh，inner sticky 100svh
2. **section 必须 `overflow: clip`**，不能 `hidden`（sticky 失效）
3. progress=0 = 入场初态：image 缩小 + 不透 + 两栏离场远
4. translateX 数值用 `6.5 * fg-font-s` 不是 `65 *`

## 10. Hero 滚动视差（leave 模式典型用法）

需求：用户滚动时，前景文字"飞得快"，背景图"慢一拍"（经典 parallax depth）。

原站做法：`<div class="asset">` 包住背景视频/图，scroll-trigger `top top → bottom top`，asset 用 `gsap.fromTo({ yPercent: 0 }, { yPercent: 50 })`。背景被翻译 +50% 自身高度向下 = 跟随页面滚动的速度只有一半。

CSS 复刻：

```liquid
<section class="fg-home-header" data-fg-scroll="leave">
  <div class="container">
    <p class="base-heading" data-fg-reveal-lines>...</p>
    <div class="indicator">Scroll to explore</div>
    <h2 class="base-title">...</h2>
    <p class="text">...</p>
  </div>
  <div class="background">
    <div class="asset">              <!-- 这层做视差 -->
      <video class="video" />
    </div>
  </div>
</section>
```

```css
.fg-home-header { overflow: hidden; }  /* 翻译出去的部分要裁掉 */
@media (min-width: 601px) {
  .fg-home-header .asset {
    transform: translateY(calc(var(--fg-progress, 0) * 50%));
    will-change: transform;
  }
}
/* "Scroll to explore" 在前 10% 滚动内迅速淡出（原站 duration: 0.1） */
.fg-home-header .indicator {
  opacity: max(0, calc(1 - var(--fg-progress, 0) * 10));
}
```

要点：
1. `leave` 模式：progress=0 = section 顶贴视口顶（初始全屏可见），progress=1 = section 底贴视口顶（完全滚过去）
2. asset translateY 5
0% = 50svh（section 是 100svh），跟随页面滚动 -100svh 的同时下移 50svh = 净位移 -50svh = 一半速度
3. `.indicator` 用 `max(0, ...)` 防止负值（CSS 会自动 clamp 但显式更稳）
4. 这个 indicator 入场时不要加 `animation: fg-rise`，否则跟 scroll-driven opacity 打架。要的话用嵌套 wrap div（外层做 intro fade，内层做 scroll fade）

## 11. 双向 toggle（菜单 / 抽屉等）— animation vs transition 取舍

**规则**：能用 transition 就别用 animation。

为什么：

| 场景 | 用 animation | 用 transition |
|---|---|---|
| 入场动画 + 完成停在终态 | ✓ `animation-fill-mode: both` | ✓ 切换 class 触发 |
| 切换 class 想反向播放 | ✗ class 移除 → animation 卸下、元素瞬间回到基础态（snap close） | ✓ class 移除 → 反向 transition 平滑过渡 |
| 需要"开 / 关方向不同的 delay" | ✗ delay 写在 class 里两次 keyframes | ✓ 把 `transition-delay` 放在 `.is-open` 选择器里 |
| Carousel/可重放 | ✓ 重启三件套 | ✗ 同一帧切换被合并 |

**asymmetric delay 技巧**（菜单打开有 stagger 延迟、关闭无延迟）：

```css
/* 基础态：opacity 0，transition 不带 delay */
.fg-menu-title {
  opacity: 0;
  transition: opacity 0.5s var(--fg-ease-in-out-cubic);
}
/* 打开态：opacity 1 + 仅这里加 delay */
.fg-menu.is-open .fg-menu-title {
  opacity: 1;
  transition-delay: 0.3s;
}
```

- 加 `.is-open`：transition-delay 变 0.3s，opacity 0→1 走 0.3s 后开始
- 移除 `.is-open`：transition-delay 回到基础态的 0，opacity 1→0 立刻开始

完美匹配原站 GSAP `fromTo({autoAlpha:0},{delay:.3,autoAlpha:1})` 的开 + `to({autoAlpha:0})` 的关（无 delay）。

**多元素 stagger 用 CSS 变量**：

```css
.fg-menu-links .link {
  transform: translateY(150%);
  transition: transform 0.8s var(--fg-ease-in-out-cubic) var(--fg-link-delay, 0s);
}
.fg-menu.is-open .fg-menu-links .link { transform: translateY(0); }

.fg-menu-links .mask:nth-child(1) .link { --fg-link-delay: 0.02s; }
.fg-menu-links .mask:nth-child(2) .link { --fg-link-delay: 0.04s; }
/* ... */
```

`transition-delay` 里嵌套 `var(...)`，每个 link 通过 `:nth-child` 设自己的 delay 值。

## 12. 模态从某个固定点"长出来"配方（菜单展开）

需求：菜单不是从屏幕中央 fade 出来，而是从某个具体位置（例如关闭按钮上方）"长出一个小长方形 → 扩展成完整面板"。原站 GSAP `me.fromTo(bg, { scale: 0 }, { scale: 1, ease: power3.inOut })` + CSS `transform-origin: center bottom`。

**第一选择：背景直接放容器上，整个容器做 scale**

```css
.fg-menu-panel {
  position: absolute;
  left: 50%;
  bottom: calc(10 * var(--fg-font-s));    /* 锚在底部，不是垂直居中 */
  background: color-mix(in srgb, var(--fg-color-black) 88%, transparent);
  backdrop-filter: blur(2rem);
  transform: translateX(-50%) scale(0);
  transform-origin: center bottom;         /* ← 关键：从底部中线展开 */
  transition: transform 0.5s var(--fg-ease-in-out-cubic);
}
.fg-menu.is-open .fg-menu-panel {
  transform: translateX(-50%) scale(1);
}
```

- 打开：scale 0 → 1，从底部中点（紧贴关闭 pill 上方）长出来
- 关闭：scale 1 → 0，从同一原点缩回
- 内容自动跟着缩放（scale=0 时整体不可见，scale=1 时完整）

**为什么不做"独立背景层 + 单独 scale"**：参见 [02-css-cascade.md §10](02-css-cascade.md) —— z-index: -1 + transform 容器 + overflow + isolation 组合在不少浏览器会让负 z-index 子元素整体不可见。如果不是必须分层（比如不需要内容跟背景独立动画），就别多拆一层。

**配合：内容 stagger 入场延迟到 scale 完成之后**

容器 scale 0.5s，内容（title / meta / CTA）opacity 入场 delay 0.3s + duration 0.5s = 0.8s 结束。视觉上：
- 0-0.3s：盒子从下方一点长出 60% 大小，里面空
- 0.3-0.5s：盒子继续长到 100%，文字开始淡入
- 0.5-0.8s：文字渐显完成
- Links 同时按 stagger 0.02s 各自从 `translateY(150%)` 升起

像盒子先撑开、内容随后浮现。

## 13. 滚到底自动开菜单

```js
var bottomSettleTimer = null;

function atBottom() {
  return window.scrollY >= document.documentElement.scrollHeight - window.innerHeight - 2;
}

function onScroll() {
  if (atBottom()) {
    if (menu.classList.contains('is-open') || userDismissedAtBottom) return;
    if (bottomSettleTimer) return;
    // Settle delay: 用户到底后等 250ms 仍在底部才打开，避免边滚边触发
    bottomSettleTimer = setTimeout(function () {
      bottomSettleTimer = null;
      if (atBottom() && !menu.classList.contains('is-open') && !userDismissedAtBottom) {
        open(/* isAuto */ true);
      }
    }, 250);
  } else {
    if (bottomSettleTimer) { clearTimeout(bottomSettleTimer); bottomSettleTimer = null; }
    userDismissedAtBottom = false;
    if (autoOpened && menu.classList.contains('is-open')) close();
  }
}
```

要点：
- **不锁 body 滚动**（手动开才锁），方便往上滚自动关
- **settle 延迟 250ms**：避免用户还在快速滚动时菜单弹出来；只有真正停在底部 250ms 才触发
- 用 `userDismissedAtBottom` 标志避免"在底部按 X 关掉后立刻又被滚动事件重新打开"；用户滚开底部再回来时重置
