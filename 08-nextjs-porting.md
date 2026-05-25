# 08 — Next.js 15 + Tailwind v4 移植专题

CRAV（cravburgers.shop）项目沉淀。源站特征：Next.js 15 App Router + Turbopack + Tailwind v4 + GSAP + Lenis + React Server Components hydration。

---

## 1. 立项判定

接到 Next.js 站任务时先 grep 这几个标记：

| Grep 关键字 | 命中说明 |
|---|---|
| `globalThis.TURBOPACK` | Next.js 15 / Turbopack |
| `@layer utilities` in CSS | Tailwind v4 (CSS Layer cascade — 必 unwrap，见 [BUGS.md P4-01](BUGS.md#p4-01)) |
| `modak_xxxxxx-module__variable` 之类 html class | next/font 模块化字体加载 |
| `_next/image?url=...&w=...` srcSet | Next.js image optimizer |
| `data-rsc` / `<!--$-->` 注释 | React Server Components SSR |

---

## 2. 资源抓取的特殊问题

### 2.1 `_next/image` 优化器输出 JPEG 丢 alpha

详见 [BUGS.md P4-05](BUGS.md#p4-05)。**透明 PNG/WebP 必须从 raw 路径下载**：

```js
// fetch-{prefix}-originals.mjs
for (const name of imgs) {
  const url = `https://${domain}/img-webp/${name}.webp`;
  const resp = await p.context().request.get(url);
  if (resp.status() === 200) fs.writeFileSync(dst, await resp.body());
}
// 额外下载 /img/ raw PNG（如 plane.png, burgerselfie.png, burger-boy.png）
```

### 2.2 next/font 的 CSS variable 不在 :root，在 html.module_class 上

源站 `<html class="modak_e8c77bb9-module__To9VbG__variable mouse_memoirs_a09dbcb8-module__HOyvqG__variable ...">`。CSS 里：

```css
.modak_e8c77bb9-module__To9VbG__variable { --font-modak: "Modak", "Modak Fallback" }
```

Shopify 主题的 `<html>` 没有这些 module class → `--font-modak` undefined → `font-family: var(--font-modak)` 回落到 serif/默认 → 字体不对 + 字号比例错乱。

**修法**：在 `{prefix}-base.css` 顶端把 CSS vars 提到 `:root`：

```css
body.{prefix}-index, html.{prefix}-html, :root {
  --font-modak: "Modak", "Modak Fallback", "Modak Extended", serif;
  --font-mouse-memoirs: "Mouse Memoirs", "Mouse Memoirs Fallback", system-ui, sans-serif;
}
```

### 2.3 next/font 的 @font-face url 是 hash 命名

`crav-app.css` 抓下来后 `@font-face` 里是 `url(../media/e24507cc2fd99222-s.p.116v.fn8s2.9a.woff2)`。需要：

1. 从 `_next/static/media/` 下载这些 hash woff2
2. 重命名为人类可读：`crav-modak.woff2` / `crav-mouse-memoirs.woff2`
3. 改写 chunk CSS 里 `url(...)`：`rewrite-{prefix}-css.mjs` 做一遍 hash→flat 映射替换

**容易踩**：Next.js 多个 hash 文件对应 unicode-range 切片（Latin / Extended Latin / Devanagari）。**字形对照时不要搞反**：
- `1c549cf76d17aa84-s.p.0duk76ik6drf0.woff2` (preloaded in head) → **Mouse Memoirs latin**
- `e24507cc2fd99222-s.p.116v.fn8s2.9a.woff2` (preloaded in head) → **Modak latin**
- `8eef5a389a468817-s.02i1v5~1n-qbz.woff2` → **Modak extended-latin**

判断办法：把 woff2 文件名跟 chunk CSS 里的 `@font-face { font-family: ... }` 块映射，font-family 名字就是该文件实际字形。CRAV 项目首次部署就把名字写反了，后来发现 hero 文字字体不对才回滚交换。

---

## 3. Tailwind v4 @layer cascade 必须 unwrap

详见 [BUGS.md P4-01](BUGS.md#p4-01)。这是 **Tailwind v4 移植到任何有母版 CSS 的环境时的首要陷阱**。CSS Cascade Layers 规范：unlayered 样式永远赢 layered，与 specificity 无关。

**unwrap 脚本**（`unwrap-{prefix}-layers.mjs`）：

```js
import fs from 'node:fs';
const path = 'assets/{prefix}-app.css';
let css = fs.readFileSync(path, 'utf-8');
const out = [];
let i = 0;
const len = css.length;
while (i < len) {
  if (css.startsWith('@layer', i)) {
    const m = /^@layer\s+[\w\s,]+\s*[;{]/.exec(css.slice(i));
    if (!m) { out.push(css[i]); i++; continue; }
    const headEnd = i + m[0].length - 1;
    if (css[headEnd] === ';') { i = headEnd + 1; continue; } // @layer foo, bar; → drop
    // Block: count braces from headEnd+1 to find matching }, accounting for string literals
    let depth = 1, j = headEnd + 1;
    while (j < len && depth > 0) {
      const c = css[j];
      if (c === '"' || c === "'") {
        const q = c; j++;
        while (j < len && css[j] !== q) { if (css[j] === '\\') j++; j++; }
      } else if (c === '{') depth++;
      else if (c === '}') depth--;
      j++;
    }
    out.push(css.slice(headEnd + 1, j - 1));  // inner content
    i = j;
  } else { out.push(css[i]); i++; }
}
fs.writeFileSync(path, out.join(''));
```

跑完后 grep `@layer` 应该 0 个匹配。Tailwind utilities 现在以 (0,1,0) specificity 跟 Lumia 的 element selector (0,0,1) 公平竞争。

---

## 4. Tailwind utility class 防御 cascade

### 4.1 不要给 inheritable property 加 `!important` 在父级

详见 [BUGS.md P4-02](BUGS.md#p4-02)。`body.{prefix}-index` 上写 `color: ... !important; font-family: ... !important` 会通过继承传染所有子元素，把 Tailwind 的 `.text-red`、`.font-modak` 压死。

**只对 non-inherited 属性用 !important**：

| 属性类型 | 例子 | 父级 !important 安全？ |
|---|---|---|
| Non-inherited | `background`, `display`, `margin`, `padding`, `width`, `border` | ✓ 安全 |
| Inherited | `color`, `font-family`, `font-size`, `line-height`, `letter-spacing`, `visibility`, `cursor`, `white-space` | ✗ 会传染 |

### 4.2 不要用 `.{prefix}-page` wrapper-prefix 给通用元素加规则

详见 [BUGS.md P4-03](BUGS.md#p4-03)。`.crav-page h2 { line-height: inherit }` (spec 0,1,1) 反伤 Tailwind `.leading-\[\.75\]` (spec 0,1,0)。

**正确做法**：要去掉母版的 h2/p 默认样式，直接 override 母版的具体 selector（`.h2-style, h2`）用同样 specificity，不要套 wrapper prefix。

### 4.3 中和母版 wrapper-级继承

母版常用 `.page-content`、`.modal-content`、`.body` 等 wrapper class 偷塞 inherited properties（见 [BUGS.md P4-04](BUGS.md#p4-04)）。中和方式：

```css
.{prefix}-page.page-content {
  font-size: inherit;
  line-height: normal;
}
```

更高 specificity (0,2,0) 比单 `.page-content` (0,1,0) 赢，值用 `inherit` 让继承链回到 body/html 默认。

---

## 5. Char-split spans 不能 prettify 换行

详见 [BUGS.md P4-06](BUGS.md#p4-06)。源站 minified `<span>C</span><span>R</span>...` 间无空白，我的 extract-body 脚本 prettify 时给每个 span 单独一行，HTML parser 把换行+缩进当成 whitespace 文本节点，inline-block 元素之间渲染为一个空格字符（35vw=504px 下 ~118px）。

**修法**：在 `extract-{prefix}-body.mjs` 里给 char-split spans 特殊处理 — 同组紧邻的 `<span data-pop>` 拼回一行：

```js
function prettify(s) {
  return s
    .replace(/></g, '>\n<')
    .split('\n')
    .map(l => l.trim())
    .filter(Boolean)
    .join('\n');
}
// 后处理：紧邻 <span data-pop> 拼回一行
function collapseCharSplit(html) {
  return html.replace(
    /(<span[^>]*data-pop="true"[^>]*>[^<]<\/span>\s*)+/g,
    m => m.replace(/\s+/g, '')
  );
}
```

---

## 6. GSAP 付费 plugin vanilla 替代

详见 [BUGS.md P4-08](BUGS.md#p4-08)。Next.js 站经常用 InertiaPlugin / MotionPathPlugin / DrawSVGPlugin / MorphSVGPlugin / 老 SplitText。免费版 GSAP 没这些。

| 付费 plugin | Vanilla 替代 | 视觉损失 |
|---|---|---|
| MotionPathPlugin | `SVGGeometryElement.getPointAtLength()` + ScrollTrigger scrub | 无（手动计算 autoRotate angle 即可） |
| InertiaPlugin | CSS `transition: cubic-bezier(.34,1.56,.64,1)` + mousemove/leave handlers | 失 momentum，弹性接近 |
| DrawSVGPlugin | `stroke-dasharray` + `stroke-dashoffset` 手 tween | 无 |
| MorphSVGPlugin (path morph) | 自己写 path "d" interpolator，每帧 `path.setAttribute('d', ...)` | 简单形状无损 |
| 老 SplitText | 自己 `text.split(/(\s+)/).filter(Boolean).map(w => <span>w</span>)` | 无 |

CRAV 项目 [crav-app.js initMapPlane](d:/shopify_try/crav-theme/assets/crav-app.js) 是 MotionPath vanilla 替代的参考实现。

---

## 7. React 控制的"初始隐藏"状态必须清

源站很多元素带 `style="visibility:hidden"`、`style="opacity:0"`、`class="opacity-0"`：

- `<span aria-hidden="true" style="visibility:hidden">` — pop 文字初始隐藏，等 GSAP onEnter 设 visible
- `<div class="w-full h-full opacity-0">` — hero burger 图片初始 opacity:0，等 entrance 动画
- 14 个 loader balls 都有 `style="opacity:0"`

**JS 必须主动清这些状态**，因为我们的 GSAP runtime 重新接管动画。`crav-app.js` 里：

```js
function clearZeroStates() {
  document.querySelectorAll('[style*="visibility:hidden"]').forEach((el) => {
    el.style.visibility = '';
  });
  document.querySelectorAll('[style*="opacity:0"], [style*="opacity: 0"]').forEach((el) => {
    el.setAttribute('data-crav-was-zero', '1');
    el.style.opacity = '';
  });
}
```

**注意 Tailwind class `.opacity-0`、`.invisible` 等不是 inline style**，需要单独处理（直接改 className 或在 liquid 里删）。

---

## 8. Source 动效完整 spec 速查（CRAV 实战）

每条都包含从 chunk 反推出来的完整参数。复刻新 Next.js 站时按这个清单逐项找对应实现。

### 8.1 data-pop 文字入场

源 (chunk `0dvuw12hbr-ev.js`)：

```js
gsap.set(pops, {opacity:0, scale:0, y:0, rotation:0, transformOrigin:"50% 90%", force3D:true});
ScrollTrigger.create({trigger: host, start: 'top 88%', once: true,
  onEnter: () => gsap.fromTo(pops,
    {opacity:0, scale:0, y:()=>gsap.utils.random(18,40), rotation:()=>gsap.utils.random(-16,16)},
    {opacity:1, scale:1, y:0, rotation:0, duration:.72, ease:'back.out(2.35)', stagger:.055, overwrite:'auto'}
  )});
```

Defaults: `delay:0, split:"words", stagger:.055, duration:.72, ease:"back.out(2.35)", scrollStart:"top 88%", once:true`。

### 8.2 Sticker peel (scroll-driven)

```js
const peel = {p: 1};
gsap.to(peel, {p: 0, ease: 'none',
  scrollTrigger: {trigger: host, start: '-120% 50%', end: '30% 50%', scrub: 1},
  onUpdate: () => host.style.setProperty('--peel-amount', peel.p)
});
// 加 mousemove 跟随 svg feSpecularLighting > fePointLight x/y
```

### 8.3 Jelly bar morph + scrub scaleY

Yoyo timeline 持续做 path d 形变（3 个 control points 各自 1s/2s/1.8s sine.inOut tween），加 ScrollTrigger scrub 让 svg `scaleY: 1.5` (desktop) / `1.2` (mobile)。

### 8.4 Map plane 沿 SVG 路径

```js
const len = path.getTotalLength();
function setAt(progress) {
  const p1 = path.getPointAtLength(progress * len);
  const p2 = path.getPointAtLength(Math.min(len, progress * len + 1));
  const angle = Math.atan2(p2.y - p1.y, p2.x - p1.x) * 180 / Math.PI - 90;
  // map SVG viewBox coords (1728x2176) to actual container px
  el.style.transform = `translate3d(${px}px, ${py}px, 0) translate(-50%, -50%) rotate(${angle}deg)`;
}
ScrollTrigger.create({trigger: section, start: '-10% top', end: '75%', scrub: 1,
  onUpdate: (st) => setAt(st.progress)
});
// 加 country labels reveal at progress thresholds [.14, .28, .5, .68, .91]
```

### 8.5 Cursor rope (4 ingredient dots on chain)

详见 [BUGS.md P4-09](BUGS.md#p4-09)。8 段链 + 4 个 knot dots at indices [0,3,5,9]。每 mousemove：
1. `gsap.to(segLen, {v: 8, duration: .01, overwrite: true})` 段间距复位
2. `setTimeout(() => gsap.to(segLen, {v: 0, duration: .5, ease: 'expo.out'}), 0)` 排队 collapse

链段 physics：head 跟 mouse lerp 0.45；后续段被前一段拉拽（`if (dist > segLen) { ... }`）。

### 8.6 Eye-follow mouse pointer

```js
const a = gsap.quickSetter(eyeContainer1, 'rotation', 'deg');
const i = gsap.quickSetter(eyeContainer2, 'rotation', 'deg');
window.addEventListener('mousemove', (e) => {
  const angle = Math.atan2(e.clientY - vh/2, e.clientX - vw/2) * 180 / Math.PI - 180;
  a(angle); i(angle);
});
```

Eye container 是 `<div style="position:absolute; transform: translate(-50%,-50%) rotate(0deg); transform-origin: 50% 50%">`，里面装黑色 pupil。旋转容器让 pupil 在眼眶里转圈。

### 8.7 Footer juggling balls (4 ingredients)

```js
const cfg = {restVH:22, apex:[38,60], driftStart:[2,7], driftEnd:[8,24],
  upDur:[.9,1.3], gravityRatio:[1.15,1.5], spin:[220,600], repeatDelay:[.2,.9], staggerStart:.55};
function throwBall(el) {
  // random apex/drift/duration; timeline of: opacity↑ → y:-apex (power2.out) → y:restVH (power2.in) →
  // x:driftEnd (sine.inOut) → rotation:spin (linear) → opacity↓ → onComplete: delayedCall(repeatDelay, () => throwBall(el))
}
```

Mobile cfg 更柔和（apex [12,18], spin [80,180]）。

### 8.8 Smooth scroll (Lenis-like)

Next.js 站常用 `@studio-freight/lenis`。Shopify 的 Lumia 母版会跟 Lenis wheel listener 冲突（[BUGS.md P3-05](BUGS.md#p3-05)）。**直接手写 wheel handler ~40 行**，参考 `crav-app.js initSmoothScroll`。

---

## 9. 反向 cascade self-sabotage 检查清单

写完 `{prefix}-base.css` 后，**自己审一遍**：

1. ❓ 是否在 `body.{prefix}-index` 上加了 `color`、`font-family`、`font-size`、`line-height` 的 `!important`？→ **删掉**（[P4-02](BUGS.md#p4-02)）
2. ❓ 是否有 `.{prefix}-page h1, .{prefix}-page h2` 之类 wrapper-prefix 给通用元素加规则？→ **降级** 或针对具体 Lumia selector override（[P4-03](BUGS.md#p4-03)）
3. ❓ 是否 `.{prefix}-page` 上写了 inheritable property override？→ 改成 `.{prefix}-page.page-content { ... }` 中和母版的 wrapper inheritance（[P4-04](BUGS.md#p4-04)）
4. ❓ 是否用了 `font-size: 0` collapse-whitespace 技巧？→ **删**，改 HTML 层面（[P4-07](BUGS.md#p4-07)）
