# Spec Kit 使用練習

References:

- [github: Spec Kit](https://github.com/github/spec-kit)
- [github: 保哥 Spec Kit 翻譯](https://github.com/doggy8088/spec-kit)
- [多奇教育: SDD with Spec Kit](https://learn.duotify.com/courses/sdd)
-

---

### 安裝 Spec Kit

1. 使用 PowerShell 5 + (win+R -> `powershell`)
    ```
    # 安裝 Scoop 套件管理器
    Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force
    Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression
    
    # 安裝 pwsh、git、uv 與 nvm
    scoop install pwsh git uv nvm
    
    # 安裝搜尋文字檔案的利器 ripgrep
    scoop install ripgrep
    
    # 安裝 Windows Terminal
    scoop bucket add extras
    scoop install extras/windows-terminal
    
    # 更新 uv 的路徑
    uv tool update-shell
    
    # 重新啟動檔案總管，讓環境變數生效
    Stop-Process -Name explorer -Force; Start-Process explorer
    ```
2. 開啟 Windows Terminal 終端機視窗 (win+R -> `wt`)
   ```
    # 透過 nvm 安裝 Node.js LTS 版本
    nvm install lts
    nvm use lts
    
    # 安裝 Spec Kit 的 Specify CLI 命令列工具
    uv tool install specify-cli --from git+https://github.com/github/spec-kit.git
    
    # 安裝 Copilot CLI (請自由選擇)
    npm install -g @github/copilot
    
    # 安裝 Claude Code (請自由選擇)
    npm install -g @anthropic-ai/claude-code
    
    # 檢查 Spec Kit 需要的工具支援
    specify check
   ```
3. 至少需要看到 Git version control 是綠燈, 還有 GitHub Copilot 或 Claude Code 或任何一套 AI Coding 工具是綠燈.
4. 升級 `uv tool install specify-cli --force --from git+https://github.com/github/spec-kit.git`

---

## START

1. 在專案根目錄初始化 Spec Kit 專案範本 `specify init --ai copilot --script ps --here`
2. 建立一個 `AGENTS.md` 檔案 (給 Copilot CLI 看的)
3. `git add AGENTS.md && git commit -m "Add AGENTS.md"`
