# 09 — Playwright 自验方法论

CRAV 项目沉淀，详细背景见 [BUGS.md P4-10](BUGS.md#p4-10)。**每次 `shopify theme push` 完成后，强制跑 Playwright probe + screenshot，再向用户报告**。

---

## 1. 为什么必须自验

之前 Shopify 部署流程里，"push 成功" = 报告完成。但 push 成功只意味着文件上传到 admin，**不等于渲染正确**。明显的渲染错误（贴图黑框、字号、字体、动效缺失）只有打开 preview URL 才能看到。

不自验的后果：用户截图反复反馈 → 多轮迭代 → 信任损耗。CRAV 项目就经历了一次"4 个明显错误用户截图后才发现"的事故，被用户原话："**做完自己 playwright 确认位置，截图的都是明显不对的，修复完再自查**"。

---

## 2. 环境准备（每个项目一次）

```bash
cd d:/shopify_try/{project}/
npm i -D playwright
npx playwright install chromium
```

---

## 3. 基础探针模板

```js
// d:/shopify_try/{project}-probe.mjs
import { chromium } from 'playwright';
import fs from 'node:fs';

const URL = 'https://himaxlimited.myshopify.com?preview_theme_id={ID}';
const b = await chromium.launch({ headless: true });
const ctx = await b.newContext({ viewport: { width: 1440, height: 900 } });
const p = await ctx.newPage();

const errs = [];
p.on('pageerror', (e) => errs.push(e.message));
p.on('console', (m) => { if (m.type() === 'error') errs.push(m.text()); });

await p.goto(URL, { waitUntil: 'domcontentloaded', timeout: 60000 });
await p.waitForTimeout(5500);  // wait for loader exit + initial animations

// Force-hide loader if still up
await p.evaluate(() => {
  const l = document.querySelector('div[role="dialog"][aria-label="Page loading"]');
  if (l) l.style.display = 'none';
});
await p.waitForTimeout(800);

// ... probe + screenshot ...

console.log('errors:', errs.slice(0, 10));
await b.close();
```

---

## 4. 逐 section scroll + screenshot

ScrollTrigger 动画**只在元素进入视口后**才触发。Element screenshot 不会自动滚动。必须：

```js
const sections = [
  ['hero', '#hero'], ['about', '#about'], ['ingredients', '#ingredients'],
  ['map-desktop', '#map-desktop'], ['cta', '#cta'], ['footer', 'footer'],
];
for (const [name, sel] of sections) {
  const el = await p.$(sel);
  if (!el) { console.log('miss', name); continue; }
  await el.scrollIntoViewIfNeeded();
  await p.waitForTimeout(1500);  // animation play
  await el.screenshot({ path: `d:/shopify_try/probe-${name}.png` });
}
```

对于 React-rendered (非 SSR) sections（如 cravburgers 的 experience 红底块），用 querySelector 找文本特征：

```js
const handle = await p.evaluateHandle(() => {
  const sp = [...document.querySelectorAll('h2 [data-pop="true"]')]
    .find(e => e.textContent.toLowerCase().includes('food'));
  let el = sp; while (el && !el.classList.contains('bg-red')) el = el.parentElement;
  return el;
});
if (handle) await handle.screenshot({ path: 'd:/shopify_try/probe-exp.png' });
```

---

## 5. CDP 用 `CSS.getMatchedStylesForNode` 找赢的规则

DOM stylesheets API (`document.styleSheets`) 受 CORS 限制（Shopify CDN 跨域），跨域 stylesheet 的 `cssRules` 抛异常。改用 Chrome DevTools Protocol：

```js
const cdp = await p.context().newCDPSession(p);
await cdp.send('DOM.enable'); await cdp.send('CSS.enable');
const doc = await cdp.send('DOM.getDocument', { depth: -1 });
const f = await cdp.send('DOM.querySelector', { nodeId: doc.root.nodeId, selector: '#hero h1' });
const styles = await cdp.send('CSS.getMatchedStylesForNode', { nodeId: f.nodeId });

// 列出所有 set font-size 的规则及 specificity
const all = [...(styles.matchedCSSRules || [])];
for (const inh of styles.inherited || []) for (const mr of inh.matchedCSSRules || []) all.push(mr);
for (const mr of all) {
  const r = mr.rule;
  for (const p of r.style.cssProperties) {
    if (p.name === 'font-size') {
      const matched = (mr.matchingSelectors || []).map(i => r.selectorList.selectors[i].text).join(',');
      console.log(`[${matched}] ${p.value} ${p.important ? '!important' : ''}`);
    }
  }
}
```

输出按 cascade 顺序排列。**最后一条赢**（specificity + source order）。

---

## 6. 诊断顺序

`computed 跟预期不符` 时按这个顺序走：

1. **`getComputedStyle(el).XXX`** 看实际值 — 跟期望比对
2. **CDP `CSS.getMatchedStylesForNode`** 列所有 matched rules — 找最后赢的规则
3. **specificity 计算** — 哪条 selector spec 最高
4. **`@layer` 检查** — layered rules 会输给 unlayered，无视 specificity（详 [BUGS.md P4-01](BUGS.md#p4-01)）
5. **自家防御层审计** — 检查 base.css 是否反向 sabotage（[BUGS.md P4-02 / P4-03](BUGS.md#p4-02)）
6. **inherited 检查** — 父级 `color/font-family !important` 通过继承传染
7. **screenshot 对比** — 跟源站 preview 像素级对比

---

## 7. 关键 probe 模式

### 7.1 字体回退验证

```js
const info = await p.evaluate(() => {
  const el = document.querySelector('#hero h1');
  const cs = getComputedStyle(el);
  return {
    fontFamily: cs.fontFamily,  // 是否 "Modak"，不是 fallback "Modak Fallback" 或 serif
    fontSize: cs.fontSize,      // 30vw at 1440 = 432px
    color: cs.color,
  };
});
```

### 7.2 inline 零状态清除验证

```js
const stuckEls = await p.evaluate(() => {
  return [...document.querySelectorAll('[style*="opacity:0"], [style*="visibility:hidden"]')]
    .map(el => ({ tag: el.tagName, text: el.textContent.slice(0, 30) }));
});
// 应该为空数组；非空说明 clearZeroStates 漏了某些元素
```

### 7.3 ScrollTrigger 触发验证

逐段滚 + 等 + probe transform：

```js
await p.evaluate(() => document.querySelector('#cta').scrollIntoView());
await p.waitForTimeout(2000);
const ctaPops = await p.evaluate(() => 
  [...document.querySelectorAll('#cta [data-pop="true"]')].map(p => ({
    text: p.textContent.trim(),
    opacity: getComputedStyle(p).opacity,
    transform: getComputedStyle(p).transform,
  }))
);
// transform 应该是 matrix(1,0,0,1,0,0) — 即 scale 1，已动画完
// 如果仍是 matrix(0,0,0,0,0,0)，说明 trigger 没 fire
```

### 7.4 hover 动效验证

```js
const btn = p.locator('a[href="/menu"]').first();
await btn.scrollIntoViewIfNeeded();
await btn.screenshot({ path: 'before.png' });
await btn.hover();
await p.waitForTimeout(500);
await btn.screenshot({ path: 'after.png' });
// 对比两张图，看 hover 状态是否生效
```

### 7.5 跨页对比源站

```js
// 同时打开源站和 preview 测同一指标
for (const [tag, url] of [['orig', SRC_URL], ['ours', PREVIEW_URL]]) {
  await p.goto(url, ...);
  // probe & screenshot 后保存为 ${tag}-footer.png 等
}
```

像 CRAV footer CRAV 字母 width 对比，源站 1265.67 vs 我们 1620 → 一眼看到差 360px → 排查到 whitespace gap。

---

## 8. CDP 一次性诊断脚本

复用模板：

```js
// d:/shopify_try/diag.mjs — usage: node diag.mjs '#hero h1' font-size
import { chromium } from 'playwright';
const [sel, prop] = process.argv.slice(2);
const b = await chromium.launch({ headless: true });
const p = await b.newPage({ viewport: { width: 1440, height: 900 } });
const cdp = await p.context().newCDPSession(p);
await cdp.send('DOM.enable'); await cdp.send('CSS.enable');
await p.goto('https://himaxlimited.myshopify.com?preview_theme_id={ID}', { waitUntil: 'domcontentloaded' });
await p.waitForTimeout(5500);
const doc = await cdp.send('DOM.getDocument', { depth: -1 });
const found = await cdp.send('DOM.querySelector', { nodeId: doc.root.nodeId, selector: sel });
const styles = await cdp.send('CSS.getMatchedStylesForNode', { nodeId: found.nodeId });
const all = [...(styles.matchedCSSRules || [])];
for (const inh of styles.inherited || []) for (const mr of inh.matchedCSSRules || []) all.push(mr);
for (const mr of all) {
  const r = mr.rule;
  for (const pp of r.style.cssProperties) {
    if (pp.name === prop) {
      const m = (mr.matchingSelectors || []).map(i => r.selectorList.selectors[i].text).join(',');
      console.log(`${pp.important ? '!' : ' '} [${m}] ${pp.value}    /* ${r.selectorList.text.slice(0,60)} */`);
    }
  }
}
await b.close();
```

调用：`node diag.mjs '#hero h1' font-size`、`node diag.mjs 'footer h2' line-height`。

---

## 9. 必查 checklist（push 后）

每次 `shopify theme push --theme="..."` 完成后，**至少跑这 6 项**才向用户报告：

- [ ] 无 console errors / pageerrors（除已知第三方 app 噪声）
- [ ] 字体 family computed 是 Modak/Mouse Memoirs（不是 serif fallback）
- [ ] hero 字号 computed 跟源站对得上（误差 < 5%）
- [ ] 颜色 computed 跟源站对得上（red 是 #f91814，不是 #22292f）
- [ ] 关键 section screenshot 截下来跟源站像素级肉眼对比
- [ ] 关键 hover/scroll 动效手动触发并 screenshot 验证

任何一项不通过 → 不要报告完成，深挖根因。
