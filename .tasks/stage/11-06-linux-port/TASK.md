---
title: Overseer Linux 支援：外掛可用、worktree 生命週期可用
status: review
created: 2026-11-06
base_commit: f8e62d1f61e965990c206453fb52639bcd0c4e75
---
RESUME: （待驗收）

## 目標

在 Linux 上 `tern plugin install` 後，Overseer 的 Projects 面板可開啟、可新增 Git 專案、可由面板建立與刪除 worktree、PR/ticket 徽章正常；macOS 行為與現狀完全一致。

不使用編譯步驟：外掛是 Luau，執行時由 Tern 載入；Tern 本體已有 Linux 原生版。

## 驗收

- [x] 🎯 在 Linux（本機 Tern 0.9.12、Worktrunk 0.81）新增一個 Git 專案成功：選擇器可開、選完專案列出現，全程不需要 `/usr/bin/osascript`
- [x] 🎯 由面板建立 worktree：branch 與 worktree 產生、`creating` 列轉為一般列（本機實測，含 Worktrunk hook）
- [x] 🎯 由面板刪除 worktree：worktree 目錄消失、branch 保留（本機實測）
- [x] 沒有 zenity／kdialog 的桌面（如純 tty 或精簡 WM）仍能新增專案：退回輸入路徑的表單，不出現 `Folder picker failed`
- [x] macOS 分支的程式路徑仍以 `/usr/bin/osascript` 起步，且 `MACOS` 判定在 Linux 為 false（實測 Linux 走 zenity 分支）
- [x] `tests/` 在 Linux 全綠（目前 20 項），改動後重跑仍全綠
- [x] README、AGENTS.md（Known limitations）、CHANGELOG、`plugin.toml` 版本更新

## 做法

執行方式：main alone（改動集中在 3～5 個檔案、步驟有先後；研究已完成，無需平行切片）。

Phase 1 — 研究（已完成）：確認 Tern Linux 可用、外掛載入、面板渲染、worktree helper 與測試在 Linux 的行為。證據見下。
Phase 2 — 資料夾選擇器抽象（`window.luau`）：`MACOS`（`/usr/bin/osascript` 存在）→ 原 AppleScript；否則 `zenity --file-selection --directory`；否則 `kdialog --getexistingdirectory`；都沒有 → `cx:prompt` 輸入絕對路徑（取消碼分流：AppleScript -128、zenity/kdialog 1）。
Phase 3 — 二進位與 shell 路徑：
  - `worktrunk.luau` 新增 `platform_tool`（PATH → 絕對路徑 → 裸名）、`M.tern_binary()`、`M.bash()`、`M.sleep()`；`intercom.luau`、`window.luau`、`omplink.luau`、`worktree-operation.sh` 改用它們（`bash`、`sleep`、`pwd`、`kill`、`mktemp`、`mv`、`rm`、`tern`）。
  - 快捷鍵維持 `cmd+alt+shift+p`：Tern 在 Linux 把 `cmd` 映射成 Super，與 macOS 同一個鍵；實測 `ctrl+alt+shift+p` 已被 Tern preset 佔用（`plugin bind left out`），故不採用。
Phase 4 — Linux 端到端實測：隔離 `TERN_CONFIG_DIR`，以 zenity stub 驗證選擇器 argv 與新增專案；由面板建立／刪除 worktree；另以稀疏 PATH（無 zenity/kdialog）驗證輸入路徑的退路。
Phase 5 — 測試與文件：重跑 `tests/`（20/20）、更新 README/AGENTS.md/CHANGELOG、`plugin.toml` 2.4.0。

## 分工

| 切片 | 職責 | 代理 | 驗收 |
|---|---|---|---|
| 全部 | 研究、實作、Linux 實測、文件 | main | 上述 🎯 三項 |

## 證據

