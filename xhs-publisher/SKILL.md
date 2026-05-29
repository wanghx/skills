---
name: xhs-publisher
description: 小红书创作者平台自动化发布 Skill。通过浏览器自动化在 creator.xiaohongshu.com 完成手机验证码登录、图文内容上传与发布。应在用户需要自动发布小红书图文笔记时使用，触发词包括发布小红书、小红书发帖、小红书自动发布、xhs publish 等。
---

# 小红书创作者平台自动化发布

## 概述

通过 Playwright 浏览器自动化，在创作者平台 (creator.xiaohongshu.com) 完成：
登录（会话复用） → 进入图文发布页 → 上传图片 → 填写标题/正文 → 点击发布。

## 工具

本 Skill 自带 `playwright-cli` 命令速查，无需额外加载浏览器自动化 Skill。所有用到的命令见 `references/playwright-cli.md`。

> **登录方式说明**：创作者平台仅支持短信验证码登录，无账号密码入口。通过会话持久化，验证码只需输入一次，后续自动复用。

## 工作流

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
