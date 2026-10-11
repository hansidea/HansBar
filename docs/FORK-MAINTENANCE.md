---
summary: "HansBar 個人 Fork 的分支用途、安全建置及上游更新流程。"
read_when:
  - 開始 HansBar 客製開發
  - 建置原版或合併 CodexBar 上游更新
---

# HansBar 維護指南

## 遠端與分支

「遠端」是 GitHub 上的版本庫；「分支」是同一個專案的不同開發路線。

| 名稱 | 用途 |
| --- | --- |
| `origin` | 自己的 Fork：`https://github.com/hansidea/HansBar`，個人修改推送到這裡。 |
| `upstream` | 原作者：`https://github.com/steipete/CodexBar.git`，只取回更新，不向這裡推送。 |
| `main` | 保留原版程式碼，用來承接上游更新；不加入 HansBar 客製內容。 |
| `hansbar` | 個人開發分支，客製功能及本文件都放在這裡；追蹤 `origin/hansbar`。 |

初始化從原有 `main` 的 `444821be5` 建立 `hansbar`；已確認此基準屬於 `upstream/main` 歷史。
沒有為了初始化追進其他版本，也沒有修改 App 名稱、Bundle ID、圖示或功能。
本機 `remote.pushDefault` 設為 `origin`。這是預設目的地，不是阻擋手動推送 upstream 的權限防護；仍應明確使用 `git push origin ...`。

## 安全界線

- 初始化時工作目錄乾淨，沒有需要暫存或備份的未提交修改。
- 本機 `/Applications/CodexBar.app` 已存在。不要把本機建置拖進「應用程式」覆蓋它。
- `Scripts/compile_and_run.sh`、`Scripts/launch.sh` 及部分 Make 啟動命令會停止同名 CodexBar，不適合與既有安裝並行驗證。
- `Scripts/package_app.sh` 在版本庫內產生及替換 `CodexBar.app`，不安裝到 `/Applications`；以 `CODEXBAR_SIGNING=adhoc` 避免選用正式簽署憑證。
- App 仍使用原版身份。一般方式啟動可能共用偏好設定、`~/.config/codexbar/config.json`／舊的 `~/.codexbar/config.json`、App Group 資料及服務登入來源。只改資料夾名稱或單獨設定 `HOME` 不等於完整隔離。
- 原版自動更新來源也尚未客製；不要在個人開發版操作自動更新。
- 本次採用上游既有的 `Scripts/verify_packaged_app_launch.sh`：複製 App 到暫存區、清空繼承環境、使用合成設定並停用供應商、隔離檔案與 Keychain 存取、禁止網路及寫入真實家目錄。測試只停止自己的程序，結束後清除暫存區。
- 這是短暫的啟動驗證，不是可登入帳號長期使用的隔離安裝，也不代表已驗證真實帳號用量或完整 UI。
- 不使用 `hard reset`、強制推送或丟棄修改。不讀取或輸出 API Key、Cookie、Token 或登入憑證內容。

## 開發工具

本次檢查：Apple Silicon arm64、macOS 27.0.1、Xcode 27.0（27A266a）、Swift 6.4、macOS SDK 27.0。
Command Line Tools 選用 `/Applications/Xcode.app/Contents/Developer`，`xcrun` 可找到編譯器及 SDK。
Python 原為 Xcode 附帶的 3.9.6，缺少完整測試要求的 `os.waitid`。
經使用者同意，以既有 Homebrew 安裝 `python@3.14`；執行檢查時只調整該命令的 PATH，不替換系統 Python。
Git、Make、Node.js、codesign、sandbox-exec 及 GitHub CLI 已存在。
SwiftFormat、SwiftLint 等由 `make check` 呼叫專案的固定版本安裝腳本，放在 `.build/lint-tools`。

`xcodebuild -checkFirstLaunchStatus` 曾回傳 69（沒有說明文字），但 SDK 查詢與實際 Swift 編譯可執行。
若日後 Xcode 要求補安裝元件，開啟「應用程式 → Xcode」，依畫面完成授權及必要元件安裝；不需要填入原作者的 Apple 帳號。

不需要原作者的 Developer ID、Sparkle 私鑰或 App Store Connect 憑證；本次不是公開發佈，不執行 `Scripts/release.sh`。

