# 「选中即引用」功能设计稿

> 对应 Issue: https://github.com/clacky-ai/openclacky/issues/519
> 维护者回复：官方近期路线图无此功能，欢迎 PR。
> 本文所有结构描述基于上游 `openclacky-upstream` 当前代码实测，非推测。

---

## 1. 范围（做什么 / 不做什么）

**做**：Web UI 中，用户在 **assistant 或 user 消息**内选中一段文本，选区旁浮现「引用」按钮，点击后把选中文本以引用格式追加到 composer，用户可以紧接着写出自己的追问。四步操作（选中→复制→点回输入框→粘贴）压缩为两步。

**不做**（本次上游 PR 范围外）：
- 多选多处引用（Ctrl 多选区）
- 每处引用分别回复（多轮并行回复）
- 结构化引用存储（消息格式、数据库、AI 侧结构化提示词均不改）
- 引用深链 / 跳回原消息锚点
- 移动端原生选区菜单（text selection toolbar）定制

> 多选 + 分别回复是凯哥的生产级需求，留自建版；本次上游 PR 对齐行业基线（DeepSeek / Kimi / 通义同款交互）。

---

## 2. 上游架构事实底座（全部代码实测确认）

### 消息 DOM 结构（#messages 容器内）
| 类型 | DOM | 关键属性 |
|---|---|---|
| user | `div.msg-user-wrap > div.msg.msg-user` | innerHTML 为 escape 后的文本（+ 斜杠命令高亮），无 markdown 渲染 |
| assistant | `div.msg.msg-assistant` | `dataset.raw` 存原始 markdown 全文；`innerHTML = _renderMarkdown(...)`（marked + hljs + katex）；尾部有 copy 按钮 |
| info/error | `div.msg.msg-info` / `.msg-error` | 工具输出等 |

- 消息顺序号：DOM 中**没有**序号属性 → 需运行时数 `#messages` 下 `.msg-user-wrap, .msg-assistant` 的 index（1 起）。
- 历史消息懒加载（滚动到顶分页），DOM 会被追加/重绘 → 监听必须用**事件委托**，按钮每次动态创建销毁。
- 流式渲染：assistant 消息经 `assistant_message` WS 事件整条写入（`Sessions.appendMsg("assistant", ev.content)`）。流式期间 DOM 可能重建导致选区丢失 → 按钮自动消失，用户重选即可，不需要特殊处理。

### Composer
- `#user-input`（textarea），无 auto-grow JS（CSS `max-height:12.5rem; overflow-y:auto`）。
- 发送链：`_sendMessage()` 读 `input.value` → `WS.send({type:"message", content, lang})`。**content 是纯文本**。
- 用户气泡渲染：`SkillAC.renderUserMessageHtml(content)` — 只做 escape + `/command` 高亮，**不渲染 markdown** → 引用块发出去后，用户侧气泡显示为 `> xxx` 纯文本，AI 侧依然按 markdown 解析 blockquote。可接受；如需用户侧视觉引用块，属 P2 增强（本文第 9 节）。
- 草稿机制：切换会话时读/写 `_drafts`（画面级），与 input 事件无绑定 → 程序化插入无需手动派发 input 事件。

### 事件/样式/国际化现状
- `#messages` / 文档上**没有任何 selection 监听**（无冲突面，全新挂载点）。
- 设计 token 完备：`--color-accent-primary`、`--color-board-pr​​...`（语义色）、`--color-text-*`；有成熟浮层范例 `#skill-autocomplete`。
- i18n：仅 en / zh 两语言，`I18n.t("key")` + `data-i18n`。
- 前端无测试基建（spec/ 全为 Ruby 服务端），验证走 CDP headless Chrome E2E + `node --check`。

### 后端
**零改动**。引用最终形态是 composer 里的纯文本（markdown blockquote），走既有 `WS.send({type:"message", content})` 通道，存储/历史/渲染全复用现有管线。

---

## 3. 交互流程

