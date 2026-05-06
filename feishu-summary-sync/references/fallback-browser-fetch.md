# 浏览器会话回退抓取流程

当通过命令行方式获取 URL 正文失败时，不要反复硬撞同一种抓取方式。常见失败表现包括：

- `403 Forbidden`
- `Cloudflare challenge`
- `Enable JavaScript and cookies to continue`
- 页面标题可见，但正文不可见
- 抓到的是整页样式、脚本或挑战页，而不是正文

## 适用场景

- 普通 `Invoke-WebRequest` / `web.open` 只能拿到 `403`
- 页面标题可见，但正文不可见
- 需要复用本机已登录浏览器会话抓正文

## 已验证可用浏览器方案

- **优先使用 `Chrome Beta`**
- Windows 已验证路径示例：
  - `C:\Program Files\Google\Chrome Beta\Application\chrome.exe`
- Linux 常见可执行名示例：
  - `google-chrome-beta`

原因：
- `chrome-devtools-mcp` 的 `--channel=beta` 配置会直接找 `chrome-beta`
- 在当前环境里，普通 `Chrome` + `browserUrl` 可单独走通 CDP，但和现有 `chrome-devtools` MCP 的默认加载逻辑不完全一致
- 为了减少分叉，遇到“浏览器会话抓正文”需求时优先统一到 `Chrome Beta`
- 若目标环境没有 `Chrome Beta`，应明确标注“当前仅验证普通 Chrome / 未验证 Linux beta 通道”，不要把未验证路径写成既定事实

## MCP 配置

如果当前客户端读取的是 `~/.codex/config.toml`，推荐配置为：

```toml
[mcp_servers.chrome-devtools]
command = "npx"
args = ["chrome-devtools-mcp@latest", "--autoConnect", "--channel=beta"]
```

## 处理步骤

1. 安装 `Chrome Beta`
2. 完整重启 `Codex` / `AionUi`
3. 确认 `chrome-devtools` MCP 已加载
4. 让 MCP 自动连接 `Chrome Beta`
5. 用浏览器会话访问目标页，优先提取：
   - 页面标题
   - 首帖正文 `article .cooked` / `.cooked`
6. 如果正文抓到后，再按正常流程生成本地 `Markdown`、飞书 `docx` 和年度索引记录

## 关于重启生效

- 仅修改 `~/.codex/config.toml` 后，当前会话**不会自动热加载**新的 `mcp_servers`
- 必须 **完整退出并重新打开** `Codex` / `AionUi`
- 如果重启后 `chrome-devtools` 仍不可见，优先检查：
  - 配置是否写入了正确文件
  - `Chrome Beta` 是否已安装
  - `--channel=beta` 是否仍在配置中

## 避免重复踩坑

- **不要**一上来就只靠 `Invoke-WebRequest` 或普通 `web.open`
- **不要**把某个具体站点是否可访问，误判成命令行抓取链路本身一定可行
- **不要**把 `Edge` 当成默认浏览器路径
- **不要**优先走 cookie 解密、影子复制这类重方案
- **不要**把“页面标题能打开”误当作“正文已可读”

## 可接受的中间验证信号

- 页面标题从挑战页变成真实帖标题
- 正文选择器如 `.cooked` / `article .cooked` 能拿到非空文本
- 文本里出现首帖正文，而不是整页 CSS/脚本/挑战页提示

## 正文提取选择器优先级

抓浏览器正文时，优先按以下顺序尝试：

1. `.topic-post .regular.contents`
2. `.post-stream .topic-post:first-child .regular.contents`
3. `article .cooked`
4. `.cooked`

不要直接把 `document.body.innerText` 当成最终正文，因为容易混入大量样式、导航和脚本文字。

## 失败边界

如果在以下条件下仍无法拿到正文：

- `chrome-devtools` MCP 未加载
- `Chrome Beta` 不存在或未被自动连接
- 页面仍被 challenge 拦截

则必须明确标记：

- 链接无法稳定获取正文
- 仅记录访问受限状态
- 请求用户提供页面正文、截图、导出 HTML 或可访问副本
