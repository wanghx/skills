# 小红书创作者平台发布 — 详细参考

## 账号信息

- 手机号：18268098750
- 昵称：汪红霞

## 登录方式限制

创作者平台仅支持**短信验证码登录**和**APP 扫一扫登录**，无密码登录入口。因此必须使用会话持久化方案避免重复验证。

## 会话持久化

### 首次登录并保存

```bash
# SMS 登录完成后
playwright-cli state-save .playwright-cli/xhs-auth.json
```

### 后续复用

```bash
# 每次发布前加载会话，跳过登录
playwright-cli state-load .playwright-cli/xhs-auth.json
playwright-cli goto https://creator.xiaohongshu.com/publish/publish?from=menu&target=image
```

### 会话过期检测

加载会话后，检查页面 URL：
- 若为 `/publish/publish` → 会话有效，直接进入发布流程
- 若为 `/login` → 会话过期，重新 SMS 登录并保存

### 注意事项

- 会话文件 `.playwright-cli/xhs-auth.json` 包含 cookies 和 localStorage
- 建议定期（每周）重新登录刷新会话，避免 cookie 过期
- 不要提交含敏感信息的会话文件到 Git

## 关键 URL

| 页面 | URL |
|------|-----|
| 登录页 | `https://creator.xiaohongshu.com/login` |
| 图文发布页 | `https://creator.xiaohongshu.com/publish/publish?from=menu&target=image` |
| 发布成功页 | `publish/success` |

## 发布 API

- **端点**: `POST https://edith.xiaohongshu.com/web_api/sns/v2/note`
- **响应**: 200 OK 表示发布成功

## 关键 DOM 元素

### 发布按钮 (xhs-publish-btn) ⚠️ 最关键的步骤

这是一个 Vue 自定义 Web Component，**没有 shadow DOM，innerHTML 为空，不在 page snapshot 中出现**。

#### 为什么普通点击无效

`<xhs-publish-btn>` 的发布逻辑绑定在 Vue 组件内部，不响应以下任何方式：
- ❌ `click()` 原生点击
- ❌ `dispatchEvent(new Event('click'))` 事件派发
- ❌ CDP 坐标点击 `Input.dispatchMouseEvent`
- ❌ Pinia store 搜索发布 action

#### 正确的定位与触发方法

```javascript
// 1. 找到按钮（通过标签名选择器，不在 snapshot 中）
const btn = document.querySelector('xhs-publish-btn');

// 2. 直接调用 Vue 内部方法
btn._onPublish();  // 触发发布
btn._onSave();     // 触发暂存离开
```

关键属性确认：
```
is-publish="true"       → 可发布状态
submit-text="发布"       → 按钮文本
is-save-draft="true"    → 支持暂存
save-text="暂存离开"     → 暂存按钮文本
```

按钮位置信息（示例）：
```
x=718, y=1350, width=864, height=90
```

#### 发布成功判断

发布后页面自动跳转到 `https://creator.xiaohongshu.com/publish/success`，检查 URL 即可确认。

### 发布页面内容区域

- 选择器：`.publish-page-content`
- 需要滚动到底部才能让 `<xhs-publish-btn>` 进入视野

### 标题输入框

- 页面内第一个 input 元素，placeholder 包含"标题"
- 最多 20 个字符

### 正文编辑器

- 标题下方的富文本编辑区域
- 支持直接输入文本和 emoji

### 图片上传区域

- 页面上的上传占位区域
- 触发 `<input type="file">` 选择文件

## 完整发布流程（playwright-cli 实操）

### 第一步：登录（会话复用优先）

```bash
# 打开浏览器并加载会话
playwright-cli open
playwright-cli state-load .playwright-cli/xhs-auth.json
playwright-cli goto 'https://creator.xiaohongshu.com/publish/publish?from=menu&target=image'

# 检查 URL：若停留在 /login 说明过期，走 SMS 登录
playwright-cli goto 'https://creator.xiaohongshu.com/login'
playwright-cli fill e53 "18268098750"          # 手机号
playwright-cli click e64                         # 同意协议
playwright-cli click e59                         # 发送验证码
# 等待用户提供验证码后：
playwright-cli fill e55 "验证码"
playwright-cli click e61                         # 登录
playwright-cli state-save .playwright-cli/xhs-auth.json
```

### 第二步：上传图片

```bash
# 获取快照找到上传区域 ref（通常为 e126 或类似）
playwright-cli snapshot --filename=snap.yaml
cat snap.yaml

# 拖拽图片到上传区域
playwright-cli drop e126 --path '/path/to/image.jpg'

# 等待上传完成
sleep 4
```

### 第三步：填写标题和正文

```bash
# 重新获取快照（上传后页面结构变化）
playwright-cli snapshot --filename=snap.yaml
cat snap.yaml

# 填写标题（placeholder 含"标题"）
playwright-cli fill e198 '你的标题'

# 填写正文（标题下方的 textbox）
playwright-cli fill e207 '正文内容，支持换行和 emoji'
```

### 第四步：发布 ⚠️ 关键

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

# 成功标志：页面跳转到 /publish/success
```

### 第五步：关闭浏览器

```bash
playwright-cli close
```