```
用户在 assistant / user 消息内按住拖选文本（mouseup 后 60ms）
      │
      ├─ 选区为空 / 跨越消息边界 / 不在 #messages 内 ──→ 不显示按钮
      │
      ├─ 选区合法（锚点与焦点同属一条 .msg-user-wrap 或 .msg-assistant）
      │     ├─ 立即快照：{ 引文文本, 消息角色, 消息序号, 选区矩形 }
      │     ├─ 按钮出现在选区矩形顶部中点上方 10px（顶部放不下则选区下方）
      │     ├─ 任何 scroll / 点按也清除选区 → 按钮移除
      │
      └─ 点击「引用」
            ├─ 用快照生成引用块（不是读实时选区——点击瞬间选区已变）
            ├─ 追加到 composer：
            │     空输入框 → 直接放引用块
            │     非空且末尾无空白行 → 先补一个空行
            │     末尾追加，光标置于引用块后的空行
            ├─ composer 聚焦
            └─ 按钮移除，选区保留（用户可继续选第二处引用）
```

**连续引用**：第二次点击引用时若 composer 末尾已是引用块且其后无空行，自动先补空行，形成两个 blockquote 块。

**发送后**：引用块随 content 文本发入会话；AI 收到 markdown blockquote；用户气泡显示纯文本 `> ...`（P2 可选渲染增强）。

---

## 4. 引用格式方案

### 方案 B（推荐）：带出处锚点
```
> [助手 · 第3条]
> 选中的文本第一行
> 选中的文本第二行
>
> （用户续写的提问）
```
- 出处行语义：`角色 + 会话内序号`，role 用 i18n 文案（你/助手/You/Assistant）。消歧义场景：长会话里追问「你刚才说“xx”」时，模型不用回翻上下文猜测用户指哪条。
- 多段落引用：段间插入 `>`（空引用行），保持 blockquote 段落边界。
- 引用文本规范化：两端 trim，中间连续空行折叠为单个空行，行首不存在的 `>` 统一补上（防用户选中内容里已有 blockquote 冲突）。

### 方案 A（备选）：只贴文本
```
> 选中的文本

（用户续写）
```
无出处行，视觉更干净，但多轮会话中模型容易锚错对象。

**我的判断**：选 B。行业基线产品不做出处是因为它们主打轻聊天；Clacky 核心场景是长任务迭代（查资料/改代码/审报告），同一条消息被反复引用是高概率事件，出处行就是把歧义消灭在源头。格式仍为 markdown 原生语法，不引入任何私有协议——发到任何模型都不丢信息（模型天然懂 blockquote）。

---

## 5. 边界情况清单

| # | 场景 | 行为 |
|---|---|---|
| 1 | 选区为空（仅点击） | 不显示按钮 |
| 2 | 选区跨越两条消息 | 不显示按钮（出处无法唯一） |
| 3 | 选区在 thinking 块 / 代码块 / 表格内 | 允许引用（所见即所得，`selection.toString()` 取干净文本；代码块无行号污染——上游 hljs 未开 line-numbers） |
| 4 | 流式中 DOM 重建导致选区丢失 | 按钮自动消失，用户重选，无残留状态 |
| 5 | 选区含大量文本（全选长消息） | 不截断。composer 原生 `max-height:12.5rem` 滚动承载 |
| 6 | 按钮点击瞬间选区已被清除 | 用显示时快照生成引用块，与实时选区无关 |
| 7 | 消息内已有 copy 按钮等交互元素 | 引用按钮 z-index 独立浮层，不遮挡正文交互（copy 按钮在气泡内，quote 浮在选区上方） |
| 8 | 滚动页面 | scroll 捕获阶段立即移除按钮，不跟漂 |
| 9 | composer 已有草稿 | 换行后追加，不覆盖草稿；切换会话由既有 _drafts 机制保存 |
| 10 | 深色主题 / 自定义 brand 色 | 按钮底色用中性深色 + border token，不依赖 brand 色 |
| 11 | 触屏（长按拖选） | `touchend` 同 mouseup 处理，按钮出现即点即用 |
| 12 | WebSocket 未连接时点引用 | 正常插入 composer（引用只是文本），发送时由既有断线提示处理 |
| 13 | 空 assistant 消息 / 纯附件消息 | 无文本可选中，按钮根本不出现 |
| 14 | 编辑器模态（重命名/输prompt弹窗）下的 selection | 按钮只在选区属于 #messages 内 .msg 时出现，模态场景天然隔离 |

