# 第三方字幕適配規格 / Third-party subtitle adapter specification

這份規格對應 1.0.6 的適配接口，核對日期為 2026-10-30。1.0.5 及之前只有 Hook 描述目錄，不能執行下列 runtime 規則。使用時需安裝版本相同且同簽章的主程式與 Hook 模組。

This specification covers the adapter interfaces in 1.0.6, checked on 2026-10-30. Versions 1.0.5 and earlier only have a descriptive Hook catalog and cannot execute the runtime rules below. Install a matching main app and signed Hook module.

## 1. 交付與操作 / Distribution and operation

第三方作者交付 UTF-8 JSON。無障礙規則從「無障礙支援 → 字幕適配」匯入；Hook 規則從「Hook 管理」匯入。匯入及同 ID 更新後均先停用，需手動開啟。每次匯入會完整校驗，任何條目不合法都不寫入。內建條目不能被匯入檔覆蓋或刪除；同 ID 的自訂條目會被整筆替換。

Authors distribute UTF-8 JSON. Import accessibility rules under **Accessibility support → Subtitle adapters**, and Hook rules under **Hook management**. Imported and updated adapters start disabled. Validation finishes before storage changes. A failed entry rejects the entire import. Built-in entries cannot be replaced or deleted; matching custom IDs are replaced as complete entries.

- [無障礙範例 / Accessibility template](examples/accessibility-player.json)
- [Media3 Hook 範例 / Media3 Hook template](examples/hook-media3.json)
- [文字回呼 Hook 範例 / Text callback Hook template](examples/hook-text.json)

