# CCG-Workflow 安全分析報告

> **分析日期**: 2026-03-24
> **版本**: v1.7.14
> **分析範圍**: 完整原始碼（TypeScript CLI + Go wrapper + 模板 + 預編譯二進位）

---

## 零、已修復項目

以下風險已在本次提交中修復：

| 原等級 | 項目 | 修復方式 |
|--------|------|----------|
| ~~高~~ → 已修復 | Token 明文儲存 | 改用 `env.ACE_TOOL_TOKEN` 環境變數傳遞，不再寫入 CLI args |
| ~~低~~ → 已修復 | ROLE_FILE 路徑讀取 | 新增家目錄限制 + symlink 解析防護 |

---

## 一、總體評估

**結論：此程式碼庫整體安全，適合使用。** 主要風險已修復，剩餘為中等風險項目。

| 等級 | 數量 | 說明 |
|------|------|------|
| 嚴重 (Critical) | 0 | 無 |
| 高風險 (High) | 0 | Token 明文儲存 → **已修復** |
| 中風險 (Medium) | 2 | 二進位供應鏈信任、npm 依賴 |
| 低風險 (Low) | 1 | Shell RC 修改（已有重複檢測） |

---

## 二、安全優點

1. **無任意程式碼執行** — 無 `eval()`、`Function()`、動態程式碼產生
2. **無命令注入風險** — Go wrapper 使用 `exec.CommandContext()` 直接執行二進位，不經 shell 解譯
3. **無權限提升** — 從不要求 sudo/admin
4. **所有變更限於使用者家目錄** — `~/.claude/` 範圍內
5. **修改共享配置前自動備份** — `~/.claude/backup/`
6. **密碼輸入有遮罩** — inquirer 的 `password` type + `mask: '*'`
7. **無遙測/追蹤** — 不會電話回撥、無分析
8. **安全的依賴項** — 所有 runtime 依賴皆為知名套件，無網路依賴
9. **跨平台路徑安全** — 使用 `pathe` 庫處理，Windows/Unix 皆正確
10. **Go wrapper 參數以陣列傳遞** — 避免 shell injection by design

---

## 三、風險詳細分析

### 3.1 ~~高風險~~ 已修復：Token 明文儲存於 ~/.claude.json

**位置**: `src/utils/installer.ts` — `installAceTool()`

**修復前**：Token 以 CLI 參數形式寫入 `--token YOUR_TOKEN`，暴露於 process list 和配置檔。

**修復後**：Token 改用 `env` 欄位傳遞，MCP 配置變為：

```json
{
  "mcpServers": {
    "ace-tool": {
      "command": "npx",
      "args": ["-y", "ace-tool@latest"],
      "env": {
        "ACE_TOOL_TOKEN": "使用者的TOKEN在此"
      }
    }
  }
}
```

**改善**:
- Token 不再出現在 `ps aux` process list 中
- 環境變數僅對子程序可見，不暴露於命令列
- `~/.claude.json` 仍含 token（MCP 設計限制），建議保護家目錄權限

### 3.2 中風險：預編譯二進位的供應鏈信任

**位置**: `bin/codeagent-wrapper-*`（6 個平台二進位）

- 二進位隨 npm 套件發佈，無密碼學校驗（無 checksum/signature）
- 安裝時僅驗證 `--version` 能否執行

**緩解**: npm registry 本身提供套件完整性驗證。

**建議**: 發佈時附帶 SHA256 checksums 文件。

### 3.3 中風險：npm Registry 依賴

**位置**: `src/utils/version.ts:45`, `src/commands/update.ts:257`

```typescript
await execAsync(`npm view ccg-workflow version`)
await execAsync(`npx --yes ccg-workflow@latest --version`)
```

- 版本檢查與更新依賴 npm registry
- 若 registry 被攻破可能下載惡意套件

**緩解**: 使用者須明確觸發更新；命令為硬編碼字串（無使用者輸入注入）。

### 3.4 低風險：Shell RC 檔案修改

**位置**: `src/commands/init.ts:334`

自動附加至 `~/.bashrc` 或 `~/.zshrc`：
```bash
export PATH="~/.claude/bin:$PATH"
```

- 多次執行可能產生重複條目
- 需使用者明確確認

### 3.5 ~~低風險~~ 已修復：ROLE_FILE 路徑讀取

**位置**: `codeagent-wrapper/utils.go` — `injectRoleFile()`

**修復前**：允許透過 `ROLE_FILE:` 指令讀取任意檔案路徑，無限制。

**修復後**：新增安全檢查：
- 解析絕對路徑 (`filepath.Abs`)
- 解析 symlink (`filepath.EvalSymlinks`) 防止符號連結繞過
- 驗證最終路徑位於使用者家目錄內
- 拒絕家目錄外的路徑並記錄警告

---

## 四、安裝流程分析

### 系統寫入的檔案

