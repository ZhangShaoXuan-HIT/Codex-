# 修复 ChatGPT 桌面版更新后路径错误问题 —— winget 安装官方 Codex CLI

> 适用系统：Windows（PowerShell）
> 更新时间：2026-08-26

---

## 一、问题背景

ChatGPT 桌面版（Electron 应用）启动时报错：

```
ChatGPT failed to start.
Unable to locate the Codex CLI binary.
Set CODEX_CLI_PATH or ensure the Electron resources include bin/codex.
```

### 解决思路

改用 **winget 安装 OpenAI 官方发布的 Codex 包**，它自带真正的 `codex.exe` 可执行文件，再把 `CODEX_CLI_PATH` 指向该 `.exe` 路径。

---

## 二、操作步骤

### 第 1 步：检查当前环境（可选）

在 PowerShell 中确认当前 `CODEX_CLI_PATH` 指向什么：

```powershell
# 查看当前环境变量值
[Environment]::GetEnvironmentVariable("CODEX_CLI_PATH", "User")

# 确认它是不是 .cmd（若是，即为问题根源）
Get-Item ([Environment]::GetEnvironmentVariable("CODEX_CLI_PATH", "User")) | Select-Object Extension
```

若 `Extension` 显示 `.cmd`，说明就是本文要解决的问题。

### 第 2 步：用 winget 安装官方 Codex

```powershell
# 安装官方版（自带真正的 codex.exe）
winget install OpenAI.Codex
```

> 提示：若已安装过，可先升级：
> ```powershell
> winget upgrade OpenAI.Codex
> ```

### 第 3 步：定位 codex.exe 的真实路径

```powershell
# 在常见安装目录中搜索 codex.exe
Get-ChildItem "$env:LOCALAPPDATA\Programs" -Recurse -Filter codex.exe -ErrorAction SilentlyContinue | Select-Object FullName
```

如果上面没有结果，扩大搜索范围：

```powershell
Get-ChildItem "$env:LOCALAPPDATA" -Recurse -Filter codex.exe -ErrorAction SilentlyContinue | Select-Object FullName
```

记录返回的完整路径，形如：

```
C:\Users\<用户名>\AppData\Local\Programs\Codex\codex.exe
```

### 第 4 步：设置 CODEX_CLI_PATH 指向 codex.exe

把下面的路径**替换为第 3 步查到的真实路径**：

```powershell
[Environment]::SetEnvironmentVariable(
    "CODEX_CLI_PATH",
    "C:\Users\<你的用户名>\AppData\Local\Programs\Codex\codex.exe",
    "User"
)
```

### 第 5 步：重启并验证

1. **关闭所有终端窗口和 ChatGPT 应用**（环境变量只在应用启动时读取）；
2. 重新打开 ChatGPT 桌面版，确认不再报错；
3. 可在新开的 PowerShell 中验证变量已生效：

```powershell
# 确认变量值
[Environment]::GetEnvironmentVariable("CODEX_CLI_PATH", "User")

# 确认 codex 可正常执行
codex --version
```

---

## 三、快速自查命令汇总

```powershell
# 查看当前 CODEX_CLI_PATH
[Environment]::GetEnvironmentVariable("CODEX_CLI_PATH", "User")

# 确认指向的文件扩展名（.cmd 即问题所在）
Get-Item ([Environment]::GetEnvironmentVariable("CODEX_CLI_PATH", "User")) | Select-Object Extension

# 搜索 codex.exe 真实位置
Get-ChildItem "$env:LOCALAPPDATA\Programs" -Recurse -Filter codex.exe -ErrorAction SilentlyContinue | Select-Object FullName

# 验证 codex 可用
codex --version
```

---

## 四、常见问题（FAQ）

### Q1：winget 安装失败或找不到 OpenAI.Codex 包？

- 确认 winget 版本较新：`winget --version`
- 可改用 npm 方案后直接找包内 exe：
  ```powershell
  Get-ChildItem "$env:APPDATA\npm\node_modules\@openai\codex" -Recurse -Filter *.exe | Select-Object FullName
  ```

### Q2：设置了 exe 路径后仍然报错？

- 检查路径中是否包含空格或特殊字符，确认 PowerShell 命令中的引号完整；
- 确认 `codex.exe` 文件真实存在（`Test-Path "路径"`）；
- 若仍失败，可能是桌面版自身 bug，参考「兜底方案」。

### Q3：非 Windows 系统怎么办？

- **macOS**：用 `find /Applications -name codex -type f` 或 `which codex` 找到二进制，再执行：
  ```bash
  launchctl setenv CODEX_CLI_PATH /实际/路径/codex
  ```
- **Linux**：将 `export CODEX_CLI_PATH=/实际/路径/codex` 写入 `~/.zshrc` 或 `~/.bashrc` 后 `source`。

---

## 五、兜底方案

如果以上均无效（可能是桌面版自身 bug），建议：

1. **删除环境变量**，排除干扰：
   ```powershell
   [Environment]::SetEnvironmentVariable("CODEX_CLI_PATH", $null, "User")
   ```
2. **彻底重装 ChatGPT 桌面版**：卸载 → 清理残留 → 从官网 [openai.com/chatgpt/download](https://openai.com/chatgpt/download) 下载最新版安装；
3. **直接用 Codex CLI**（功能与桌面版一致）：
   ```powershell
   cd 你的项目目录
   codex
   ```

---

## 附：参考资料

- ChatGPT 桌面版官方下载：https://openai.com/chatgpt/download
- OpenAI Codex GitHub Releases：https://github.com/openai/codex/releases
- 相关讨论：openai/codex GitHub Issues（#25671、#25886、#27979）
