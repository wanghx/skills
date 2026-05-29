---
name: xhs-publisher
description: 小红书创作者平台自动化发布 Skill。支持自动生成小红书风格文案 + 浏览器自动化发布。通过 playwright-cli 在 creator.xiaohongshu.com 完成手机验证码登录、图文内容上传与发布。应在用户需要自动发布小红书图文笔记时使用，触发词包括发布小红书、小红书发帖、小红书自动发布、xhs publish 等。
---

# 小红书创作者平台自动化发布

## 概述

本 Skill 提供两大能力：
1. **文案生成**：根据用户提供的关键信息，自动生成符合小红书平台风格的标题、正文和话题标签
2. **自动发布**：通过 Playwright 浏览器自动化完成登录 → 上传图片 → 填写内容 → 发布

## 工具

本 Skill 自带 `playwright-cli` 命令速查，无需额外加载浏览器自动化 Skill。所有用到的命令见 `references/playwright-cli.md`。

> **登录方式说明**：创作者平台仅支持短信验证码登录，无账号密码入口。通过会话持久化，验证码只需输入一次，后续自动复用。

---

## 文案生成

当用户提供关键信息（如车型、价格、里程、卖点等）但未给出完整文案时，自动生成符合小红书平台风格的标题、正文和话题标签。

### 小红书文案核心特点

| 特点 | 说明 |
|------|------|
| **口语化、亲切** | 用"家人们""姐妹们""亲测"等称呼，像朋友聊天 |
| **emoji 点缀** | 🚗💰✅📩👌✨ 等增强可读性，别每句都加 |
| **信息前置** | 价格、里程、车况等关键数据放前面 |
| **分段清晰** | 空行分隔信息块，长文必须分段 |
| **互动引导** | 结尾引导评论、私信、点赞 |
| **标签精准** | 3~7 个相关 hashtag，宁精勿滥 |

### 标题规范

- **最多 20 字**（小红书硬限制，超出会被截断）
- 用 emoji 开头吸引注意
- 突出最核心卖点
- 示例：`🚗6万！卡罗拉5万公里零事故`

### 正文结构模板

```
[打招呼 + 说明来意 — 1 行]
[空行]
[核心信息块 — 价格/参数/状态，每条一行，emoji 领衔]
[空行]
[情感化描述 — 车况/体验/感受，口语化]
[互动引导 — 引导私信或评论]
[空行]
[话题标签 — 3~7 个]
```

### 不同内容类型的完整模板

#### 二手车 / 闲置转让

```
家人们！帮朋友出一台{车型}🚗

💰 {价格表述，如：一口价：6万（诚心可小刀）}
🛣️ 里程：{里程}万公里
✅ 车况：{亮点，如：无事故！零出险！}

{省油/耐造/好开 等口语化评价}，{适用场景，如：代步练手都合适}～
车况真的嘎嘎好，开过的都说香👌
诚心要的直接私信我📩

#{主标签} #{场景标签} #{人群标签}
```

#### 好物分享 / 产品推荐

```
姐妹们！最近挖到宝了✨

{产品名称}
💰 {价格}
💡 {核心卖点/使用体验}

真的太{感受词}了，{推荐理由}
需要的姐妹评论区扣1～

#{产品标签} #{场景标签} #{品类标签}
```

#### 生活分享 / Vlog 文案

```
{一句话戳中情绪}

{事情经过 2~3 句}
{感受/感悟 1~2 句}

{互动：你们有没有类似的经历？}

#{情绪标签} #{场景标签} #{城市/地点标签}
```

### 话题标签策略

| 内容类型 | 推荐标签组合 |
|----------|------------|
| 二手车 | #车型名 + #二手车 + #代步车 + #练手车 + 人群标签（如 #女生代步车） |
| 好物分享 | #品类名 + #好物推荐 + #性价比 + #使用场景 |
| 生活分享 | #情绪词 + #场景 + #城市 + #日常 |
| 知识干货 | #领域 + #干货 + #新手 + #教程 |

### 生成策略

当用户只给了关键信息时，按以下步骤自动生成文案：

1. **识别内容类型**：二手车转让 → 匹配对应模板
2. **提取关键信息**：车型、价格、里程、卖点
3. **生成标题**：emoji + 核心数字 + 车型 + 最大卖点，控制在 20 字内
4. **填充正文**：按模板结构填入，保持口语化
5. **匹配标签**：根据内容类型和关键词自动生成 4~6 个标签
6. **展示给用户**：生成后先展示完整文案供用户确认，再进入发布流程

