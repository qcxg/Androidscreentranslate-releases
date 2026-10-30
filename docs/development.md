# 開發、驗證與發佈

對應 1.0.6，核對日期為 2026-10-30。工具與依賴以 Gradle 設定為準；本文件不保存會過期的「已完成」項目、測試總數或漏洞掃描結論。

本文件描述私人源碼倉庫的開發與發佈流程。公開下載倉庫提供 APK、文件及 JSON 適配範例；第三方適配作者可直接依[接口規格](adapters/README.md)製作規則，不需要取得私人源碼。

## 環境

| 項目 | 設定來源／值 |
| --- | --- |
| Gradle | `gradle/wrapper/gradle-wrapper.properties`，9.5.0，含 distribution SHA-256 |
| AGP／Kotlin | `gradle/libs.versions.toml`，9.3.0／2.4.10 |
| Java | CI 使用 17，本機可使用已驗證可建置的 JBR 21；bytecode target 17 |
| Android | compile SDK 37，Build Tools 37.0.0，min SDK 26，target SDK 36 |
| 模組 | `app`、`hook`；Hook 使用 libxposed API／service 102.0.0 |

安裝 SDK Platform／Build Tools／Platform Tools，以 `ANDROID_HOME` 或 `local.properties` 指定 SDK。`local.properties`、Android Studio 設定及簽章資料不提交。

## 本機建置與驗證

Windows PowerShell 範例：

```powershell
$env:JAVA_HOME = 'C:/Program Files/Android/Android Studio/jbr'
$env:ANDROID_HOME = "$env:LOCALAPPDATA/Android/Sdk"
./gradlew.bat :app:assembleDebug :hook:assembleDebug
./gradlew.bat :app:testDebugUnitTest :hook:testDebugUnitTest :app:lintDebug :hook:lintDebug
./gradlew.bat :app:assembleRelease :hook:assembleRelease
```

macOS／Linux 使用 `./gradlew`。Debug APK 位於各模組 `build/outputs/apk/debug/`，本機 Release 位於 `build/outputs/apk/release/` 且未簽署。主程式 Release 不啟用 minify，Hook Release 啟用 minify；修改 Hook 入口時須核對其 ProGuard 規則與 Xposed resources。

單元測試位於 `app/src/test/` 與 `hook/src/test/`，覆蓋字幕快照與發布、事件保留策略、OCR 管線、翻譯執行器與 Provider、備份解析／加密、適配格式與範例、欄位／集合提取、來源身分及每來源令牌、權限狀態等。Provider 測試多使用 MockWebServer，不能代表第三方服務當下可用性；適配解析測試不代替目標 App 實測。

需要介面／系統權限驗證時，連接測試手機：

```powershell
./gradlew.bat :app:connectedDebugAndroidTest --no-configuration-cache
```

直接執行 instrumentation runner 的方式：

```powershell
./gradlew.bat :app:assembleDebug :app:assembleDebugAndroidTest
adb install -r app/build/outputs/apk/debug/app-debug.apk
adb install -r app/build/outputs/apk/androidTest/debug/app-debug-androidTest.apk
adb shell am instrument -w -r com.shiraka.screentranslator.test/androidx.test.runner.AndroidJUnitRunner
```

介面測試會操作裝置權限及設定。安裝 Debug 前核對既有主程式／Hook 的簽章；正式 Release 與本機 Debug 不應混裝。`apksigner verify --print-certs <APK>` 可核對憑證 SHA-256；檔案 SHA-256 校驗與簽章憑證校驗是不同用途。

測試報告及 Lint 位於 `app/build/reports/` 等產物目錄。每次驗證結果以該次報告、提交及 Actions run 為準，不沿用舊測試數量或裝置驗證結論。YouTube／NHK+ 新版相容性、真實 API、OEM 模糊效果及長時間工作階段需依使用環境實測。

## 版本與發佈

`app/build.gradle.kts` 與 `hook/build.gradle.kts` 同步遞增 `versionCode`／`versionName`；發佈說明放在 `docs/releases/<version>.md`。版本升級應為持續遞增的安裝版本，不加 `(1)` 等產物計數後綴。

`.github/workflows/release.yml` 的觸發方式：