---

## 6. 前端伪代码（quote-select.js，新文件，自包含模块）

```
// lib/clacky/web/components/quote-select.js
const QuoteSelect = (() => {
  // ── 状态
  let _btn = null;              // 浮动按钮 DOM（全局唯一，动态创建销毁）
  let _snap = null;             // { text, role, idx, rect } 显示按钮时的选区快照

  // ── 工具 ──
  function _msgRoot(node) {     // 找最近的 .msg-user-wrap 或 .msg-assistant
    ...
  }
  function _msgRoleIndex(root) { // type: "user"|"assistant"；orders: index+1
    const items = document.querySelectorAll("#messages .msg-user-wrap, #messages .msg-assistant");
    return { role: ..., index: Array.prototype.indexOf.call(items, root) + 1 };
  }
  function _cleanText(s) {      // trim 两端；折叠连续空行；行首无 "> " 则补
    ...
  }
  function _buildBlock(snap) {  // 生成引用块字符串（方案 B）
    //   "[助手 · 第3条]\n"  → 每行 "> " 前缀 … 返回多行字符串
  }

  // ── 选区检查（事件委托，无绑定周期问题）──
  function _onMouseUp(e) {
    setTimeout(() => {          // 60ms：等浏览器完成选区更新
      const sel = window.getSelection();
      if (!sel || sel.isCollapsed) return _hide();
      const range = sel.getRangeAt(0);
      const r1 = _msgRoot(range.startContainer), r2 = _msgRoot(range.endContainer);
      if (!r1 || r1 !== r2) return _hide();          // 边界 #2：跨消息不引
      if (!r1.closest("#messages")) return _hide();  // 只在消息区
      _snap = {
        text: _cleanText(sel.toString()),
        ..._msgRoleIndex(r1),
        rect: range.getBoundingClientRect(),
      };
      if (!_snap.text) return _hide();
      _show(_snap.rect);
    }, 60);
  }

  // ── 按钮显示 / 定位 / 隐藏 ──
  function _show(rect) {
    if (!_btn) _btn = _createBtn();                  // 胶囊按钮 + quote svg + i18n 文案
    const btnRect = _btn.getBoundingClientRect();
    let top = rect.top - btnRect.height - 10;        // 选区上方 10px
    if (top < 8) top = rect.bottom + 10;             // 顶部出界 → 翻转选区下方
    _btn.style.top = top + "px";
    _btn.style.left = (rect.left + rect.width / 2) + "px";  // 水平居中，CSS translateX(-50%)
    _btn.classList.add("visible");
  }
  function _hide() { if (_btn) _btn.classList.remove("visible"); _snap = null; }

  // ── 点击引用（唯一动作入口）──
  function _onQuoteClick(e) {
    e.preventDefault();
    if (!_snap) return _hide();
    _insert(_buildBlock(_snap));
    _hide();
  }

  // ── 插入 composer ──
  function _insert(block) {
    const ta = document.getElementById("user-input");
    if (!ta) return;
    let v = ta.value;
    if (!v.endsWith("\n")) v += "\n";                 // 边界 #9：空行分隔
    v += block + "\n";                                // 块尾空行，光标落这里
    ta.value = v;
    ta.focus();
    ta.setSelectionRange(v.length, v.length);         // 光标置于引用块后
  }

  // ── 初始化（一次绑定，委托制）──
  function init() {
    // 时机：DOMContentLoaded 后 / 由 sessions.js 的 Sessions.init() 末尾调用
    document.addEventListener("mouseup", _onMouseUp);
    document.addEventListener("touchend", _onMouseUp);
    document.addEventListener("scroll", _hide, true); // 捕获阶段：任何滚动即隐藏（边界 #8）
    document.addEventListener("keydown", (e) => { if (e.key === "Escape") _hide(); });
  }

  return { init };
})();
```