---

## 工作流（自动发布）

### 第一步：登录（会话复用优先）

#### 路径 A：加载已有会话（推荐）

如果之前成功登录过并保存了会话状态，直接加载跳过登录：

```bash
playwright-cli state-load .playwright-cli/xhs-auth.json
playwright-cli goto https://creator.xiaohongshu.com/publish/publish?from=menu&target=image
```

然后检查当前页面 URL：若停留在 `/login` 说明会话已过期，走路径 B；若成功进入发布页，直接跳到第三步。

#### 路径 B：短信验证码登录（首次或会话过期时）

1. 打开 `https://creator.xiaohongshu.com/login`
2. 输入手机号（默认：18268098750）
3. 点击「发送验证码」
4. 等待用户提供验证码
5. 输入验证码并点击登录
6. 等待页面跳转至创作者中心
7. **登录成功后立即保存会话**：

```bash
playwright-cli state-save .playwright-cli/xhs-auth.json
```

### 第二步：进入图文发布页

导航到：
```
https://creator.xiaohongshu.com/publish/publish?from=menu&target=image
```

### 第三步：上传图片

1. 点击上传区域触发 file chooser 对话框
2. 使用 file chooser 事件上传图片文件
3. 等待图片上传完成（观察上传进度消失）

> 如果页面有 "x-mark" 关闭弹窗按钮，需先关闭可能出现的引导弹窗。

### 第四步：填写内容

定位并填写以下字段：

| 字段 | 定位方式 | 说明 |
|------|----------|------|
| 标题 | 第一个 input/文本框，placeholder 含"标题" | 最多 20 字 |
| 正文 | 标题下方的富文本编辑器区域 | 支持 emoji |

### 第五步：点击发布 ⚠️ 最关键

#### 发布按钮是什么

`<xhs-publish-btn>` 是一个 **Vue 自定义 Web Component**，特点是：
- **没有 shadow DOM**，innerHTML 为空
- 不响应普通 DOM 事件（click、dispatchEvent 都无效）
- 不在 playwright-cli snapshot 中出现

#### 找到按钮

```javascript
document.querySelector('xhs-publish-btn')
```

按钮位于 `.publish-page-content` 容器底部，通常**不在可视区域内**，需要先滚动：

```javascript
const container = document.querySelector('.publish-page-content');
container.scrollTop = container.scrollHeight;
```

#### 触发发布

```javascript
const btn = document.querySelector('xhs-publish-btn');
btn._onPublish();  // 直接调用 Vue 内部方法
```

备用：`btn._onSave()` 用于「暂存离开」，`is-publish="true"` 表示可发布状态。

#### playwright-cli 实操

```bash
# 滚动到底部，让发布按钮可见
playwright-cli eval "
(() => {
  const container = document.querySelector('.publish-page-content');
  container.scrollTop = container.scrollHeight;
})()
"

# 调用 Vue 内部方法发布
playwright-cli eval "
(() => {
  const btn = document.querySelector('xhs-publish-btn');
  btn._onPublish();
})()
"
```

#### 发布 API

- **端点**：`POST https://edith.xiaohongshu.com/web_api/sns/v2/note`
- **成功标志**：页面跳转到 `publish/success`，API 返回 200

### 第六步：确认发布结果

- 观察页面是否跳转至 `publish/success`
- 检查网络请求 `web_api/sns/v2/note` 是否返回 200

## 常见问题排查

| 问题 | 原因 | 解决 |
|------|------|------|
| 找不到发布按钮 | `<xhs-publish-btn>` 不在可视区域 | 滚动 `.publish-page-content` 到底部 |
| 点击发布无反应 | Web Component 不响应 click 事件 | 使用 `_onPublish()` 调用 |
| 图片无法上传 | 文件对话框未正确处理 | 确保使用 `page.on('filechooser')` 处理 |
| 登录后未跳转 | 可能有弹窗遮挡 | 检查并关闭 x-mark 弹窗 |
| 会话加载后仍需登录 | Cookie 过期 | 走路径 B 重新 SMS 登录并保存 |
| 一直要求验证码 | 平台不支持密码登录 | 这是正常行为，用会话持久化绕过 |

## 参考

| 文件 | 内容 |
|------|------|
| `references/publishing-workflow.md` | DOM 结构、发布 API、会话持久化、完整脚本模板 |
| `references/playwright-cli.md` | playwright-cli 命令速查（环境检查、快照读取、核心命令） |
