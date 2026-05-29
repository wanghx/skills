# playwright-cli 命令参考（精简版）

本文件包含 xhs-publisher 工作流所需的所有 playwright-cli 命令，无需额外加载 playwright-cli Skill。

## 环境检查与安装

```bash
# 检查是否已安装
which playwright-cli || npm install -g @playwright/cli@latest
```

## 快照注意事项

`snapshot` 默认输出到 `.playwright-cli/` 目录文件，**必须用 cat 读取**：

```bash
playwright-cli snapshot --filename=snap.yaml
cat .playwright-cli/snap.yaml
```

## 核心命令

```bash
# ── 浏览器生命周期 ──
playwright-cli open                             # 打开浏览器
playwright-cli open https://example.com         # 打开并导航
playwright-cli list                             # 列出所有会话
playwright-cli close                            # 关闭当前会话

# ── 页面导航 ──
playwright-cli goto https://example.com/page

# ── 页面交互 ──
playwright-cli snapshot --filename=snap.yaml   # 获取元素引用
playwright-cli click e3                         # 点击元素
playwright-cli fill e5 "内容"                   # 填充输入框
playwright-cli type "文本"                       # 键盘输入

# ── JavaScript 执行 ──
playwright-cli eval "document.title"            # 执行 JS 表达式

# ── 截图调试 ──
playwright-cli screenshot --filename=debug.png

# ── 会话持久化（关键） ──
playwright-cli state-save xhs-auth.json         # 保存登录状态
playwright-cli state-load xhs-auth.json         # 恢复登录状态
```

## 典型使用流程

```bash
# 1. 加载会话
playwright-cli state-load xhs-auth.json

# 2. 导航
playwright-cli goto https://creator.xiaohongshu.com/publish/publish?from=menu&target=image

# 3. 检查是否需重新登录（快照看 URL）
playwright-cli snapshot --filename=check.yaml
cat .playwright-cli/check.yaml

# 4. 如果重定向到 login，走 SMS 登录；否则直接操作页面元素

# 5. 操作完成后关闭
playwright-cli close
```
