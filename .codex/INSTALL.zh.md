# 为 Codex 安装 Superpowers

通过原生技能发现机制在 Codex 中启用 superpowers 技能。只需克隆并创建符号链接即可。

## 前提条件

- Git

## 安装

1. **克隆 superpowers 仓库：**
   ```bash
   git clone https://github.com/obra/superpowers.git ~/.codex/superpowers
   ```

2. **创建技能符号链接：**
   ```bash
   mkdir -p ~/.agents/skills
   ln -s ~/.codex/superpowers/skills ~/.agents/skills/superpowers
   ```

   **Windows (PowerShell)：**
   ```powershell
   New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills"
   cmd /c mklink /J "$env:USERPROFILE\.agents\skills\superpowers" "$env:USERPROFILE\.codex\superpowers\skills"
   ```

3. **重启 Codex**（退出并重新启动 CLI）以发现技能。

## 从旧版引导程序迁移

如果你在原生技能发现功能推出之前安装了 superpowers，需要执行以下步骤：

1. **更新仓库：**
   ```bash
   cd ~/.codex/superpowers && git pull
   ```

2. **创建技能符号链接**（即上述第 2 步）—— 这是新的发现机制。

3. **从 `~/.codex/AGENTS.md` 中删除旧的引导块** —— 任何引用 `superpowers-codex bootstrap` 的块均不再需要。

4. **重启 Codex。**

## 验证

```bash
ls -la ~/.agents/skills/superpowers
```

你应该看到一个符号链接（Windows 上为目录联接），指向你的 superpowers 技能目录。

## 更新

```bash
cd ~/.codex/superpowers && git pull
```

技能通过符号链接即时更新。

## 卸载

```bash
rm ~/.agents/skills/superpowers
```

可选：删除克隆的仓库：`rm -rf ~/.codex/superpowers`。
