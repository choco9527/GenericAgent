# Windows 家用电脑通过 Tailscale 启动 GenericAgent

这份文档用于下面这个场景：

- 办公室 Mac 可以访问 `http://47.236.3.167:8080`
- 家里 Windows 电脑不能直接访问这个地址
- 家里 Windows 电脑已经安装 Python 3
- 家里 Windows 电脑已经下载好 GenericAgent 文件夹
- 家里 Windows 电脑已经安装 Tailscale，并登录了和办公室 Mac 相同的账号

目标链路：

```text
家里 Windows GenericAgent
  -> http://127.0.0.1:8080
  -> SSH 隧道
  -> 办公室 Mac
  -> http://47.236.3.167:8080
```

这样只有 GenericAgent 访问本机 `127.0.0.1:8080` 的请求会走办公室 Mac，不会让家里电脑所有网络都走代理。

## 1. 办公室 Mac 前提检查

办公室 Mac 需要保持：

- 开机
- 联网
- Tailscale 已连接
- 能访问 `http://47.236.3.167:8080`
- 已开启 SSH 远程登录

办公室 Mac 当前 Tailscale 信息：

```text
设备名：chocos-mac
Tailscale IP：100.105.189.124
MagicDNS：chocos-mac.tail2e7a8d.ts.net
macOS 用户名：choco
```

在办公室 Mac 上开启 SSH：

```text
系统设置 -> 通用 -> 共享 -> 远程登录
```

确保允许用户 `choco` 远程登录。

## 2. Windows 检查 Tailscale 连通性

在家里 Windows 打开 PowerShell，测试是否能看到办公室 Mac：

```powershell
ping 100.105.189.124
```

或者：

```powershell
ping chocos-mac.tail2e7a8d.ts.net
```

如果 ping 不通：

- 确认办公室 Mac 的 Tailscale 是 Connected
- 确认 Windows 的 Tailscale 是 Connected
- 确认两台设备登录的是同一个 Tailscale 账号
- 确认 Tailscale 管理后台里两台设备都在线

## 3. Windows 检查 SSH 客户端

PowerShell 中运行：

```powershell
ssh -V
```

如果能看到 OpenSSH 版本，说明可以继续。

如果提示找不到 `ssh`，在 Windows 设置里安装：

```text
设置 -> 应用 -> 可选功能 -> 添加可选功能 -> OpenSSH Client
```

安装后重新打开 PowerShell。

## 4. 建立 SSH 隧道

在家里 Windows 的 PowerShell 中运行：

```powershell
ssh -N -L 8080:47.236.3.167:8080 choco@100.105.189.124
```

也可以使用 MagicDNS：

```powershell
ssh -N -L 8080:47.236.3.167:8080 choco@chocos-mac.tail2e7a8d.ts.net
```

第一次连接可能会提示：

```text
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

输入：

```text
yes
```

然后输入办公室 Mac 用户 `choco` 的登录密码。

注意：

- 输入密码时终端不会显示字符，这是正常的。
- 这个 PowerShell 窗口要一直开着。
- 关闭这个窗口，SSH 隧道就会断开。

## 5. 验证隧道是否成功

保持上一步的 SSH 窗口不要关闭。

再打开一个新的 PowerShell，运行：

```powershell
curl.exe http://127.0.0.1:8080
```

如果看到 FlatAI / API Gateway 相关 HTML 页面，说明隧道成功。

如果失败：

- `Connection refused`：SSH 隧道没有成功建立，或 8080 端口被占用
- `Connection timed out`：Windows 到办公室 Mac 的 Tailscale/SSH 不通
- 有 SSH 密码错误：确认办公室 Mac 的用户名和登录密码

如果家里 Windows 的 `8080` 端口被占用，可以改用 `18080`：

```powershell
ssh -N -L 18080:47.236.3.167:8080 choco@100.105.189.124
```

后续 GenericAgent 的 `ANTHROPIC_BASE_URL` 也要改成：

```text
http://127.0.0.1:18080
```

## 6. 安装 GenericAgent 依赖

进入家里 Windows 的 GenericAgent 文件夹，例如：

```powershell
cd C:\Users\你的用户名\Downloads\GenericAgent
```

安装最小依赖：

```powershell
py -3 -m pip install streamlit pywebview
```

如果你不用 `py` 启动器，也可以：

```powershell
python -m pip install streamlit pywebview
```

确认 Python 版本：

```powershell
py -3 --version
```

建议 Python 版本是 `3.11` 或 `3.12`。

## 7. 配置 GenericAgent API

推荐在家里 Windows 上创建：

```text
%USERPROFILE%\.claude\settings.json
```

PowerShell 可运行：

```powershell
mkdir $env:USERPROFILE\.claude -Force
notepad $env:USERPROFILE\.claude\settings.json
```

写入下面内容，把 `你的_API_TOKEN` 替换成真实 token：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:8080",
    "ANTHROPIC_AUTH_TOKEN": "你的_API_TOKEN",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",
    "CLAUDE_CODE_ATTRIBUTION_HEADER": "0"
  }
}
```