| 檔案 | 用途 | 敏感度 |
|------|------|--------|
| `~/.claude/commands/ccg/*.md` | 14 個斜杠命令模板 | 低 |
| `~/.claude/agents/ccg/*.md` | 4 個子智能體模板 | 低 |
| `~/.claude/.ccg/prompts/**/*.md` | 12+6 個專家提示詞 | 低 |
| `~/.claude/.ccg/config.toml` | CCG 配置（無密鑰） | 低 |
| `~/.claude/bin/codeagent-wrapper` | 平台二進位 (755) | 中 |
| `~/.claude.json` | MCP 配置 **含 token** | **高** |
| `~/.bashrc` 或 `~/.zshrc` | PATH 環境變數 | 低 |

### 網路存取

| 時機 | 目的 | 觸發方式 |
|------|------|----------|
| `update` 命令 | 檢查 npm 最新版本 | 使用者主動 |
| `npx ccg-workflow@latest` | 下載更新 | 使用者確認後 |

**安裝過程本身不下載任何外部資源**（所有模板與二進位隨 npm 套件一起發佈）。

---

## 五、Go Wrapper 安全分析

Go wrapper (`codeagent-wrapper`) 是核心執行元件，負責呼叫 Codex/Gemini/Claude CLI：

| 安全面向 | 評估 | 說明 |
|----------|------|------|
| 命令注入 | **安全** | 使用 `exec.CommandContext()`，不經 shell |
| 參數傳遞 | **安全** | 陣列式 argv[]，非字串串接 |
| stdin 處理 | **安全** | 原始位元組寫入，無解譯 |
| 路徑遍歷 | **受保護** | `filepath.Rel()` + 前綴檢查 |
| 信號處理 | **安全** | Context-based cancellation + graceful termination |
| 並發安全 | **安全** | `sync.WaitGroup`、channels、`atomic.Pointer` |
| 外部依賴 | **零** | 純 Go 標準庫 |

**特殊安全機制**：當任務含 `$`、反引號、換行等特殊字元時，自動切換為 stdin 模式，增加一層防護。

---

## 六、用法說明

### 6.1 這是什麼？

CCG (Claude + Codex + Gemini) 是一個多模型 AI 協作系統，安裝為 Claude Code 的擴展命令。它提供：

- **14 個斜杠命令**（`/ccg:workflow`, `/ccg:frontend`, `/ccg:backend` 等）
- **固定路由**：Gemini 處理前端、Codex 處理後端、Claude 負責編排
- **codeagent-wrapper**：Go 編寫的跨平台二進位，負責呼叫外部模型 CLI

### 6.2 基本使用流程

```bash
# 1. 安裝
npx ccg-workflow

# 2. 在 Claude Code 中使用斜杠命令
/ccg:workflow    # 完整 6 階段工作流
/ccg:frontend   # 前端專項（Gemini）
/ccg:backend    # 後端專項（Codex）
/ccg:feat       # 智能功能開發
/ccg:analyze    # 技術分析
/ccg:debug      # 問題診斷 + 修復
/ccg:review     # 程式碼審查
/ccg:commit     # 智能提交
```

### 6.3 前提條件

- **Node.js** (支援 npx)
- **Claude Code CLI** 已安裝
- **Codex CLI** 和/或 **Gemini CLI** 已安裝（取決於使用的命令）
- （可選）ace-tool API token（用於 prompt 增強）

---

## 七、導入建議

### 7.1 推薦導入方式

```bash
# 直接從 npm 安裝（最簡單、最安全）
npx ccg-workflow
```

這是官方推薦方式，透過 npm registry 的完整性校驗保障安全。

### 7.2 如果需要審計或自訂

```bash
# 1. Clone 原始碼
git clone https://github.com/Lnanhung14/ccg-workflow.git
cd ccg-workflow

# 2. 審計程式碼（特別關注以下檔案）
#    - src/utils/installer.ts  (安裝邏輯)
#    - src/utils/mcp.ts        (MCP 配置)
#    - codeagent-wrapper/       (Go wrapper)

# 3. 本地建構
pnpm install
pnpm build

# 4. 本地測試
node bin/ccg.mjs
```

### 7.3 安全加固建議

1. **不使用 ace-tool MCP**（跳過 token 配置）— 消除最大風險項
2. **定期檢查 `~/.claude.json`** — 確認無非預期的 MCP server 配置
3. **Pin 版本號** — `npx ccg-workflow@1.7.14` 而非 `@latest`
4. **審查二進位** — 可從原始碼重新編譯 Go wrapper：
   ```bash
   cd codeagent-wrapper
   go build -o ../bin/codeagent-wrapper-$(go env GOOS)-$(go env GOARCH)
   ```

### 7.4 團隊導入注意事項

- 此工具修改使用者個人目錄（`~/.claude/`），不影響專案倉庫
- 每位團隊成員需各自安裝
- 配置檔案 `~/.claude/.ccg/config.toml` 可統一管理
- 如有 ace-tool token，建議透過環境變數而非 config 檔案傳遞