- Tern Linux：`tern 0.9.12 (e498b49)`，ELF x86-64；`tern plugin list` → `overseer 2.3.3 — 1 blocks, 0 lenses, window  ready`。
- 面板渲染：Projects 空狀態、`Add your first project`、指令面板顯示 `Show or hide Projects` 與 `Super Alt Shift P`。
- 阻斷點：按新增專案 → `Couldn't open the folder picker: tern.process.run /usr/bin/osascript: No such file or directory`，toast `Folder picker failed`（`window.luau:1913`）。
- 原型：改 25 行加 zenity 分支後，`zenity --file-selection --directory --filename=…` 成功啟動（原型已還原，未留在 repo）。
- Worktrunk 0.81 Linux（x86_64-musl）：
  - `wt -C repo switch --create --base main --no-cd feat/x` → 建立成功。
  - `wt -C repo list --format=json` → `schema: 2`，欄位 `items[].worktree.{path,main,detached,changes{diff}}` 與 `parse_worktrees` 相符。
- `worktree-operation.sh` 直接執行：create → receipt `0`、worktree 產生；remove → receipt `0`、目錄消失、`feat/y` 分支保留。
- 測試（PATH 加 wt）：`tests/test_worktree_operation.py` 12/12 OK；`tests/test_subworktree.py` 8/8 OK。
- 無原生選擇器 API：`tern plugin types` 產生的定義只有 `cx:pick`（項目選單），沒有檔案系統選擇器。
- 實作後 E2E（Linux 7.0 核心、Tern 0.9.12、Worktrunk 0.81、隔離 `TERN_CONFIG_DIR`，外掛在 `/mnt/data/Projects/overseer`）：
  - 載入：`tern plugin list` → `overseer 2.4.0 … window  ready`；無載入錯誤。
  - 新增專案：`plugins run plugin.overseer.add` → 選擇器 argv 為 `--file-selection --directory --title=… --filename=/home/chris/`；選完 `projects` 出現 `{"path":"/tmp/e2e/demo","base":"main"}`。（以 stub 代替對話框，只為自動化；真實 zenity 開啟已另行確認。）
  - 建立 worktree：點面板 `+`、輸入 `feat/linux-e2e`、Enter → `git worktree list` 出現 `/tmp/e2e/demo.feat-linux-e2e`，面板列為 `feat/linux-e2e`，toast `Worktree created`；`ops.log`：`create … finished status 0`。
  - 刪除 worktree：右鍵列 → `Delete worktree` → `git worktree list` 只剩主 checkout、`git branch --list feat/linux-e2e` 仍在、toast `Worktree deleted`；`ops.log`：`remove … finished status 0`。
  - 無對話框退路：以稀疏 PATH（無 zenity/kdialog）啟動 Tern → `/tmp/e2e2/tern.out` 出現 `no dialog tool (install zenity or kdialog)`；`plugin.overseer.add` 跳出輸入框（placeholder `Absolute path to a Git repository`），輸入 `/tmp/e2e/demo` 後專案加入成功。
  - 快捷鍵：`Super Alt Shift P` 註冊無 `bind left out` 警告且觸發 `togglePanel`；`ctrl+alt+shift+p` 實測被 preset 佔用而略過。
  - 測試（改動後重跑，PATH 加 wt）：`test_worktree_operation.py` 12/12 OK、`test_subworktree.py` 8/8 OK。
  - macOS：本機無 macOS，未實機驗證；程式碼上 `MACOS` 為 false 時才走 zenity/kdialog，macOS 分支與原本相同。
- 變更報告：`.tasks/stage/11-06-linux-port/report.html`（oil-diff，`lint_report.py` → `PASS: 0 errors, 0 warnings`），範圍 `f8e62d1..WT`，14 個檔案 +241 −35。

## 紀錄

- 2026-11-06 建立
- 2026-11-06 Phase 1 研究完成：外掛可載入、面板可渲染；唯一功能阻斷為 osascript；worktree 建立/刪除在 Linux 的 helper 與測試全數通過。
- 2026-11-06 依文字計畫開工
- 2026-11-06 開始實作
- 2026-11-06 實作完成並在 Linux 端到端驗證：picker（osascript→zenity→kdialog→輸入路徑）、工具路徑解析、文件與版本 2.4.0；測試 20/20。
- 2026-11-06 改動報告：report.html
- 2026-11-06 送交驗收