Release 附件中的本規格可搭配 [adapter-examples.zip](https://github.com/qcxg/Androidscreentranslate-releases/releases/latest/download/adapter-examples.zip) 使用；解壓後取得上述三份 JSON。When reading this document as a release attachment, download that ZIP to obtain the three JSON templates.

範例使用虛構套件 `org.example.player`，需要改成實際包名；它們不是已驗證的 App 適配。啟停／移除無障礙規則會立即更新服務；Hook 新增或修改掛載點需要重新啟動目標 App。停用或移除 Hook 條目後，主程式即停止接受該適配的事件。

同一 App 的內建與自訂 Hook 適配可獨立啟停；如果要以自訂規則取代內建來源，可停用內建條目並保留自訂條目。套件作用域會供該套件仍啟用的適配共用。

Templates use the fictional package `org.example.player`. Replace it with the real package; these files do not claim compatibility with a particular app. Accessibility changes update the service immediately. Restart the target app after adding or changing Hook points. Disabling/removing a Hook adapter immediately stops acceptance of its events by the main app.

Built-in and custom Hook adapters for the same app can be toggled independently. Disable the built-in entry when replacing its source with custom rules. Package scope remains shared by the enabled adapters for that package.

## 2. 無障礙格式 / Accessibility format v1

```json
{
  "format": "screen-translator.accessibility-adapters",
  "formatVersion": 1,
  "entries": [{
    "id": "my-player",
    "displayName": "My Player",
    "packageName": "org.example.player",
    "description": "Visible caption node",
    "selectors": [{"viewId": "org.example.player:id/subtitle"}],
    "textSource": "text",
    "includeDescendants": true,
    "stabilityMs": 280,
    "clearDelayMs": 900
  }]
}
```

| 欄位 / Field | 規格 / Contract |
| --- | --- |
| `id` | 必填；`[a-z0-9][a-z0-9._-]{0,63}`，同一目錄中唯一。Required, unique within the catalog. |
| `displayName` | 必填；1–128 字元。Required, 1–128 characters. |
| `packageName` | 必填；完整 Android 包名，限定事件來源。Required Android package; scopes event processing. |
| `description` | 選填；最多 512 字元。Optional, up to 512 characters. |
| `selectors` | 必填；1–16 個條件物件。Required, 1–16 condition objects. |
| `textSource` | `text`（預設）、`contentDescription` 或 `both`。`both` 優先文字，無文字時以描述作備援。`both` prefers text and falls back to the description when no text is available. |
| `includeDescendants` | 預設 `true`；合併符合節點下的可見子節點。`false` 只讀命中節點本身。Defaults to true; false reads only the matching node. |
| `stabilityMs` | 0–2000，預設 280；字幕穩定等待。Caption stabilization delay. |
| `clearDelayMs` | 100–10000，預設 900；確認沒有匹配字幕後的清空延遲。Clear delay after a scan finds no matching caption. |

每個 selector 至少包含以下一項；值必須為 1–256 字元、非全空白。字串比較區分大小寫，不使用正則表達式。

Each selector must contain at least one condition below. Values are nonblank strings of 1–256 characters. Comparisons are case-sensitive and do not use regular expressions.

| 條件 / Condition | 意義 / Meaning |
| --- | --- |
| `viewId` | 節點完整 resource ID 精確相等，例如 `org.example.player:id/subtitle`。Exact full resource ID. |
| `viewIdContains` | resource ID 包含字串。Substring of the resource ID. |
| `className` | 節點類別精確相等，例如 `android.widget.TextView`。Exact node class name. |
| `classNameContains` | 節點類別包含字串。Substring of the node class name. |

同一 selector 的條件全部需要成立（AND）；不同 selector 任一成立即可（OR）。例如 `{ "viewIdContains": "subtitle", "className": "android.widget.TextView" }` 要同時符合 ID 和類別。多個命中字幕按畫面位置合併，保留不同文字並去除父子重複表示。同套件有多個啟用規則時合併其結果，穩定／清空延遲各取最大值。

Conditions within one selector are ANDed; selectors are ORed. The example above requires both the ID substring and class match. Captions are combined in screen order, with duplicate parent/child representations removed. Multiple enabled profiles for a package combine their text and use the maximum configured stabilization/clear delays.

取得節點資訊時，可在目標 App 顯示字幕的情況下使用 Android UI Automator dump 或 Layout Inspector，記錄包名、resource ID、class、字幕是否位於 `text` 或 `content-desc`。只用 `android.widget.TextView` 會匹配太多一般介面文字，應加上字幕容器或 ID 條件。

Use an Android UI Automator dump or Layout Inspector while captions are visible to identify the package, resource ID, class, and whether captions appear as `text` or `content-desc`. A class-only `android.widget.TextView` rule often selects ordinary UI labels; add a caption-specific ID condition.

節點掃描每個規則最多訪問 512 個節點、保留 16 個候選、每個字幕子樹 64 個節點，每個節點最多 64 個子節點。最終文字最多 2000 字元。未公開於無障礙樹、受保護視窗或只有像素的字幕無法用 selector 取得。

Each rule scans at most 512 nodes, keeps 16 candidates, and snapshots at most 64 nodes per caption subtree with at most 64 children per node. Combined text is capped at 2000 characters. Selectors cannot retrieve captions absent from the accessibility tree, protected windows, or pixels alone.

## 3. Hook 格式 / Hook catalog v2

```json
{
  "format": "screen-translator.hook-catalog",
  "formatVersion": 2,
  "entries": [{
    "id": "my-media3",
    "displayName": "My Media3 Player",
    "packageName": "org.example.player",
    "adapterId": "my-media3-v1",
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

| 條目欄位 / Entry field | 規格 / Contract |
| --- | --- |
| `id`, `displayName`, `packageName`, `description` | ID、名稱、包名規則與無障礙格式相同。Same identity rules as the accessibility format. |
| `adapterId` | 必填；`[A-Za-z0-9][A-Za-z0-9._-]{0,127}`，識別輸出事件。Required event adapter identifier. |
| `adapterVersion` | 必填；1–1000，由作者遞增。Required author-managed revision. |
| `apiVersion` | runtime 規則固定為 `102`。Runtime rules require 102. |
| `processName` | 選填；最多 128 字元。省略時只安裝在與包名相同的主程序；例如 `org.example.player:player` 可選子程序。Optional exact process name; defaults to the package main process. |
| `hookPoints` | 最多 32 個；每個包含 runtime 的點都會獨立安裝，不會自動選取備援。Up to 32 points; all runtime points are installed independently, without automatic fallback selection. |
| `notes` | 選填；最多 2048 字元，記錄目標 App 版本與調查結果。Optional compatibility notes. |

| 掛載點欄位 / Hook point field | 規格 / Contract |
| --- | --- |
| `className` | 宣告方法的 Java binary name；內部類別使用 `$`。Exact declaring class; use `$` for nested classes. |
| `methodName` | 精確方法名；依參數型別解析 overload。Exact method name, resolved with the parameter list. |
| `strategy` | runtime 規則使用 `after`。Required execution strategy is `after`. |
| `signature` | 選填；最多 512 字元，僅供顯示，執行時以 `parameterTypes` 為準。Optional display text. |
| `runtime.parameterTypes` | 必填陣列；0–16 個完整型別名。Required exact parameter list. |
| `runtime.source` | `argument`（預設）、`return` 或 `this`。Select an argument, the original return value, or the receiver. |
| `runtime.argumentIndex` | `argument` 模式下為從 0 起算的有效索引。Zero-based valid index when source is argument. |
| `runtime.fieldPath` | 0–8 個欄位名，預設空陣列。Up to 8 field names; empty by default. |
| `runtime.clearOnNull` | 預設 `true`；來源值為 null 時發清空，false 則略過此回呼。Defaults to true; false ignores a null source. |

參數型別使用 `java.lang.String`、`java.lang.CharSequence`、`java.util.List` 等完整名稱。泛型擦除後只填 `java.util.List`，不填 `List<Cue>`。支援八種 primitive 與一維 `[]` 型別，例如 `int`、`byte[]`；不使用 Kotlin 的 `String`、`Int` 等別名。

Use full names such as `java.lang.String`, `java.lang.CharSequence`, and `java.util.List`. Generic types use their erased name, such as `java.util.List`, rather than `List<Cue>`. The eight primitive types and one-dimensional arrays such as `int` and `byte[]` are supported. Kotlin aliases are not accepted as JVM class names.

原方法會先正常執行；其回傳值與例外維持原行為。成功回傳後才讀取來源及欄位。`fieldPath` 在一般物件上讀取指定欄位（含繼承欄位），遇到 iterable／陣列則對各元素套用相同剩餘路徑。最終值須為 CharSequence。範例 `fieldPath: ["text"]` 會把 `List<Cue>` 中各 Cue 的 `text` 合併為多行；`["cues", "text"]` 適用於持有 `cues` 欄位的 CueGroup。getter 可透過 hook 該 getter 並使用 `source: "return"` 取得結果。

The original method runs normally first; its return value and exceptions are preserved. The adapter reads its source only after a successful return. `fieldPath` reads named fields, including inherited fields. For iterables and arrays, the remaining path is applied to each element. Final values must be CharSequence. `["text"]` extracts text from each element of a `List<Cue>`; `["cues", "text"]` handles a group containing a `cues` field. To use a getter, hook that getter and select `source: "return"`.

null、空集合及空文字會產生清空（`clearOnNull: false` 只略過根來源值的 null）。找不到欄位、來源型別不對或超過限制時略過該事件；不把提取錯誤當成清空。提取最多訪問 256 個值、遞迴深度 16，輸出最多 8192 字元；不執行任意 getter／script／DEX，也不對非文字物件呼叫 `toString()`。

Nulls, empty collections, and empty text produce clear events; `clearOnNull: false` only skips a null root source. Missing fields, wrong types, or exceeded limits skip the event rather than clearing captions. Extraction visits at most 256 values with recursion depth 16 and output of at most 8192 characters. It does not execute imported scripts/DEX, invoke arbitrary getters, or call `toString()` on non-text objects.

模組於 package-ready 時依目標 class loader 查找宣告方法。類別尚未載入於可用 loader、位於後載入 split／自訂 loader、方法已改名或完全原生實作時可能無法掛載。應選擇穩定的 Java 字幕交付點；首次建立規則時記錄並測試目標 App 的 versionName／versionCode。

The module resolves declared methods through the target class loader at package-ready time. Deferred splits, custom loaders, renamed methods, or native-only rendering can prevent installation. Choose a stable Java caption callback and record the tested target app versionName/versionCode.

## 4. 同步、授權與相容 / Synchronization, authorization, compatibility

主程式透過簽章權限保護的狀態 Activity 傳送完整規則快照；模組將快照寫入框架 Remote Preferences，目標程序以唯讀方式讀取。啟用 Hook 規則會請求作用域。介面顯示「規則已同步、作用域已授權」後，完整關閉並重新開啟目標 App。框架不支援 Remote Preferences 或模組過舊時，自訂規則無法同步，內建路徑仍可使用。

The main app sends a full snapshot through the signature-protected status activity. The module writes it into framework Remote Preferences; target processes read it. Enabling a Hook rule requests scope approval. After the UI reports synced rules and approved scope, fully close and restart the target app. Custom rules cannot synchronize with an old module or a framework lacking Remote Preferences; built-in paths remain available.

Hook v1 目錄仍需原有 `integrity` SHA-256 且只作描述。v2 的 integrity 可省略，方便作者手寫範例；若提供則會校驗。程式匯出時會附上 checksum，手動改檔後可移除整個 integrity 物件再匯入 v2。checksum 用於檢查檔案內容，並非作者簽章。內建 ID `youtube`、`nhk-plus` 保留；建議以作者／專案名稱作自訂 ID 前綴。

Legacy Hook v1 catalogs still require their original integrity checksum and remain descriptive. v2 permits omission of integrity for hand-authored files; when provided, it is validated. App exports include a checksum; remove the entire integrity object after manually editing a v2 export, or regenerate it. The checksum is not an author signature. Built-in Hook IDs `youtube` and `nhk-plus` are reserved; use an author/project prefix for custom IDs.

每個目錄最多 128 項（含內建）；匯入檔最多 1,000,000 字元。所有自訂 Hook 規則合計的執行快照最多 200,000 UTF-8 bytes。更新會替換條目，不作欄位合併。匯出的 enabled/builtIn 不能讓匯入者跳過手動啟用或覆寫內建實作。

Each catalog holds at most 128 entries including built-ins. Import files are capped at 1,000,000 characters. The combined runtime snapshot of all custom Hook adapters is capped at 200,000 UTF-8 bytes. Updates replace entries rather than merging fields. Exported enabled/builtIn flags cannot bypass manual activation or replace built-in implementations.

## 5. 驗證與回報 / Validation and reporting

適配作者應確認：兩行字幕完整、逐字更新不漏句、字幕消失可清空、切換影片與 App 不混入前一來源、暫停／停止後不新增翻譯、停用／刪除規則後不再接收。請分開記錄「格式驗證通過」與「真實 App 測試通過」。分享時附上目標 App 版本、Android／框架版本、主程式／Hook 版本、測試範圍及已知限制，不附帶字幕原文、憑證或會話令牌。

Authors should verify complete multiline captions, incremental updates, clear events, video/app switching, pause/stop behavior, and disabling/removing rules. Distinguish format validation from real app compatibility. Include target app, Android/framework, main app/Hook versions, tested scenarios, and known limits when sharing an adapter. Omit private caption text, credentials, and session tokens from reports.

框架接口參考 / Framework references: [libxposed API](https://libxposed.github.io/api/io/github/libxposed/api/XposedInterface.html), [module metadata](https://libxposed.github.io/api/io/github/libxposed/api/package-summary.html), [libxposed service](https://github.com/libxposed/service).