### 挂载点改动（3 行级）

```
# lib/clacky/web/index.html（script 引入，加在 sessions.js 之后）
<script src="components/quote-select.js"></script>

# lib/clacky/web/sessions.js（Sessions.init() 末尾）
if (typeof QuoteSelect !== "undefined") QuoteSelect.init();

# lib/clacky/web/app.css（按钮样式，复用 token）
.quote-btn { position: fixed; z-index: 3000; ...  }

# lib/clacky/web/i18n.js（2 个 key）
en: chat.quote → "Quote";  zh: chat.quote → "引用"
（出处行角色词：en "You"/"Assistant"，zh "你"/"助手"）
```

---

## 7. 后端逻辑

**零改动**。等价伪代码描述（说明数据如何流经现有管线，无新增面）：

```
# 前端点击「引用」只改变 composer 文本 → 发送时：
WS.send({ type: "message", content: "> [助手 · 第3条]\n> 引用文本\n\n我的追问",
          session_id, lang, files })

# 后端（http_server / web_ui_controller 既有通道）：
message.content 落库为会话消息（纯文本）→ 进 agent 上下文（模型按 markdown 解析 blockquote）
→ assistant 回复经 assistant_message 广播 → 前端 _renderMarkdown 渲染 blockquote

# 历史回放：
GET /api/sessions/:id/messages → content 原样返回 → 重放时同一渲染管道
```

### 影响面审计（零协议变更）
- WS 消息 schema：不变（content 仍为 string）
- 存储：不变
- API：不变
- 唯一行为变化：**新增一个输入侧快捷路径**，把「复制粘贴」换成了「按钮追加」，最终产物与用户手工粘贴 `> 文本` 完全等价。

---

## 8. 用户侧气泡的显示效果（诚实声明）

用户消息气泡由 `SkillAC.renderUserMessageHtml` 做纯 escape → 引用块在**用户自己的气泡里**显示为带 `>` 的纯文本，不是渲染后的引用样式。选择：
- **v1 基线**：接受纯文本显示（后端与 AI 语义无损，改动最小）。
- **P2 增强**：给 renderUserMessageHtml 加「行首 `> ` → 引用高亮 span」的轻量处理，约 +20 行，视觉对齐。

我在 PR 中先做 v1 基线；若你要视觉引用块，P2 一并带上，不影响合入难度。

---

## 9. 验证方案

1. **JS 语法闸**：`node --check` 新文件 + 改动的 sessions.js
2. **CDP E2E**（localhost 起上游服务，headless Chrome 9222）：
   - 造一个含 user + assistant 消息的会话
   - 在 assistant 消息上程序化拖选文本 → 断言按钮出现于选区上方
   - 点击按钮 → 断言 composer 内容 == 期待引用块、光标在块尾
   - 边界断言：空选区不显示 / 跨消息选区不显示 / 滚动后按钮消失 / composer 末位补空行正确
   - 发消息回归：正常发送一条带引用的消息 → 气泡显示、WS 无异常
3. **.rb 回归**：全量 spec（服务端零改动，预期 0 失败，纯确认无副作用）
4. **四层 Review + 账本**：语义正确性（边界表逐条过）/ 协议路由影响面（第 7 节审计）/ 过程合规（语法+rubocop+spec）/ 唤醒效果声明（PR 描述写明交互变化）
5. PR 描述带「维护者欢迎 PR」上下文 + `Fixes #519` 链接；顺手附截图（引用按钮浮现 + 点击后 composer 状态）。

---

## 10. 需要凯哥拍板的决策点

1. **引用格式**：方案 B（带出处锚点，我推荐）还是方案 A（纯文本）？
2. **用户气泡显示**：v1 纯文本基线，还是连 P2 引用块样式一起做？
3. 按钮文案：「引用」/「Quote」跟随界面语言即可，无需拍板（默认如此）。

确认后我直接开工实现。