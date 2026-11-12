# 資料夾選擇器改用桌面原生對話框（KDE 走 kdialog／Dolphin 的 widget）

## 驗收

- [x] 🎯 KDE 桌面上按新增專案，開的是 KDE 檔案對話框（places、麵包屑、縮圖），不是 zenity 的 GTK 清單
- [x] 非 KDE 桌面（GNOME 等）仍走 `zenity --file-selection --directory`
- [x] 兩者都缺時仍退回面板上的輸入路徑表單，不出現 `Folder picker failed`
- [x] macOS 分支不變（`/usr/bin/osascript` 探測為 false 時才進 Linux 分支）
- [x] CHANGELOG、AGENTS.md、DECISIONS.md、`plugin.toml`（2.4.1）更新

## 做法

Phase 1 — 現況：`window.luau` 的 `folderPicker` 先 zenity、後 kdialog，KDE 上因此開到 GTK 的清單。
Phase 2 — 改為依桌面排序：`XDG_CURRENT_DESKTOP` 或 `XDG_SESSION_DESKTOP` 含 `KDE` 時 kdialog 優先，否則 zenity 優先；兩者互為退路，皆無 → `cx:prompt`。
  - `kdialog --title <標題> --getexistingdirectory <start>`；取消碼同為 1。
  - 否決直接啟動 `dolphin`：沒有「選完回傳路徑」模式。portal 仍否決（見 DECISIONS.md）。
Phase 3 — 驗證：隔離 `TERN_CONFIG_DIR`／`XDG_STATE_HOME`，以 stub 記錄 argv，再以真實 kdialog 在 Xvfb 內跑完整鏈路。

## 證據

- KDE（`XDG_CURRENT_DESKTOP=KDE`）：`plugins run plugin.overseer.add` → stub 記錄 `kdialog --title Choose a Git repository to add to Projects --getexistingdirectory /home/chris`，zenity 未被呼叫。
- GNOME（`XDG_CURRENT_DESKTOP=GNOME`、`XDG_SESSION_DESKTOP=gnome`）：記錄 `zenity --file-selection --directory --title=… --filename=/tmp/pick-e2e/`。
- 真實 kdialog（Xvfb、隔離 config、無 stub）：`xdotool search --name "Choose a Git repository"` 找到視窗，截圖顯示 KDE 檔案對話框（`/tmp/pick-real.png`）。
- 手動確認：`kdialog --getexistingdirectory` 在本機開出與 Dolphin 同一個 widget。
- `tests/`：安裝 Worktrunk 0.81.0 後重跑，`test_worktree_operation.py` 12/12 OK、`test_subworktree.py` 8/8 OK（先前因本機缺 `wt` 全數 skip）。

## 紀錄

- 2026-11-07 建立並完成