| 操作 | 結果 |
| --- | --- |
| 只推送一般分支提交 | 不自動編譯 |
| 手動執行 `workflow_dispatch` | 檢查、編譯、簽署並上傳 Actions artifact，不建立 Release |
| 推送 `v*` tag | 同上，並發佈私人源碼倉庫及公開下載倉庫的 Release |

發佈流程執行 `:app:testDebugUnitTest`、`:hook:testDebugUnitTest`、`:app:lintRelease`、`:app:assembleRelease`、`:hook:assembleRelease`，之後以同一份金鑰簽署兩個 APK 並驗證，再打包中英雙語指南、JSON 範例及技術文件。正式 tag 同時建立私人與公開 Release；建立公開 Release 前先同步公開 README 與文件目錄，讓新公開 tag 建立在該文件提交上。Artifacts 保留 30 天，正式下載使用 Release assets。

必要 Secrets：

| 名稱 | 用途 |
| --- | --- |
| `ANDROID_KEYSTORE_BASE64` | Release keystore 的 Base64 |
| `ANDROID_KEYSTORE_PASSWORD` | Keystore 密碼 |
| `ANDROID_KEY_ALIAS` | 簽署 alias |
| `ANDROID_KEY_PASSWORD` | Key 密碼 |
| `PUBLIC_RELEASE_TOKEN` | 具公開下載倉庫內容寫入權限的發佈權杖 |

金鑰及權杖不寫入文件或提交。一般測試更新只推送並手動觸發，不移動現有發佈 tag；只有正式發佈才建立新版本 tag。觸發後不預設等待編譯完成或自動安裝手機。

私人倉庫 Release 保留帶版本的 APK。公開倉庫 `qcxg/Androidscreentranslate-releases` 僅發佈固定名稱：

- `screen-translator.apk`
- `screen-translator-hook.apk`
- `SHA256SUMS.txt`，以這兩個公開檔名計算
- `Screen-Translator-Guide.zh-en.md`：專案介紹與使用／適配完整指南
- `Adapter-Spec.zh-en.md`：第三方接口的中英雙語規格
- `adapter-examples.zip`：可修改後匯入的 JSON 模板
- `technical-docs.zip`：含目錄結構的離線文件與範例

官網可固定連結至 `releases/latest/download/<檔名>`，完整地址見[首頁](../README.md)。公開 tag 建立在公開倉庫自身歷史，不推送私人源碼。若建置成功而公開發佈失敗，先檢查該步驟及權杖權限，再重新執行該 tag 的流程；既有 assets 會以同名覆寫。

## 公開文件同步

`scripts/publish_public_docs.py` 使用固定檔案清單：完整指南（同時作為公開 README）、接口規格、三份 JSON 範例、架構、安全、備份、開發、Hook 文件與該次版本說明。它建立公開倉庫自身的文件提交，不複製或推送私人 Git 歷史，也不掃描上傳整個工作目錄。更新公開分支時採非強制推送語意，避免覆蓋同時發生的提交。

本機可先檢查清單與離線包：

```powershell
python scripts/publish_public_docs.py --version 1.0.6 --output-zip artifacts/technical-docs.zip
```

`--publish` 會透過已登入的 gh CLI 寫入固定的公開下載倉庫，且拒絕以較舊版本覆蓋較新公開 Release 的文件。正常發佈由 workflow 在簽署與打包後、建立公開 Release 前執行；公開文件步驟失敗時可在修正問題後單獨重跑這個步驟。完成時核對公開 latest tag、七個附件、README 版本及 docs/adapters/examples 內容，不僅以私人 Release 成功作為完成條件。

## 工作目錄與清理

`build/`、各模組 `build/`、專案內 `.gradle/`、`.kotlin/` 是可重建產物或快取；`.maintenance/` 僅供臨時依賴檢查、裝置 dump 及除錯截圖。`artifacts/` 保存本機輸出，不提交；其中 `website/selected/` 與展示頁規格是保留的交付素材，不能整個當快取刪除。

清理前確認沒有正在使用的建置，解析每個絕對目標並限制在本工作目錄；使用 PowerShell 原生 `Remove-Item -LiteralPath`，不要跨 shell 拼接刪除命令。不要清理使用者全域 SDK／Gradle 快取、簽章金鑰、有效交付素材或 `gradle/wrapper/`。
