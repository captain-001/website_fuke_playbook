# 05 — 每个 Feature 必走的验证清单

每完成一个 section / 交互后，**按顺序走完下面 7 步**才算 done。漏一步几乎必有回归。

## 1. 静态首屏（F5 后初始态）

- [ ] 颜色对：文字 / 背景 / 边框，特别是 dark section 里的文字（容易被 Dawn 灰色 cascade 污染，参见 [02-css-cascade.md §1](02-css-cascade.md)）
- [ ] 位置对：用浏览器尺子量像素，跟原站对照
- [ ] 字号 / 字体 / 行高对：DevTools Computed 看具体 px 值
- [ ] 没有 horizontal scrollbar（动画初始的 translateX 偏移别露出来）
- [ ] 用 600px 视口和 1600px 视口各看一次

## 2. 滚动行为

- [ ] 从顶部慢慢滚到底部，每个滚动驱动动画的入场都自然
- [ ] 每个动画的**初始 progress=0** 时元素状态正确（不会闪现"完成态"再被 JS 推回去）
- [ ] sticky 元素到位置正常 pin、过位置正常 unpin
- [ ] 200svh sticky 区段的 `progress` 在 pin 期间正确从 0 走到 1
- [ ] **滚到页面底**：浮动 nav 隐藏 + 菜单自动打开
- [ ] **从底往上滚**：菜单自动关、浮动 nav 出现

## 3. Hover 交互

- [ ] 每个 hover 状态触发一次，视觉对
- [ ] 鼠标移开再移回，**动画能重新触发**（这是 transition vs animation 选错的最常见症状，参见 [03-animations.md §2](03-animations.md)）
- [ ] cursor 跟随 + label 切换正常
- [ ] 没有 hover "粘住"（mouseleave 没正确清状态）

## 4. 点击 / 多次触发

- [ ] 按钮 / 链接的 click 真的导航
- [ ] Carousel / toggle 快速点 3-5 次，动画稳定**每次都重放**（reviews 的 line-reveal 重触发是重灾区）
- [ ] 菜单打开 / 关闭快速切换不闪烁
- [ ] ESC 键能关菜单

## 5. 响应式

- [ ] 600px 临界点两侧各看一次（599px / 601px）
- [ ] 320px 极窄视口不破版
- [ ] 1920px 宽屏不留过多空白 / 元素不超 viewport
- [ ] 触屏设备 cursor 系统不显示（`(hover: hover)` 媒体查询有效）

## 6. Dawn cascade 检查

打开 DevTools → Elements → 选一个关键元素 → **Computed 面板**：

- [ ] `background`, `color`, `padding`, `margin`, `font-family`, `border` 最终值是预期的
- [ ] 如果不对，点开属性看是哪条规则赢的
- [ ] Dawn 的 `.button` / `.gradient` / `.media` / `.field` 等全局规则**有没有偷偷生效**

## 7. 回归

改了 A 之后**至少抽查相邻 2 个 section**，确认没影响：

- [ ] 改了 `fg-base.css` → 全站抽查（hero / showroom / footer）
- [ ] 改了 `fg-scroll-progress.js` → 所有带 `data-fg-scroll` 的 section 都测一遍
- [ ] 改了 cascade 影响范围（color / 全局 class）→ 走一遍主页所有 section

---

## 工程化建议

写 section 时按这个顺序：

1. **先做静态版**（不带任何动画）—— 验证布局、字号、颜色
2. **加 hover 交互** —— 验证 cursor / 按钮 hover
3. **加滚动驱动** —— 验证 `--fg-progress` + CSS calc
4. **加入场动画** —— `data-fg-reveal-lines` + 配套
5. **每步推一次 preview，让用户看一次**

不要一次写完所有动画再推送测试，**bug 难定位**。Phase 2 早期就是因为想一次写完才反复返工。

---

## "看上去对了" ≠ "真的对了" 的常见情况

| 表面 | 实际可能 |
|---|---|
| 动画首次播放正常 | 不能重放（transition 选错） |
| 局部测试白色文字正常 | 全局 cascade 在生产被推翻（specificity 不够） |
| 桌面 1600px 测正常 | 1920/1440 视口元素偏移（用了 fixed px 而非 fg-font-s） |
| editor 里没问题 | pure preview URL 上 JS 失效（IO 时机不一样） |
| 第一个 carousel slide 动画正常 | 切换后续 slide 没重放（line-reveal 没重触发） |
| 桌面 hover 正常 | 移动端被 hover 状态卡死（没用 `@media (hover: hover)`） |

**结论**：每个 feature 都按上面 7 步走，能省下大量返工。