如果你第 5 步使用的是 `18080`，则改成：

```json
"ANTHROPIC_BASE_URL": "http://127.0.0.1:18080"
```

## 8. 创建 mykey.py

在 GenericAgent 项目根目录创建 `mykey.py`：

```powershell
notepad mykey.py
```

写入：

```python
import json
import os
from pathlib import Path


def _claude_env(name):
    value = os.environ.get(name)
    if value:
        return value

    settings_path = Path.home() / ".claude" / "settings.json"
    if not settings_path.exists():
        return ""

    try:
        settings = json.loads(settings_path.read_text(encoding="utf-8"))
    except (OSError, json.JSONDecodeError):
        return ""

    return str(settings.get("env", {}).get(name, "")).strip()


def _first_claude_env(*names):
    return next((value for value in (_claude_env(name) for name in names) if value), "")


native_claude_config = {
    "name": "claude-relay",
    "apikey": _first_claude_env("ANTHROPIC_AUTH_TOKEN", "ANTHROPIC_API_KEY"),
    "apibase": _claude_env("ANTHROPIC_BASE_URL"),
    "model": "claude-opus-4-7",
    "fake_cc_system_prompt": True,
    "thinking_type": "adaptive",
    "max_retries": 3,
    "read_timeout": 180,
}
```

不要把 `mykey.py` 提交到 Git。GenericAgent 默认 `.gitignore` 通常已经忽略 `mykey.py`。

## 9. 验证 API 配置

在 GenericAgent 项目根目录运行：

```powershell
py -3 -c "import mykey; cfg=mykey.native_claude_config; print(cfg['apibase']); print(bool(cfg['apikey'])); print(cfg['model'])"
```

预期输出类似：

```text
http://127.0.0.1:8080
True
claude-opus-4-7
```

如果第二行是 `False`，说明 token 没读到，检查：

```text
%USERPROFILE%\.claude\settings.json
```

## 10. 启动 GenericAgent 桌面版

保持 SSH 隧道窗口开着。

在 GenericAgent 项目根目录运行：

```powershell
py -3 launch.pyw
```

如果 `py` 不可用：

```powershell
python launch.pyw
```

启动成功后，会看到类似：

```text
URL: http://localhost:185xx
```

打开这个地址即可使用 GenericAgent。

## 11. 启动命令行版

如果只想用命令行版：

```powershell
py -3 agentmain.py
```

或：

```powershell
python agentmain.py
```

## 12. 安装浏览器能力扩展

如果要让 GenericAgent 控制 Chrome，打开 Chrome：

```text
chrome://extensions
```

然后：

1. 打开右上角“开发者模式”
2. 点击“加载已解压的扩展程序”
3. 选择 GenericAgent 里的目录：

```text
assets\tmwd_cdp_bridge
```

安装成功后应该看到：

```text
TMWD CDP Bridge
```

并且扩展处于启用状态。

## 13. 首次验证

在 GenericAgent 里先发：

```text
/status
```

然后测试普通模型请求：

```text
你好，请回复 pong
```

测试浏览器能力：

```text
打开百度，搜索"今天天气"
```

## 14. 每次在家里使用的启动顺序

每次使用时按这个顺序：

1. 确认办公室 Mac 开机、联网、Tailscale Connected
2. 家里 Windows 打开 Tailscale，确认 Connected
3. PowerShell 开 SSH 隧道：

```powershell
ssh -N -L 8080:47.236.3.167:8080 choco@100.105.189.124
```

4. 新开 PowerShell，进入 GenericAgent：

```powershell
cd C:\Users\你的用户名\Downloads\GenericAgent
py -3 launch.pyw
```

5. 打开启动日志里的 `http://localhost:185xx`

## 15. 常见问题

### `ConnectionError`

优先检查 SSH 隧道是否还开着：

```powershell
curl.exe http://127.0.0.1:8080
```

如果不通，重新建立 SSH 隧道。

### `ssh: connect to host ... port 22: Connection refused`

办公室 Mac 没开启“远程登录”，或防火墙阻止 SSH。

检查：

```text
系统设置 -> 通用 -> 共享 -> 远程登录
```

### `Permission denied`

用户名或密码不对。

当前办公室 Mac 用户名是：

```text
choco
```

### `python` 报语法错误

确认使用的是 Python 3：

```powershell
python --version
py -3 --version
```

GenericAgent 必须用 Python 3 启动。

### 端口 8080 被占用

改用 18080：

```powershell
ssh -N -L 18080:47.236.3.167:8080 choco@100.105.189.124
```

然后把配置里的：

```text
http://127.0.0.1:8080
```

改成：

```text
http://127.0.0.1:18080
```

## 16. 安全提醒

- 不要把 API token 写进公开仓库
- 不要把 `mykey.py` 提交到 Git
- 不要把办公室 Mac 的 SSH 暴露到公网
- SSH 隧道只转发 `127.0.0.1:8080 -> 47.236.3.167:8080`，不会代理所有网络
- 如果 token 曾经发到聊天或截图里，建议去服务端轮换新 token

