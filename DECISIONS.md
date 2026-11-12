# 決策紀錄

## 2026-11-08 模組化 AGENTS.md 規格文件

- **核心原則：不可妥協的硬性約束留守根目錄，查閱型細節外部化至 `docs/`。**
  - 根目錄 `AGENTS.md` 會被自動注入至所有 Agent Session 的 Context，原本 325 行的內容在每輪對話皆造成被動 Token 負擔；精簡後（縮減約 70% 體積）僅保留 50 ms 規則、單一面板不變量、安全防護與檔案對照。
  - 拆分出四份參考文件：`docs/state.md`（KV Schema、磁碟收據、記憶體狀態）、`docs/flows.md`（Refresh、Tick 健康度、Coordination、Sub-worktrees、建立/刪除流程）、`docs/behavior.md`（UI 面板與選單完整行為、Carly/Visualize PR 規格、安全與故障排除）、`docs/development.md`（測試執行、隔離視窗除錯、已知限制）。
- **保留強制查閱指引（Mandatory Pointers）。** 在根目錄 `AGENTS.md` 頂部明訂改動特定子系統前「MUST read」對應的外部文件，避免 Agent 因未查閱規格而破壞 KV 結構或打破既有流程假定。
  - 否決直接刪減細節：Overseer 包含許多特殊的邊界防護（如 50ms 懲罰避險、gone worktree 不自動關閉 session 等），若移除會導致後續開發產生猜測。
  - 否決合成為單一龐大參考文件：單一大檔依然難以在對話中精準取閱，按「狀態、流程、行為、開發」四維度拆分最貼合修改程式時的上下文需求。

## 2026-11-06 Linux 支援（.tasks/stage/11-06-linux-port）

- **不引入建置步驟。** Overseer 是 Tern 外掛（Luau），執行時由 Tern 載入；Linux 需要的只是 Tern 本體，而它已有原生版。因此「編譯 Linux 版」實際上是移除平台假設，不是加一條 toolchain。
- **資料夾選擇器：osascript →（KDE 上 kdialog 優先）→ zenity／kdialog → 輸入路徑。** 依 `/usr/bin/osascript` 是否存在判斷平台；Linux 先看 `XDG_CURRENT_DESKTOP`／`XDG_SESSION_DESKTOP` 是否含 KDE：是則 `kdialog --getexistingdirectory` 優先，否則 `zenity --file-selection --directory` 優先，兩者互為退路；都沒有時用 `cx:prompt` 請使用者輸入絕對路徑。
  - `kdialog` 開的就是 KDE 檔案對話框（與 Dolphin 同一個 widget：places、縮圖、麵包屑），比 zenity 的 GTK 清單好用；zenity 在 GNOME 上才是原生。
  - 否決直接啟動 `dolphin`：Dolphin 沒有「選完資料夾回傳路徑」的模式（`--select` 只開窗、不輸出），拿不到結果。
  - 否決 xdg-desktop-portal：需要額外 D-Bus client 與非同步介面，成本遠高於收益。
  - 否決自繪檔案瀏覽器：等於在面板裡重做一個檔案管理員，維護負擔大。
- **快捷鍵維持 `cmd+alt+shift+p`。** Tern 在 Linux 把 `cmd` 映射為 Super，因此 macOS 與 Linux 是同一個實體組合鍵。
  - 否決 `ctrl+alt+shift+p`：Tern preset 已佔用（實測 `plugin bind left out`），`ctrl+alt+shift+o` 同樣被佔用。
- **外部工具以 PATH 優先，其次絕對路徑，最後裸名**（`bash`、`sleep`、`pwd`、`kill`、`mktemp`、`mv`、`rm`、`tern`）。
  - 否決繼續寫死 `/bin/...`：usrmerge 發行版可用，但 NixOS 類系統與稀疏 PATH 的桌面啟動會失效。
- **平台判斷用檔案探測（`/usr/bin/osascript`）而非 `uname`。** Tern 沒有公開的平台 API，而 `uname` 要多起一個子行程；osascript 正好是 picker 分支的判準，兩者共用同一個開關。
- **版本 2.4.0。** 使用者可見的新平台支援，不動面板或狀態結構，因此是 minor 版。
