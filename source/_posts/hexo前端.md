---
title: Hexo 自定义主题
date: 2026-06-05
tags: [Hexo, 前端, EJS, 主题]
categories: 前端
---

## 一、EJS 模板引擎

EJS（Embedded JavaScript）是 Hexo 使用的模板引擎，负责将数据动态渲染为 HTML。

### 1.1 基础语法

| 语法 | 作用 | 示例 |
|------|------|------|
| `<%= var %>` | 输出变量（转义 HTML，防 XSS） | `<%= config.title %>` |
| `<%- var %>` | 输出变量（不转义，渲染 HTML） | `<%- page.content %>` |
| `<% code %>` | 执行 JS 逻辑，不输出 | `<% if (page.prev) { %>` |
| `<%# 注释 %>` | 模板注释，不输出到 HTML | — |

### 1.2 循环与条件

```ejs
<!-- 遍历文章列表 -->
<% page.posts.each(function(post){ %>
  <a href="<%= url_for(post.path) %>"><%= post.title %></a>
<% }); %>

<!-- 条件判断 -->
<% if (page.tags && page.tags.length) { %>
  <div class="tags">...</div>
<% } %>
```

### 1.3 Hexo 内置模板 API

| 方法/变量 | 说明 | 使用场景 |
|-----------|------|----------|
| `config.title` | 站点标题 | 导航栏、`<title>` |
| `config.author` | 站点作者 | 页脚版权 |
| `url_for(path)` | 生成站点内绝对路径 | 所有链接和资源引用 |
| `date(obj, format)` | 格式化日期 | 文章发布日期 |
| `page.posts` | 当前页文章列表 | 首页遍历 |
| `page.content` | 文章 HTML 正文 | 文章详情页 |
| `page.title` | 文章标题 | 详情页标题 |
| `page.tags` | 文章标签集合 | 标签展示 |
| `page.categories` | 文章分类集合 | 分类展示 |
| `page.prev` / `page.next` | 上/下分页对象 | 分页导航 |
| `page.total` | 总页数 | 分页显示 "1/5" |
| `page.current` | 当前页码 | 分页显示 |
| `post.path` | 文章访问路径 | 文章链接 |

---

## 二、CSS 自定义属性（CSS Variables）

**核心价值**：一处定义，全局复用，方便换肤。

```css
:root {
  --bg: #0d1117;           /* 主背景 */
  --card-bg: #161b22;      /* 卡片背景 */
  --primary: #58a6ff;      /* 主色调（蓝） */
  --text: #c9d1d9;         /* 正文颜色 */
  --text-secondary: #8b949e; /* 次要文字 */
  --border: #30363d;       /* 边框色 */
  --hover: #1f6feb;        /* 悬停色 */
}

body {
  background: var(--bg);
  color: var(--text);
}
```

**知识点**：
- `:root` 选择器优先级高于 `html`，是定义全局变量的标准位置
- 变量名区分大小写，通常用 `--` 前缀（CSS 规范）
- 可以通过 JS `setProperty('--bg', '#fff')` 运行时换肤

---

## 三、Flexbox 弹性布局

本主题大量使用 Flexbox，是页面布局的核心技术。

### 3.1 两端对齐（导航栏）

```css
.site-header .container {
  display: flex;
  justify-content: space-between;  /* 左右两端 */
  align-items: center;             /* 垂直居中 */
}
```

### 3.2 横向排列（文章列表行）

```css
.post-item {
  display: flex;
  align-items: center;   /* 日期与标题垂直居中在同一行 */
}
```

### 3.3 换行布局（标签云）

```css
.post-tags {
  display: flex;
  flex-wrap: wrap;       /* 超出宽度自动折行 */
  gap: 8px;              /* 标签之间统一间距 */
}
```

### Flexbox 核心属性速查

| 属性 | 作用 | 常用值 |
|------|------|--------|
| `display: flex` | 开启弹性容器 | flex |
| `justify-content` | **主轴**对齐（水平） | center / space-between / flex-start |
| `align-items` | **交叉轴**对齐（垂直） | center / flex-start / stretch |
| `flex-direction` | 主轴方向 | row（默认）/ column |
| `flex-wrap` | 是否换行 | nowrap / wrap |
| `gap` | 子元素间距 | 8px, 16px, 1rem |
| `flex-shrink` | 是否允许收缩 | 0（不允许）/ 1（允许） |

