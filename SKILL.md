---
name: writing-bigfix-fixlets
description: Use when writing or modifying BigFix Fixlets (.bes files), Relevance expressions, or ActionScript — registry inspectors, default value checks, WOW64 registry views, qna / Fixlet Debugger validation, action telemetry, RemoveMSI. Also use when a Fixlet shows wrong Applicable Computers count or actions report Failed despite succeeding.
---

# Writing BigFix Fixlets

## Overview

BigFix Relevance 不是 PowerShell、不是 WMI、不是你猜的樣子。**沒被真引擎（qna / Fixlet Debugger / Console）求值過的 Relevance 就是猜測**——用 PowerShell 重寫邏輯來「驗證」只能驗你的想法，驗不出語法錯誤。

## 鐵則

```
.bes 上線前，每一句 Relevance 都必須經過真的 relevance 引擎求值，
且要在「該匹配」和「不該匹配」兩種機器狀態下各驗一次。
```

實測教訓：`value "(default)"` 寫法通過了 PowerShell 模擬驗證、通過了人工 review，進了 .bes——真引擎上恆 False。若上線：Relevance 過度匹配全部機器 + `continue if` 讓所有 action 假性失敗。

## Registry Inspector 速查（每一條都是實際踩過或看過人踩的坑）

| 要做什麼 | 正確寫法 | 常見錯誤寫法 |
|---|---|---|
| 讀 key 的預設值 | `default value of <registry key>` | `value "(default)" of ...`（regedit UI 標籤，恆不存在）；`value "" of ...`（未文件化，別賭） |
| 32-bit client 讀 64-bit registry | `key "HKLM\..." of native registry` | 以為 `native registry` 會被 WOW64 重導向而繞路——**錯**，它的用途就是取原生視圖。`registry` 才是 32-bit 視圖 |
| 防禦性讀值 | `whose (exists default value of it AND default value of it as string as lowercase contains "xxx")` | 少了 `exists` 守衛：值不存在時 singular 取值會 error，整條 relevance 變 error |
| 64-bit Program Files | `program files x64 folder` | `program files folder`（32-bit client 拿到 x86 的） |
| Client setting 帶預設值 | `value of setting "X" of client \| "fallback"` | 忘了 `\|`，setting 不存在時 error |

其他已驗證可用：`platform id of operating system != 3`（排除 WinCE 的標準護欄）、`setting "N"="V" on "{now}" for client`（Action Guide 原文例）。

## 驗證流程（不需要 BigFix Server/Client）

1. 下載 standalone Fixlet Debugger：`http://software.bigfix.com/download/bes/92/FixletDebugger-9.2.0.363.zip`（SHA1 `b3e41b555ab09fdf5e72bceb80e07371f756ad45`；zip 內含 qna.exe。Linux/macOS 的 qna 跟著 BES Client 裝）
2. **qna.exe 9.2 是 GUI**：stdin/stdout 重導向無效；**絕不透過 SSH 啟動**（pipe 永不關 → 塞爆 sshd MaxStartups → 整台 SSH 斷線，症狀 = banner exchange timeout）。在機器 console 上開，File→Open 載入問題檔，按 Evaluate
3. 問題檔格式：每行 `Q: <relevance>`，求值後每題出 `A:` 或 `E:`（錯誤）
4. 問題檔設計：**新舊寫法對照** + 每個 relevance 條件單獨一題（好定位哪句錯）+ ActionScript 裡 `{ }` substitution 的內容也抽出來驗
5. 兩種狀態各驗一次：目標狀態機器（該 MATCH）與已修復/健康機器（該 NOT MATCH）——後者驗「自限制」，防 policy action 無限重打

## ActionScript 遙測 Pattern（避免「跑完不知道狀態」）

三管道，全部放在最後的 `continue if` **之前**（驗證失敗也要留下紀錄）：

```
// 1. Endpoint log：修復前後各 dump 一次（BEFORE dump 放在第一個 continue if「之前」，早期失敗也留痕）
waithidden cmd.exe /c "(echo --- BEFORE --- & assoc .docx & ftype Word.Document.12) >> C:\XxxLog\fix.log 2>&1"
// 2. Client setting：Console 可查
setting "Xxx_Result"="{if <驗證relevance> then "OK" else "FAIL"}" on "{now}" for client
// 3. 配套 Analysis：properties 讀 setting + 即時 registry 狀態 → 全機隊儀表板
// 4. 捕捉外部程式 exit code：必須 /v:on + !errorlevel!（延遲展開）
waithidden cmd.exe /v:on /c "setup.exe /configure x.xml & echo EXIT_CODE=!errorlevel! >> C:\XxxLog\fix.log"
```

⚠️ **`%errorlevel%` 在 `cmd /c "a & echo %errorlevel%"` 是解析時展開**——永遠記到執行前的值（通常 0），不是 a 的 exit code。實測：`cmd /c "cmd /c exit 3 & echo %errorlevel%"` → 0；`cmd /v:on /c "... & echo !errorlevel!"` → 3。

`continue if` 行號本身是失敗定位器（Console 顯示 action 停在哪行）。

## Common Mistakes

| 症狀 | 原因 |
|---|---|
| Applicable Computers ≈ 全部機器 | 某條件恆 True（常見：`exists value "(default)"` 恆 False 被 `not exists ... whose` 反轉） |
| Action 全報 Failed 但機器實際修好了 | 最後的 `continue if` 用了錯誤語法恆 False |
| 修好後機器仍 relevant → 重複派送 | Relevance 沒把「已修復」排除，或用了恆真條件 |
| exit code 0 但功能壞 | setup.exe 類工具不檢查副作用（如 Office `<RemoveMSI/>` 清空 assoc/ftype） |
| 大小寫比對漏抓 | registry 值大小寫不定（`Root\Office16` vs `root\Office16`），一律 `as lowercase` 再比 |

## 官方參考

- developer.bigfix.com：`/relevance/reference/registry.html`、`/action-script/reference/`（waithidden、continue-if、parameter、appendfile）
- HCL BigFix Relevance Guide / Action Guide PDF（可下載，語法權威）
- 遇到沒把握的 inspector：先 grep Guide PDF（`pdftotext` 後搜），再 qna 實證，兩關都過才進 .bes
