# Screen Translator 螢幕翻譯

## 專案介紹、使用說明與第三方適配指南 / Project, usage, and third-party adapter guide

Screen Translator 是 Android 即時螢幕翻譯工具。它從本機 OCR、Android 無障礙節點或 LSPosed Hook 取得文字，交給使用者選擇的翻譯服務，再以懸浮字幕呈現。一般使用可只安裝主程式；Hook 字幕另需安裝配套模組與相容框架。

Screen Translator is an Android tool for translating text while using other apps. It obtains text through on-device OCR, Android accessibility nodes, or LSPosed hooks, translates it with services chosen by the user, and displays a subtitle overlay. The main app works on its own; Hook subtitles additionally require the matching module and a compatible framework.

本指南對應 1.0.6，同時供使用者與適配作者閱讀。第三方規則接口自 1.0.6 起支援；1.0.5 及之前的 Hook 匯入目錄只記錄描述。請成對更新主程式與 Hook。

This guide covers 1.0.6 and is for users and adapter authors. Third-party runtime rules are supported starting with 1.0.6; Hook catalogs in 1.0.5 and earlier only stored descriptions. Update the main app and Hook module together.

### 官方入口 / Official links

- [最新發佈與版本說明 / Latest release and changes](https://github.com/qcxg/Androidscreentranslate-releases/releases/latest)
- [主程式 APK / Main app APK](https://github.com/qcxg/Androidscreentranslate-releases/releases/latest/download/screen-translator.apk)
- [Hook 模組 APK / Hook module APK](https://github.com/qcxg/Androidscreentranslate-releases/releases/latest/download/screen-translator-hook.apk)
- [APK SHA-256 校驗檔 / APK SHA-256 checksums](https://github.com/qcxg/Androidscreentranslate-releases/releases/latest/download/SHA256SUMS.txt)
- [完整適配規格 / Full adapter specification](https://github.com/qcxg/Androidscreentranslate-releases/releases/latest/download/Adapter-Spec.zh-en.md)
- [適配範例壓縮檔 / Adapter example files](https://github.com/qcxg/Androidscreentranslate-releases/releases/latest/download/adapter-examples.zip)
- [官方網站 / Website](https://shiraka.icu)

公開倉庫只放發佈產物與說明。私人源碼倉庫不作為 App 更新檢查或一般下載的來源。

The public repository contains release artifacts and documentation. Update checks and user downloads use this public release repository.

### 適配文件 / Adapter documentation

- [第三方適配規格與範例 / Adapter specification and templates](https://github.com/qcxg/Androidscreentranslate-releases/blob/main/docs/adapters/README.md)

## 1. 主要功能 / Main features

| 功能 / Feature | 說明 / Description |
| --- | --- |
| 指定區域與全螢幕 OCR / Region and fullscreen OCR | 在裝置上辨識畫面文字，適合沒有原生字幕介面的內容。Recognizes screen text on device when no usable native subtitle source exists. |
| 無障礙字幕 / Accessibility subtitles | 讀取目標 App 公開的可見字幕節點；內建 YouTube，可匯入其他 App 的規則。Reads exposed caption nodes; includes YouTube and supports imported profiles. |
| Hook 字幕 / Hook subtitles | 接收播放器交付的字幕；內建 YouTube、NHK+，可匯入方法與欄位路徑規則。Receives player captions; includes YouTube/NHK+ and supports declarative method/field rules. |
| 多來源翻譯 / Multiple translation services | 可同時選用多個來源，也可設定私人 API 或 OpenAI 相容端點。Use multiple services or configure a private/OpenAI-compatible endpoint. |
| 字幕浮窗 / Subtitle overlay | 原文、雙語或僅譯文；可調整位置、字體大小、外觀及來源顏色。Original, bilingual, or translated text with configurable placement, size, style, and source colors. |
| 事件字幕保留 / Event subtitle history | Hook 與無障礙模式可保留多條字幕，並依設定自動隱藏。Keep multiple event captions and optionally hide them after a configured duration. |
| 字幕錄製 / Subtitle recording | 儲存啟用錄製後的字幕時間軸，回看及匯出原文與各來源譯文。Record, review, and export subtitle timelines and translations. |
| 設定備份 / Configuration backup | 選擇備份分區，支援密碼加密與還原預覽。Choose backup sections, optionally encrypt them, and preview restoration. |
| 版本檢查 / Update checks | 關於頁查詢公開 Release；Hook 與主程式版本不同時提供可略過的提示。About checks public releases; a dismissible notice reports mismatched app/module versions. |

介面提供繁體中文、簡體中文與英文。OCR 模型支援 Latin、中文、日文及韓文文字；可用翻譯語言及服務能力依所選 Provider 而異。辨識、翻譯與字幕適配是三個不同環節，新增適配不會改變翻譯服務的語言能力。

The UI supports Traditional Chinese, Simplified Chinese, and English. OCR models cover Latin, Chinese, Japanese, and Korean scripts. Translation languages depend on the selected provider. Recognition, subtitle adapters, and translation are separate stages; a new adapter does not add languages to a provider.

## 2. 選擇文字來源 / Choose a text source

| 使用情況 / Situation | 建議路徑 / Suitable path |
| --- | --- |
| 字幕已公開於無障礙樹 / Captions appear in the accessibility tree | 無障礙；不需要每次螢幕擷取授權。Accessibility; no screen capture prompt is needed. |
| 可取得播放器的 Java 字幕回呼 / A Java player caption callback is available | Hook；需要框架、配套模組與作用域。Hook; requires the framework, module, and approved scope. |
| 遊戲、自繪介面、燒錄字幕或一般畫面文字 / Games, custom rendering, burned-in captions, general screen text | 指定區域或全螢幕 OCR。Region or fullscreen OCR. |

無障礙及 Hook 以新字幕事件觸發工作。沒有新字幕時不會啟動 OCR 作備援。來源無法讀取時，應先檢查適配或切換模式；沒有任何一個模式能保證讀取所有 App 的所有畫面。

Accessibility and Hook modes act on new caption events. They do not automatically fall back to OCR when captions are unavailable. Check the adapter or choose another mode if the source cannot be read. Compatibility depends on what the target app exposes.

## 3. 安裝與第一次使用 / Installation and first use

最低系統為 Android 8.0（API 26）。下載主程式 APK 後，依 Android 系統提示允許該下載來源安裝。更新時沿用正式 Release；主程式與 Hook 應使用同一發佈來源的配套版本。SHA256SUMS.txt 可用來比對下載檔案的 SHA-256。

Android 8.0 (API 26) or later is required. Download the main APK and follow Android's installation prompt for that download source. Use matching app/module releases from the same distribution. SHA256SUMS.txt lists APK hashes for checking download integrity.

1. 開啟主程式，在首頁選擇擷取模式、原文與目標語言，以及字幕輸出方式。Open the app and select the capture mode, source/target languages, and subtitle output.
2. 在翻譯服務頁啟用要使用的來源，並完成需要的 API 設定。Enable translation services and configure any required API credentials.
3. 授予懸浮窗權限。OCR 另需當次螢幕擷取授權與對應模型；無障礙需開啟字幕讀取服務。Grant overlay access. OCR also needs capture consent and the matching models; Accessibility needs the subtitle service enabled.
4. 開始工作階段，切換到目標 App 並開啟其原生字幕。Start a session, open the target app, and enable its native captions.
5. 使用浮動控制器暫停、恢復或停止；錄製需先在設定中啟用。Use the floating controller to pause/resume or stop. Enable recording in settings before recording a session.

無障礙支援頁可直接關閉字幕讀取服務；也能設定在使用結束後自動關閉。停止工作階段會結束文字處理，暫停與完全撤回系統授權是不同操作。

The Accessibility support page can turn off the subtitle service, and settings can disable it automatically after use. Stopping a session ends text processing; pausing a session and revoking a system permission are separate actions.

### Hook 額外安裝步驟 / Additional Hook setup

安裝 `screen-translator-hook.apk`，在相容的 LSPosed／libxposed API 102 框架中啟用模組。在 Hook 管理開啟目標 App，完成框架作用域授權，再重新啟動目標 App。僅安裝模組不代表已注入或已取得字幕。

Install `screen-translator-hook.apk` and enable it in a compatible LSPosed/libxposed API 102 framework. Enable the target app in Hook management, approve its scope, then restart the target app. Installation alone does not mean the module is injected or receiving captions.

自訂 Hook 規則另需框架支援 Remote Preferences。主程式與模組的簽章必須相同，否則規則同步與作用域通訊可能無法完成。一般使用者使用配套正式 APK 即可。

Custom Hook rules also require framework Remote Preferences. The app and module must share a signing certificate for configuration/scope communication. Use the matching official APK pair for normal installation.

## 4. 導入第三方適配 / Import a third-party adapter

第三方適配是一份描述目標包名與字幕提取方式的 JSON 檔。作者可在不取得私人源碼、不重編主程式的情況下製作並分享。使用者在 App 內手動匯入與啟用；規則本身不包含 APK、DEX 或 JavaScript。

An adapter is a JSON file describing a target package and how to obtain its captions. Authors can create and share it without the private source repository or a custom app build. Users import and activate it inside the app. Rules do not embed APKs, DEX, or JavaScript.

**無障礙：**首頁選無障礙模式，進入無障礙支援頁的字幕適配區，按匯入並選 JSON，確認條目後手動開啟。規則變更會立即更新服務訂閱的套件；開始字幕工作階段後才會處理節點文字。

**Accessibility:** Select Accessibility mode, open the subtitle adapter area in Accessibility support, import the JSON, and enable its entry. Changes immediately update the service's package filter. Node text is processed only during a subtitle session.

**Hook：**在 Hook 管理匯入 JSON，開啟條目並完成框架授權。確認「規則已同步、作用域已授權」後完整關閉並重開目標 App。更新 JSON 後也需要重新啟動目標 App；重新匯入同 ID 會替換自訂規則並先停用。

**Hook:** Import the JSON in Hook management, enable the entry, and approve the framework request. Once rules are synced and scope is approved, fully close and reopen the target app. Restart after rule updates as well. Re-importing a custom ID replaces it and leaves it disabled until activated again.

無障礙頁可匯出全部或單項規則，也可移除自訂規則。Hook 管理提供選取、單項／多項匯出與移除。內建適配可啟停；匯入檔無法覆蓋其實作。適配目錄需要另外匯出，通用設定備份目前不包含這兩份目錄。

同一 App 若同時有內建與第三方 Hook 適配，可分別開關；使用自訂規則取代內建來源時，停用內建條目即可，其他已啟用適配仍共用該套件作用域。

Accessibility adapters can be exported individually or together, and custom entries can be removed. Hook management also supports selection and multi-entry export/removal. Built-in adapters can be toggled; imports cannot replace their implementation. Export adapter catalogs separately; general configuration backups currently do not include them.

Built-in and third-party Hook adapters for the same app have independent switches. Disable the built-in entry when replacing its source with a custom rule; other enabled adapters retain the shared package scope.

## 5. 製作無障礙適配 / Create an accessibility adapter

先在目標 App 顯示字幕，透過 Android UI Automator 或 Layout Inspector 觀察節點。需要確認實際包名、字幕節點 resource ID、class，以及文字位於 `text` 或 `content-desc`。例如：

Show captions in the target app and inspect its nodes with Android UI Automator or Layout Inspector. Identify the actual package, caption resource ID, node class, and whether text appears in `text` or `content-desc`. For example:

```json
{
  "format": "screen-translator.accessibility-adapters",
  "formatVersion": 1,
  "entries": [{
    "id": "sample-caption-view",
    "displayName": "Sample Player",
    "packageName": "org.example.player",
    "selectors": [{
      "viewId": "org.example.player:id/subtitle",
      "className": "android.widget.TextView"
    }],
    "textSource": "text",
    "includeDescendants": true,
    "stabilityMs": 280,
    "clearDelayMs": 900
  }]
}
```

這是模板，需要改成目標 App 的真實值。同一 selector 內是 AND，selectors 陣列各項是 OR；字串比較區分大小寫。可使用 `viewIdContains` 或 `classNameContains` 作包含比對。應選字幕專用 ID，避免把所有 TextView 或整個播放器容器都當成字幕。

This is a template: replace the package and selectors with real values. Conditions in a selector are ANDed; array entries are ORed. Comparisons are case-sensitive. `viewIdContains` and `classNameContains` provide substring matching. Prefer caption-specific IDs instead of matching every TextView or an entire player container.

`includeDescendants: true` 會讀取命中節點的子樹，適合字幕容器；false 只讀本身。`textSource` 可選 text、contentDescription、both；both 優先使用文字，無文字時才用描述。多行字幕會依位置合併並去重。

`includeDescendants: true` reads a matching node's subtree and suits caption containers; false reads only the node. `textSource` accepts text, contentDescription, or both. Both prefers text, then uses the description if text is absent. Multiline captions are combined in screen order and deduplicated.

如果字幕完全沒有出現在節點樹，調整 selector 無法讓它憑空出現。這時應調查 Hook 交付點，或使用 OCR。

If captions never appear in the node tree, a selector cannot expose them. Investigate a Hook caption callback or use OCR.

## 6. 製作 Hook 適配 / Create a Hook adapter

調查目標 App 的 Java 字幕交付方法，記錄宣告類別、方法名、精確參數型別與字幕欄位。適合的掛載點包括接收文字的字幕回呼、接收 Cue 列表的 renderer，或回傳字幕物件的方法。以版本固定且只交付字幕的點為佳。

Identify a Java caption delivery method in the target app. Record its declaring class, method name, exact parameter types, and text fields. Suitable points include a caption callback, renderer receiving a cue list, or method returning caption data. Prefer a stable callback dedicated to captions.

Media3 常見形式是 `SubtitleView.setCues(java.util.List)`，其中各 Cue 的文字位於 `text`。以下模板使用參數 0 作來源，對集合中的每個元素讀取 text 欄位：

A common Media3 shape is `SubtitleView.setCues(java.util.List)`, with caption text stored in each Cue's `text` field. This template reads argument 0 and extracts text from each collection element:

```json
{
  "format": "screen-translator.hook-catalog",
  "formatVersion": 2,
  "entries": [{
    "id": "sample-media3",
    "displayName": "Sample Media3 Player",
    "packageName": "org.example.player",
    "adapterId": "sample-media3-v1",
    "adapterVersion": 1,
    "apiVersion": 102,
    "hookPoints": [{
      "className": "androidx.media3.ui.SubtitleView",
      "methodName": "setCues",
      "strategy": "after",
      "runtime": {
        "parameterTypes": ["java.util.List"],
        "source": "argument",
        "argumentIndex": 0,
        "fieldPath": ["text"],
        "clearOnNull": true
      }
    }]
  }]
}
```

包名與方法需以實際 App 為準，並非所有使用 Media3 的 App 都經過這個 View。直接接收 String 的回呼可把 parameterTypes 設為 `["java.lang.String"]` 並使用空 fieldPath；要讀原方法回傳值可使用 `source: "return"`。參數型別依 JVM 完整名稱填寫，泛型使用擦除後型別。

Verify the real package and method; some Media3 apps do not use this View. A String callback can use `["java.lang.String"]` and an empty fieldPath. Select `source: "return"` to read the original return value. Parameter types use full JVM names with generic types erased.

掛載採 after：原方法先正常執行，成功回傳後提取字幕；不修改其參數或回傳值。fieldPath 支援欄位與集合／陣列遍歷，例如 `["cues", "text"]`。最終值需為文字；找不到欄位或型別不符時不發送字幕。所有配置點都會掛載，請避免同時配置多個重複字幕來源。

Hooks run after the original method returns successfully, preserving its arguments and result. Field paths traverse fields, collections, and arrays, such as `["cues", "text"]`. Final values must be text. A missing field or wrong type skips that event. Every configured point is installed, so avoid adding duplicate caption sources.

預設只安裝於目標 App 主程序。子程序使用 processName 指定完整程序名。若類別只存在後載入 split／自訂 class loader，或字幕完全在 native／圖像層，這個規則接口可能無法處理；需要挑選可用的交付點或使用其他模式。

Rules default to the app's main process. Use processName for an exact secondary process. Classes loaded later through splits/custom loaders, or captions confined to native/image rendering, may not be reachable through this interface. Choose an available delivery point or another capture mode.

## 7. 驗證與分享 / Test and share

匯入成功表示格式合法，仍需在目標 App 實測。至少測試完整的兩行字幕、逐字更新、字幕消失、切換影片、切換 App、暫停／停止以及關閉規則。分享規則時附上目標 App versionName／versionCode、Android 版本、框架版本、主程式與 Hook 版本、測試場景及限制。

A successful import verifies the format, while compatibility still requires real app testing. Test multiline captions, incremental updates, disappearance, video/app switching, pause/stop, and disabling the rule. Share the target versionName/versionCode, Android/framework versions, main app/Hook versions, tested scenarios, and known limitations with the adapter.

保留穩定 id，更新內容時遞增 adapterVersion，並在 description／notes 說明適用版本。避免使用內建保留 ID。先匯出一份可用規則，便於測試後還原。

Keep a stable ID, increment adapterVersion for Hook revisions, and describe supported versions in description/notes. Avoid reserved built-in IDs. Export a working rule before experimenting so it can be restored.

## 8. 常見問題 / Troubleshooting

| 現象 / Symptom | 檢查方式 / Checks |
| --- | --- |
| 匯入失敗 / Import fails | 確認匯入頁、format／formatVersion、必要欄位、重複 ID 及大小限制。Check the correct import page, format/version, required fields, duplicate IDs, and size limits. |
| 舊 Hook 條目無法開啟 / Legacy Hook entry cannot be enabled | v1 僅含描述；需建立帶 runtime 的 v2 規則。v1 is descriptive; create a v2 rule containing runtime. |
| Hook 未同步或作用域未授權 / Hook rules unsynced or scope missing | 確認配套版本、同簽章、框架啟用與 Remote Preferences 支援，再按同步並完成授權。Check versions, signing, framework/remote preferences support, then sync and approve scope. |
| Hook 已同步仍無字幕 / Hook synced, no captions | 完整重啟目標 App；核對包名、程序、宣告類別、方法 overload 與字幕欄位。Restart the target and verify the package, process, declaring class, overload, and text fields. |
| 無障礙沒有文字 / Accessibility yields no text | 確認規則已啟用、服務授權、工作階段啟動，且實際字幕出現在節點樹。Check activation, service permission, active session, and real caption nodes. |
| 翻譯到按鈕或標題 / Buttons/titles are translated | 收窄 viewId／類別條件或關閉 includeDescendants。Narrow the selector or turn off descendant extraction. |
| 出現重複或來源跳動 / Duplicate captions or source switching | 檢查是否同 App 啟用了多個相同來源，或配置了重複 Hook 點。Check overlapping enabled profiles or duplicate Hook points. |
| 更新後適配失效 / Adapter breaks after an app update | 重新觀察節點／字幕交付點；混淆名、類別及 UI 樹可能已改變。Reinspect the nodes/callbacks; obfuscated names, classes, and UI trees can change. |
| 有原文但沒有譯文 / Original text appears without translation | 檢查翻譯服務、網路、憑證及語言支援。Check the translation provider, network, credentials, and language support. |

## 9. 資料與權限 / Data and permissions

OCR 圖像在裝置上辨識。啟用翻譯後，提取出的文字會送往使用者啟用的翻譯服務。啟用錄製時，原文與譯文會保存在 App 私有儲存，匯出後則形成使用者選定的檔案。第三方規則決定被選中的節點或字幕欄位，請確認選取範圍符合預期。

OCR images are recognized on device. When translation is enabled, extracted text is sent to the selected translation services. Recording stores original/translated text in app-private storage; exports create files at a user-selected location. Third-party rules determine which nodes or fields are selected, so review that selection.

無障礙服務只在字幕工作階段處理已啟用規則的套件。Hook 事件仍核對作業系統 UID 或來源專屬的工作階段令牌，並檢查啟用狀態、適配 ID、序號及文字上限。框架與目標程序屬於 Hook 的信任範圍，JSON 的 checksum 只檢查內容一致性，不代表作者認證。

Accessibility processes enabled packages only during a subtitle session. Hook events still require an OS-verified UID or a source-specific session token and are checked for adapter activation, ID, sequence, and text limits. The framework and target process are part of the Hook trust boundary. A JSON checksum checks consistency rather than authenticating its author.

## 10. 發佈檔案與相容界線 / Release files and compatibility

正式發佈提供主程式、選用 Hook 模組、APK 校驗檔、本指南、完整適配規格及範例壓縮檔。公開倉庫同步更新使用指南、適配文件與版本說明。請先閱讀該版本的變更說明；一般適配與 App／框架更新的相容性需由實際測試確認。範例檔用於示範格式，沒有聲稱適配所有播放器。

Releases include the main app, optional Hook module, APK checksums, this guide, the adapter specification, and example files. The public repository is updated with user guides, adapter documentation, and release notes. Read each release's changes. Compatibility with a particular app/framework update requires testing; templates illustrate the format and do not claim support for every player.

若回報問題，請提供重現步驟與版本組合。診斷資料應遮蔽私人字幕、API Key 與其他個人資訊。適配格式的欄位、限制與向後相容規則詳見隨發佈附帶的 Adapter-Spec.zh-en.md。

For issue reports, include reproduction steps and the exact version combination. Remove private captions, API keys, and personal information from diagnostics. Adapter-Spec.zh-en.md, attached to the release, defines the complete fields, limits, and compatibility rules.

Copyright © Shiraka. All rights reserved.
