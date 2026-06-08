---
name: mobile-screenshot
description: >
  将文本/Markdown内容渲染为适合手机浏览器浏览的高清PNG截图。
  流程：创建HTML文件 → 启动本地HTTP服务 → Playwright全屏截图 → 上传云端。
  当用户要求"生成截图"、"生成适合手机浏览的截图"、"文字不清晰重新调整"、"去掉底部留白"、"生成手机尺寸浏览效果"时使用。
  触发词：生成截图、手机截图、高清截图、文字清晰、去掉留白、手机浏览效果、截图上传云端。
---

# Mobile Screenshot — 手机高清截图

将文本内容渲染为 **828px 宽度（2倍视网膜分辨率）** 的高清 PNG 截图，自动匹配内容高度，上传云端并返回链接。

## 完整工作流

### 1. 创建 HTML 文件

将用户内容写入 `/home/admin/.openclaw/workspace/` 下的 HTML 文件。

**核心样式**：

```css
body {
  font-family: -apple-system, BlinkMacSystemFont, "PingFang SC", "Microsoft YaHei", sans-serif;
  font-size: 32px; line-height: 1.8; color: #24292e;
  padding: 32px 24px; max-width: 828px; margin: 0 auto; background: #fff;
}
h1 { font-size: 42px; border-bottom: 4px solid #333; padding-bottom: 14px; }
h2 { font-size: 34px; border-bottom: 3px solid #eaecef; padding-bottom: 10px; margin-top: 48px; }
h3 { font-size: 28px; margin-top: 36px; }
p { margin: 14px 0; font-size: 30px; }
table { width: 100%; border-collapse: collapse; margin: 18px 0; font-size: 26px; }
th, td { padding: 14px 16px; border: 2px solid #dfe2e5; }
th { background: #f6f8fa; font-weight: 700; }
blockquote { border-left: 6px solid #333; margin: 18px 0; padding: 18px 22px; background: #f6f8fa; font-size: 28px; line-height: 1.9; }
ul li { margin: 10px 0; font-size: 28px; }
```

### 2. 启动 HTTP 服务

```bash
fuser -k 8899/tcp 2>/dev/null
cd /home/admin/.openclaw/workspace
nohup python3 -m http.server 8899 --bind 127.0.0.1 > /dev/null 2>&1 & sleep 1
```

### 3. Playwright 截图

```bash
npx playwright screenshot --full-page --viewport-size=828,800 "http://127.0.0.1:8899/文件名.html" "/home/admin/.openclaw/workspace/文件名.png"
```

**必须**：`--full-page` 自动匹配内容高度，无底部留白。

### 4. 上传云端

```bash
python3 ~/.openclaw/skills/remote-oss-file/scripts/remote_oss_file.py \
  --base-url "https://openclaw-data-gateway.souche.com" \
  --file-path "/home/admin/.openclaw/workspace/文件名.png" \
  --file-name "显示名"
```

### 5. 返回链接

```
✅ 截图已生成并上传
 云端链接：https://res-fengche.souche.com/.../文件名.png
```

## 常见问题

| 问题 | 解决 |
|------|------|
| 文字不清晰 | font-size 32px→40px，line-height≥1.8 |
| 底部留白 | 必须用 `--full-page`，不设固定高度 |
| 缺浏览器 | `npx playwright install chromium` |