---

## 四、CSS 定位（Position）

### 4.1 Sticky 粘性定位（导航栏）

```css
.site-header {
  position: sticky;
  top: 0;
  z-index: 100;
}
```

**关键理解**：`sticky` 在元素滚动到 `top: 0` 时"粘"在视口顶部，不脱离文档流。与 `fixed` 的区别：

| 值 | 参照物 | 脱离文档流？ | 占位？ |
|----|--------|:----------:|:-----:|
| `static` | 默认 | — | — |
| `relative` | 自身原位置 | 否 | 保留 |
| `absolute` | 最近定位祖先 | **是** | 不占 |
| `fixed` | 视口 | **是** | 不占 |
| `sticky` | 滚动容器 | **否** | 保留 |

> 固定导航栏为什么不推荐用 `fixed`？因为 `fixed` 脱离文档流后，下方内容会"跳"上去，需要用 `padding-top` 或 `margin-top` 手动补偿，而 `sticky` 自动保留占位。

---

## 五、CSS 过渡动画（Transition）

```css
a {
  color: var(--primary);
  transition: color 0.2s;     /* 颜色变化 0.2 秒平滑过渡 */
}
a:hover {
  color: var(--hover);
}
```

**语法**：`transition: 属性 时长 缓动函数 延迟;`

| 缓动函数 | 效果 |
|----------|------|
| `ease` | 默认，慢 → 快 → 慢 |
| `ease-in-out` | 慢 → 快 → 慢（更平滑） |
| `linear` | 匀速 |
| `ease-in` | 慢 → 快 |

> **注意**：`transition` 写在**元素默认状态**上，而不是 `:hover` 上，这样鼠标进入和离开都有过渡效果。

---

## 六、CSS 伪类（Pseudo-classes）

| 伪类 | 含义 | 本主题用途 |
|------|------|------------|
| `:hover` | 鼠标悬停 | 链接变色、列表行高亮 |
| `:last-child` | 父元素最后一个子元素 | 去掉列表最后一项边框 |
| `:first-child` | 父元素第一个子元素 | 选中第一项特殊处理 |

```css
/* 最后一行不显示分割线，避免重复边框 */
.post-item:last-child {
  border-bottom: none;
}

/* 悬停时浅蓝色高亮 */
.post-item:hover {
  background: rgba(88, 166, 255, 0.05);
}
```

---

## 七、文本溢出省略（Text Overflow）

实现"标题太长时显示 `...`"的经典三板斧：

```css
.post-title-link {
  white-space: nowrap;        /* ① 强制不换行 */
  overflow: hidden;           /* ② 超出部分隐藏 */
  text-overflow: ellipsis;    /* ③ 超出部分显示省略号 */
}
```

> **三个属性缺一不可**。如果想多行省略（如 3 行），需要改用 `-webkit-line-clamp`。

---

## 八、响应式设计（Media Queries）

```css
/* 屏幕宽度 ≤ 640px 时生效 */
@media (max-width: 640px) {
  .post-item {
    flex-direction: column;    /* 日期和标题上下排列 */
  }
  .post-full {
    padding: 24px 20px;        /* 减小内边距，利用更多宽度 */
  }
  .post-title {
    font-size: 1.5rem;         /* 缩小标题，避免换行过多 */
  }
}
```

**常见断点参考**：

| 断点 | 适配设备 |
|------|----------|
| `480px` | 小屏手机 |
| `640px` | 大屏手机 / 小平板 |
| `768px` | 平板竖屏 |
| `1024px` | 平板横屏 / 小笔记本 |
| `1280px+` | 桌面端 |

**移动优先**策略：默认样式写移动端，`@media (min-width: xxx)` 逐步增强桌面端体验。

---

## 九、字体与排版

### 9.1 系统字体栈（无需网络加载）

```css
body {
  font-family:
    -apple-system, BlinkMacSystemFont,  /* macOS / iOS */
    "Segoe UI",                          /* Windows */
    "PingFang SC", "Microsoft YaHei",   /* 中文 */
    sans-serif;                          /* 兜底 */
}
```