## 上游更新與驗證流程

以下命令均在 HansBar 專案根目錄執行。

1. 先看 `git status`。若有修改，先檢查內容並在 `hansbar` 提交要保留的檔案；不要盲目 `git add .`，以免加入機密。必要時用 `git stash push -u -m "before upstream update"` 暫存，並確認暫存成功；暫存不是遠端備份。
2. 取得上游並只以前進方式更新原版分支：

   ```bash
   git fetch upstream
   git switch main
   git merge --ff-only upstream/main
   git push origin main
   ```

   若 `--ff-only` 失敗，表示歷史需要檢查；停止，不用 reset 或 force 解決。

3. 將原版更新合併到個人分支，保留客製提交歷史：

   ```bash
   git switch hansbar
   git merge main
   ```

   若發生衝突，逐一比較原版與客製需求後解決並提交；不確定時請協助，不要整批選「全部用原版」。需要取消尚未完成的合併時，先確認合併前工作已保存，再用 `git merge --abort`。
   若使用了 stash，以 `git stash apply` 恢復，確認內容及提交後再刪除對應暫存；不要直接丟棄。

4. 重新閱讀當時版本的開發及打包文件，執行完整檢查、測試及隔離打包：

   ```bash
   # 編譯、測試與打包都在同步資料夾外的既有建置工作目錄進行。
   # 先確認該工作目錄沒有修改；有修改時停止並保存，不要覆蓋。
   git -C /Users/hanssun/Developer/HansBar-original-build status --short
   git -C /Users/hanssun/Developer/HansBar-original-build switch --detach hansbar
   (cd /Users/hanssun/Developer/HansBar-original-build && PATH="/opt/homebrew/opt/python@3.14/libexec/bin:$PATH" CODEXBAR_ALLOW_TEST_KEYCHAIN_ACCESS=0 CODEXBAR_SUPPRESS_TEST_KEYCHAIN_ACCESS=1 make check)
   (cd /Users/hanssun/Developer/HansBar-original-build && PATH="/opt/homebrew/opt/python@3.14/libexec/bin:$PATH" CODEXBAR_ALLOW_TEST_KEYCHAIN_ACCESS=0 CODEXBAR_SUPPRESS_TEST_KEYCHAIN_ACCESS=1 make test)
   (cd /Users/hanssun/Developer/HansBar-original-build && CODEXBAR_SIGNING=adhoc ./Scripts/package_app.sh)
   ```

   路徑適用本機 Apple Silicon Homebrew；其他電腦請用 `brew --prefix python@3.14` 找安裝位置。
   不使用 `make test-live`，不為了測試登入真實帳號或匯入瀏覽器 Cookie。
   建置工作目錄是同一個 Git 版本庫的額外工作目錄，不是另一條客製分支；不要在 detached HEAD 上開發。
   每次切換前先確認乾淨，建置產物留在原工作目錄，不能把 `.build` 複製到其他工作目錄。

5. 記錄實際結果；失敗時先釐清，不能把「編譯過了」當作所有測試成功。驗證後只推送自己的分支：

   ```bash
   git push origin hansbar
   ```

相關原版文件：[README](../README.md)、[開發指南](DEVELOPMENT.md)、[打包](packaging.md)、[正式發佈](RELEASING.md)。

## 已驗證的建置與隔離啟動

原始 Google Drive 目錄的打包及測試都曾遇到
`resource fork, Finder information, or similar detritus not allowed` 簽署錯誤。
相同程式碼移到同步資料夾外的 worktree 後，完整打包及啟動檢查成功。
這支持使用非同步目錄建置；沒有為此修改原版程式碼或停用簽署驗證。

成功建置的原版基準：`444821be54e2571dd115ca07b2ce3d1e0a71a45c`。
主工作目錄仍在 `hansbar`；額外建置 worktree 固定在此原版提交（detached HEAD）。
工具建立 worktree 時連同 `.build` 複製了過去，因此本次明確清除該工作目錄的快取並重新編譯，沒有依賴跨目錄的已編譯產物。

本次成功執行：

```bash
cd /Users/hanssun/Developer/HansBar-original-build
CODEXBAR_SIGNING=adhoc CODEXBAR_FORCE_CLEAN=1 ./Scripts/package_app.sh
```

