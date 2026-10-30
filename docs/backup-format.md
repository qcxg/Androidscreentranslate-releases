# 備份格式與恢復

對應 App 1.0.6，核對日期為 2026-10-30；備份容器仍為 v1。實作為 `ConfigurationBackupEnvelope`、`ConfigurationBackupRepository`、`ConfigurationBackupParser` 及 `EncryptedBackupCodec`。

## 正式容器

格式為 `screen-translator.backup`、`formatVersion: 1`。不接受先前重建過程的 schema v2／v3 設定檔；格式版本與 App 版本、單獨錄製匯出版本各自獨立。

| 欄位 | 語意 |
| --- | --- |
| `format` | 固定為 `screen-translator.backup` |
| `formatVersion` | 整數 `1` |
| `exportedAt` | 正常匯出使用 UTC ISO-8601 時間 |
| `packageName` | 必須與主程式套件相符 |
| `appVersion` | 產生備份的 App 版本 |
| `sections` | 分區名稱清單，不接受未知或重複分區 |
| `protection` | `none` 或 `password` |
| `payload` | 無密碼時的分區內容 |
| `encryptedPayload` | 有密碼時的加密物件 |

| 分區 | 內容與支援狀態 |
| --- | --- |
| `basic` | 語言、外觀、首頁與字幕、OCR、浮窗位置、無障礙使用後關閉選項、翻譯服務與私人 API 憑證 |
| `recordings` | 錄製開關與完整時間軸 |
| `plugins` | 名稱預留，目前匯出及恢復均不支援；Hook／無障礙適配目錄需另行匯出，框架作用域不備份 |

至少選一個分區，外層清單必須與解碼後 `payload` 的分區集合完全一致。各分區的完整欄位由解析器定義，不能只建立空物件作為有效備份。

## 密碼保護

可不設密碼匯出，介面會警告明文風險。`basic` 包含私人 API 憑證，無密碼檔案會直接含有憑證，沒有密碼備份的加密認證保護。

密碼非空時，將 `payload` 的 UTF-8 JSON 加密；外層版本、時間、套件與分區清單仍可讀。`encryptedPayload` 含 `algorithm`、`iterations`、`salt`、`iv`、`ciphertext`。

- 演算法標記：`PBKDF2-HMAC-SHA256/AES-256-GCM`
- 匯出 KDF：PBKDF2-HMAC-SHA256，150,000 次，256-bit key
- Salt：16 bytes；GCM IV：12 bytes；認證 tag：128 bits
- AAD：`com.shiraka.screentranslator.backup.v1`
- 二進位欄位採 Base64；解密核對演算法、迭代範圍與必要長度

GCM 認證保護加密內容，不代表外層全部 metadata 都納入認證；分區清單與解碼內容的一致性另行驗證。

## 預檢與恢復

1. 選檔後限制讀取長度，檢查格式、版本、套件、分區及保護方式。
2. 如需密碼，解密後才解析；全部分區完成預檢，失敗顯示錯誤，不進入寫入階段。
3. 顯示可用分區與錄製數量，使用者選擇實際恢復範圍後才寫入。
4. 恢復基礎設定時重新載入介面、回首頁並顯示成功提示；只恢復錄製則更新頁面及成功提示。

錄製恢復是**替換**既有集合，不是追加。先寫同層 staging 目錄再換入正式目錄，安裝失敗時嘗試回復原目錄。基礎設定分別寫入 Repository，與錄製並非單一跨儲存交易；磁碟或系統寫入失敗仍可能造成部分變更，預檢不等同整體交易保證。

容器限制 50,000,000 個字元；最多 10,000 個錄製、總字幕 250,000 條；單條原文及各譯文上限 200,000 個字元。識別碼、時間及其他欄位由解析器檢查或限值。變更上限須同步核對讀檔、Envelope 與錄製解析路徑。

錄製頁另可匯出 `screen-translator.subtitle-recordings`、版本 1 的 JSON。這是分享／匯出格式，不是設定頁接受的備份容器，目前錄製頁沒有對等的單獨匯入入口。