**原则**：优先使用操作系统自带字体，零网络请求，渲染最快。

### 9.2 等宽字体（代码）

```css
code {
  font-family: "SF Mono", "Fira Code", monospace;
}
```

### 9.3 行高与可读性

```css
body {
  line-height: 1.6;    /* 正文黄金区间 1.5 ~ 1.8 */
}
.post-content {
  line-height: 1.8;    /* 长文阅读建议 1.7 ~ 2.0 */
}
```

> 行高用无单位数值（如 `1.6`），会根据 `font-size` 自动计算，比固定 `px` 更灵活。

---

## 十、暗色主题设计要点

### 10.1 不用纯黑背景

```css
/* ❌ 避免 */
body { background: #000; }
/* ✅ 推荐：深灰护眼 */
body { background: #0d1117; }
```

纯黑（`#000`）对比度太强，长时间阅读导致视觉疲劳。深灰（如 `#0d1117` = GitHub Dark 背景）更舒适。

### 10.2 文字层次

| 用途 | 颜色 | 对比度 | 视觉效果 |
|------|------|:------:|----------|
| 标题 | `#ffffff` / `#c9d1d9` | 高 | 醒目 |
| 正文 | `#c9d1d9` | 约 12:1 | 清晰不刺眼 |
| 次要文字 | `#8b949e` | 约 6:1 | 辅助信息 |
| 禁用/提示 | `#484f58` | 低 | 不打扰 |

### 10.3 暗色下的蓝色

在暗色背景下，蓝色系（`#58a6ff`、`#79c0ff`）视觉温暖、识别度高，是链接和强调色的最佳选择。**避免**使用高饱和度纯蓝（`#0000ff`），暗背景下会很刺眼。

---

## 十一、其他 CSS 知识

### 11.1 Box Sizing

```css
* {
  box-sizing: border-box;
}
```

将 `padding` 和 `border` 计入元素总宽高，避免 `width: 100%` + `padding` 导致溢出。现代前端开发的标准操作。

### 11.2 Backdrop Filter（毛玻璃）

```css
.site-header {
  backdrop-filter: blur(10px);  /* 背景高斯模糊 */
}
```

实现导航栏对下方内容的毛玻璃效果，增强层次感。需要浏览器支持。

### 11.3 SVG 内联图标

```html
<svg width="16" height="16" viewBox="0 0 16 16" fill="currentColor">
  <path d="M3.5 0a.5.5 0 01.5.5V1h8V.5a.5.5 0 011 0V1h1..."/>
</svg>
```

**`fill="currentColor"`** 是关键——让 SVG 颜色继承父元素的 `color`，不需要单独指定颜色，跟随文字颜色自动适配。

### 11.4 溢出滚动（代码块）

```css
.post-content pre {
  overflow-x: auto;   /* 代码太长出现水平滚动条，不破坏布局 */
}
```

---

## 十二、知识体系总图

```
Hexo 自定义主题
│
├── 模板层（EJS）
│   ├── 变量输出 <%= %> vs <%- %>
│   ├── 流程控制（循环、条件）
│   └── Hexo 模板 API（url_for, date, page.*）
│
├── 布局层（CSS）
│   ├── Flexbox（导航栏、列表行、标签流式排列）
│   ├── Position（sticky 导航固定）
│   └── 响应式 Media Queries（移动端适配）
│
├── 样式层（CSS）
│   ├── 自定义属性（主题色统一管理）
│   ├── 过渡动画（hover 交互反馈）
│   ├── 伪类（:hover, :last-child）
│   ├── 文本溢出省略（单行 ...）
│   └── 字体排版（系统字体栈、行高比例）
│
├── 暗色设计（Design）
│   ├── 配色方案（深蓝黑调 #0d1117）
│   ├── 文字层次（对比度区分主次）
│   └── 蓝色强调（#58a6ff 暖蓝在暗色下舒适）
│
└── 其他技巧
    ├── box-sizing: border-box（盒模型标准化）
    ├── backdrop-filter（毛玻璃效果）
    ├── SVG currentColor（图标颜色跟随文字）
    └── overflow-x: auto（代码块滚动）
```
