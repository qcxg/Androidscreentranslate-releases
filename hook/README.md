# Screen Translator Hook

獨立的 LSPosed 字幕模組，套件為 `com.shiraka.subtitlemodule`，使用 libxposed API／service 102.0.0。本文對應 1.0.6（10006），核對日期為 2026-10-30；安裝版本以 `build.gradle.kts` 為準。第三方 runtime 規則自 1.0.6 起支援。

模組觀察目標 App 已產生的字幕並傳送給主程式，本身不翻譯、不擷取螢幕影像、不建立浮窗。主程式與模組須使用相同簽章，詳見[安全邊界](../docs/security.md)。

## 內建適配

| App | 套件 | Hook 點 |
| --- | --- | --- |
| YouTube | `com.google.android.youtube` | `SubtitleWindowView.g(cue)`，Media3 `SubtitleView.setCues(List<Cue>)` 備援 |
| NHK+ | `nhk.app.tep` | `BasePlayerHandle.putOutsideMomentarySubtitles(MomentarySubtitles)` |

可執行位置以 `YoutubeSubtitleAdapter.java`、`NhkPlusSubtitleAdapter.java` 為準；主程式顯示用目錄由 `HookCatalogDefaults` 定義。目標 App 更新可能改變混淆名稱及字幕資料結構，不能把舊參考版本當成所有新版的相容保證。

Hook 目錄 v2 可匯入 runtime 規則，由 `DeclarativeHookAdapter` 指定方法 overload，從參數／回傳值／receiver 的欄位路徑提取文字；v1 目錄仍只作描述。接口不載入第三方 bytecode。規則經同簽章 `HookStatusActivity` 同步至框架 Remote Preferences；目標程序啟動時讀取並安裝，修改後需重啟目標 App。格式、範例及限制見[適配規格](../docs/adapters/README.md)。

## 安裝與作用域

一般使用者下載公開 Release 的主程式與模組；本機 Debug 測試則應成對建置及安裝：

```powershell
./gradlew.bat :app:assembleDebug :hook:assembleDebug
adb install -r app/build/outputs/apk/debug/app-debug.apk
adb install -r hook/build/outputs/apk/debug/hook-debug.apk
```

在相容框架中啟用模組，設定作用域並重新啟動目標 App。模組沒有 launcher 圖示，由主程式設定及 Hook 管理頁提供入口。

主程式將「安裝」與「框架作用域」分開處理：

- 未安裝時，首頁隱藏 Hook 模式，設定頁提供安裝入口。
- 已安裝但框架作用域與已啟用適配套件沒有交集時，首頁模式不可選；有交集時可選，不另以模組 enabled API 判定。
- 設定頁只顯示已安裝及排查框架／目標 App 啟用狀態的提示；Hook 頁顯示作用域數量。
- 各適配開關獨立保存選擇，開啟時透過 `requestScope()` 請求目標授權；同套件最後一個適配停用時才以 `removeScope()` 移除作用域。`getScope()` 回報的授權狀態另外顯示，不覆寫各適配開關。
- 自訂規則匯入後先停用，啟用時請求目標作用域並同步規則；停用／移除後，接收端即拒絕該適配事件。既有注入程序要完整重啟才會移除舊掛載點。
- 模組使用 `staticScope=false`，允許使用者授權自訂目標。自訂規則需要框架 Remote Preferences；同步失敗時可在 Hook 管理重試。

`HookPluginRepository` 透過無介面的 `HookStatusActivity` 送出 request ID。`FrameworkStatusBridge` 接收 libxposed service 回呼並處理作用域請求；框架可能要求確認或拒絕變更，主程式以回報作用域為準。無回報或 timeout 時作用域視為空，不把「已安裝」當成已生效。

## 字幕協議

兩個模組的 `ISubtitleBridge.aidl` 必須一致；`submitEvent(Bundle)` 為單向 Binder。事件常數由主程式 `HookBridgeContract` 與模組 `BridgeContract` 對應維護。

字幕事件類型為 `hook_ready`、`source_active`、`source_inactive`、`subtitle`、`subtitle_cleared`。Payload 含協議版本、來源套件／session／版本、adapter、sequence、時間、字幕及可選播放位置。

Binder 以系統 calling UID 核對支援套件；Android 14+ 綁定不可用時使用顯式廣播，以系統 sender UID 或目前工作階段令牌驗證。令牌動態接收器要求發送者持有 `SUBTITLE_BRIDGE` 簽章權限。框架狀態回報另外要求同簽章權限及對應 request ID。

`CueBridgeClient` 以最多 64 個待傳事件的 FIFO 處理斷線與重試，滿時移除最舊項目。接收端限制長度、session 數與 sequence，擷取服務篩除過舊與錯誤工作階段事件。Debug 記錄不包含字幕原文。

Android 8–13 沒有上述廣播備援，宿主若受套件可見性影響而無法綁定 Binder，Hook 可能無法連線。主程式事件隊列與計時規則見[架構文件](../docs/architecture.md)，Release 簽署與發佈見[開發文件](../docs/development.md)。