`CODEXBAR_FORCE_CLEAN=1` 只用於這次已確認的複製快取問題；平常移除此參數，保留增量建置。
產物：`/Users/hanssun/Developer/HansBar-original-build/CodexBar.app`。
打包流程已驗證 App 與內嵌 Sparkle 簽署、App 與 CLI 資源載入，以及 App 隔離啟動後存活 6 秒。

要再次安全開啟並驗證，不必重新建置：

```bash
cd /Users/hanssun/Developer/HansBar-original-build
./Scripts/verify_packaged_app_launch.sh "$PWD/CodexBar.app"
```

這就是成功打包時實際執行的驗證腳本。看到
`Launch smoke check OK: packaged app survived 6s without the build checkout.`
代表通過。它會自行關閉測試 App，因此測試結束後沒有留在選單列是正常的。
未進行真實帳號查詢、選單點擊或截圖驗證。

如果只是想在 Finder 找到檔案：按 `Command + Shift + G`，貼上
`/Users/hanssun/Developer/HansBar-original-build`。
暫時不要雙擊 App、不要拖到「應用程式」，以免共用既有 CodexBar 狀態。
一般啟動指令 `open -n /Users/hanssun/Developer/HansBar-original-build/CodexBar.app`
**不是隔離啟動，本次未執行**。若要長期操作 UI，先另外規劃獨立 macOS 測試帳號／VM，
或在下一階段明確設計 HansBar 的 App 身份、資料儲存、登入來源與更新來源隔離。

## 初始化結果、問題與下一步

### 已完成

- `origin` 保留為 `hansidea/HansBar`，新增正確的 `upstream`，兩個遠端均成功 fetch。
- 保留原版 `main`，建立 `hansbar` 並成功 `git push --set-upstream origin hansbar`；沒有向 upstream 推送，也不需要重新 GitHub 登入。
- 新增本文件是這次唯一的客製版本控制內容；不提交 `.build` 或 App 產物。
- 安裝 Python 3.14.8；`make check` 通過，SwiftLint 掃描 2945 個檔案、0 違規。
- 非同步目錄中的原版 release 打包、App／Sparkle 簽署驗證、CLI／App 資源檢查及 6 秒隔離啟動通過。
- 沒有執行會全面停止 CodexBar 的啟動腳本，沒有安裝或覆蓋 `/Applications/CodexBar.app`。

### 尚未通過的完整測試

在 `/Users/hanssun/Developer/HansBar-original-build` 執行：

```bash
PATH="/opt/homebrew/opt/python@3.14/libexec/bin:$PATH" CODEXBAR_ALLOW_TEST_KEYCHAIN_ACCESS=0 CODEXBAR_SUPPRESS_TEST_KEYCHAIN_ACCESS=1 make test
```

結果：發現 1654 個測試選擇項，分成 150 組。116 組成功，1 組失敗；腳本自動整組重試一次仍失敗後停止，剩餘 33 組未執行，沒有逾時。不能宣稱完整測試通過。

失敗案例：

- Suite：`ProviderSwitcherShortcutsTests`
- 測試：`shifted digit shortcuts match the unmodified key`
- 位置：`Tests/CodexBarTests/PreferencesDocumentTests.swift:251`
- 條件：合成的 Shift＋Command＋2 鍵盤事件未得到預期的供應商索引 `1`。

這是原版基準在本機環境的實測失敗，原因未確定；沒有修改斷言、刪除測試或更改原版功能來規避。
原版編譯另有 C 整數轉換及 Swift 棄用等警告，但未阻擋 release 打包。

### 下一步

1. 在開始客製功能前，另行調查上述快捷鍵測試在本機 macOS／鍵盤環境的失敗原因，修復或確認上游修正後重跑完整測試；保留目前失敗紀錄。
2. 正常 UI 操作前先規劃隔離環境。本次驗證不需要手動登入 GitHub、Apple 或服務帳號，也不需要授權 Keychain。
3. 若 Xcode 顯示初次啟動元件提示，依前述步驟完成；實際打包已可用，不需為此重新安裝 Xcode。
4. 所有客製修改只在主工作目錄的 `hansbar` 上進行；保留 `main` 作為原版承接線。日後同步依本文件流程 merge，不用 hard reset 或 force push。

