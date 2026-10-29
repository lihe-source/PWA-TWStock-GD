# 股序 V1_2：手機扁平版設定步驟

本版不用電腦、不用 npm、不用手動編譯。網站、行情與上傳用輔助檔案都在同一層。請先發布網站，再啟用每日自動更新。

**外層資料夾只用來整理下載內容；GitHub 根目錄必須直接看到 index.html。SOURCE.zip 與 PIPELINE_STATE.zip 不要再解壓，直接當成檔案一起上傳。**

## 1. 用手機上傳檔案

1. 下載並解壓 `PWA-TWStock-GD-V1_2-FLAT.zip`。iPhone 可在「檔案」App 點一下 ZIP 解壓；Android 可用系統檔案管理員。
2. 用 Safari 或 Chrome 登入 github.com。建議新建 Public repository，名稱可用 `PWA-TWStock-GD`，預設分支 **main**，勾選建立 README。看不到網頁按鈕時，切換「要求桌面網站」。
3. 在 repository 選 **Add file → Upload files → choose your files**，進入解壓後的同名資料夾，全選裡面的檔案上傳。手機選檔器若無法全選，可分幾批選取。
4. **不要只上傳外層 ZIP，也不要把外層資料夾當子目錄上傳。** 上傳後根目錄應直接看到 index.html、market.json、history-00.json 等檔案。
5. 填入提交說明，選 **Commit changes**。若手機選檔器漏檔，補傳即可；必須包含 SOURCE.zip、PIPELINE_STATE.zip 與 SITE_FILES.json 才能啟用後續自動更新。

GitHub 網頁每批最多 100 檔、每檔最多 25 MiB。本版少於 100 檔，檔案大小已逐一檢查。來源與續傳壓縮檔只含程式及官方公開資料，不含你的 Google 登入或私人記錄。

若沿用 V1_1 的 repository，先匯出個人備份；到 Actions 停用舊的「市場資料與歷史補建」及「部署股序 PWA」，避免舊工作覆寫新網站。再更新根目錄及建立新的工作流程。既有 market-data 分支會沿用，不要刪除。

## 2. 先發布可直接使用的網站

1. 開啟 repository → **Settings → Pages**。
2. Source 選 **Deploy from a branch**，Branch 選 **main**，資料夾選 **/(root)**，按 Save。
3. 等發布完成，使用 Pages 顯示的網址開啟網站。若用專案 repository，網址通常是 `https://你的帳號.github.io/PWA-TWStock-GD/`。
4. 確認首頁、資料篩選、技術分析、資料分類、設定都能使用；設定頁版本應為 **1.2.0**。

這時顯示壓縮檔附帶的官方行情快照。**完成下一節，才會每天自動抓取新資料。** 資料仍顯示來源實際日期；附帶歷史並非完整五年。

## 3. 用手機啟用每日自動更新

GitHub 只會執行 `.github/workflows/` 裡的 YAML。這是唯一必須建立的路徑，其他上傳檔案全部同層。

1. 在已發布的網站網址後面加 `MOBILE.html`，例如 `https://你的帳號.github.io/PWA-TWStock-GD/MOBILE.html`，按 **複製完整工作流程**。也可在 GitHub 開啟根目錄的 `update-data.workflow.yml`，複製全部文字。
2. 到 **Settings → Actions → General** 允許 Actions，Workflow permissions 選 **Read and write permissions**，按 Save。無須提供 PAT、Google 密碼或其他 secret。
3. 到 **Settings → Pages**，把 Source 改成 **GitHub Actions**。
4. 到 **Code → Add file → Create new file**。檔名完整輸入 `.github/workflows/update-data.yml`，把剛才複製的內容貼入，確認不是空白，按 Commit changes。GitHub 會依斜線建立路徑；只把 YAML 放根目錄不會執行。
5. 到 **Actions → 股序：發布與更新**。首次 push 只發布現有資料；等工作成功後，按 **Run workflow**，分支選 main，mode 選 **update**，history_batch 選 **0**，執行一次確認官方資料可以更新。
6. 需要歷史補建時，選 mode=update、history_batch=80。之後排程自行運作，不必讓手機保持開機。mode=publish 只重新發布，不向行情來源發出請求。

工作成功會直接部署 Pages，無須另建 pages.yml。失敗請開啟該次工作看紅色步驟；網站會保留上次成功發布的版本。根目錄的 market.json 是初始快照，後續有效更新保存到 market-data 分支並直接部署，**不會每日覆寫 main 的初始快照**。

|台灣時間|用途|
|---|---|
|14:35、14:50|嘗試提早取得當日行情|
|16:10、18:10、21:10|來源較晚發布時補齊資料|
|07:10|前一交易日補正|
|02:37|繼續補建歷史，每輪最多 80 個候選交易日|

15:00 前的完整當日資料不能保證；取決於官方發布時間與 GitHub 排程延遲。未到齊時保留各來源實際日期與缺漏狀態，較晚排程會繼續補齊。GitHub 長時間沒有 repository 活動可能停用排程，請留意 Actions 提示，必要時重新啟用。

歷史行情分成 32 個同層 JSON 檔。Actions 自動處理合併與還原；你不必逐檔修改。每次先載入 market-data 的續傳進度，已成功日期跳過；來源拒絕或缺漏會保留狀態並稍後重試。五年是補建目標範圍，不代表附帶或保證取得五年完整資料。

沒有 Google Drive 仍可使用本機分類、筆記與備份。要跨裝置同步，再依以下步驟設定。

## 4. 建立 Google Cloud 專案與 API

1. 開啟 https://console.cloud.google.com/ ，用你要儲存資料的 Google 帳號登入。
2. 建立一個專案，例如 **TWStock-PWA**。
3. 在專案資訊記下 **Project number／專案編號**，這是數字，不是文字 Project ID。
4. 到 **APIs & Services → Library**，啟用 **Google Drive API** 與 **Google Picker API**。

行情資料不需要 Google API；這兩個 API 只用於私人資料同步及選擇儲存資料夾。

## 5. 設定 Google Auth Platform／OAuth 同意畫面

Google 介面可能顯示 **Google Auth Platform**，或 **APIs & Services → OAuth consent screen**。

1. App 名稱填 **股序**，填你的支援信箱及開發者聯絡信箱。
2. 個人 Gmail 帳號選 External。若目前維持 Testing，將你會登入的 Gmail 加入 **Test users**。
3. Data Access／Scopes 加入：

   `https://www.googleapis.com/auth/drive.file`

4. 不需要全 Drive 的 `drive` 權限，也不需 `drive.readonly`。
5. 如需要網站及隱私政策網址，可使用你的 Pages 首頁與 `https://你的正式網站/PRIVACY.html`。本程式已包含隱私說明。若未來對外提供他人使用，依 Google 當時要求完成發布與驗證；本步驟以你自己的測試帳號使用為主。

## 6. 建立 OAuth Web Client ID

1. 到 **Clients／Credentials → Create OAuth client**。
2. Application type 選 **Web application**。
3. Authorized JavaScript origins 填你的網站 **origin**，不含專案路徑，例如：

   `https://lihe-source.github.io`

4. 本程式使用 Google Identity Services 的彈出式 token flow，不需另外填 callback 路徑。
5. 複製 Client ID，格式類似 `123456-abcdef.apps.googleusercontent.com`。
6. **不要把 Client Secret 放進網站或 GitHub。程式只需要公開的 Client ID。**

若使用自訂網域，另外加入完整 HTTPS origin。不要在 origin 後面加 `/PWA-TWStock-GD/`。

## 7. 建立 Picker API Key

1. **Credentials → Create credentials → API key**。
2. 編輯此 Key，Application restrictions 選 **Websites／HTTP referrers**。
3. 加入你的正式網站允許來源，例如：

   `https://lihe-source.github.io/*`

4. API restrictions 限定 **Google Picker API**。
5. 複製此 Key。它會用於瀏覽器，不是私密伺服器金鑰，因此必須設定網站與 API 限制。
6. Picker API Key、OAuth Client ID 與專案編號必須屬於**同一個 Google Cloud 專案**。

## 8. 指定 Google Drive 儲存資料夾

1. 在 Google Drive 建立資料夾，例如 **PWA-TWStock-GD**。保持私人，不用改成「知道連結的任何人」。
2. 開啟資料夾，複製網址，例如 `https://drive.google.com/drive/folders/xxxxxxxx`。
3. 打開股序 → **設定 → Google Drive 同步**，填入：
   - Google OAuth Web Client ID。
   - Picker API Key。
   - Google Cloud 數字專案編號。
4. 按 **保存設定 → 連線 Google**，選擇剛才的帳號並授權。
5. 按 **選擇資料夾**，在 Google 的選擇器中選取你建立的資料夾。
6. 程式會自動保存 Folder ID 與名稱。按 **立即同步**。
7. 新增一個分類與筆記，確認設定頁顯示待同步筆數歸零。

可以貼資料夾連結／ID，但**貼連結不等於取得 Google 存取權限**。第一次若出現 404／找不到資料夾，先用「選擇資料夾」完成明確授權。只輸入「我的雲端硬碟/股票」文字路徑不足以辨識資料夾。

其他裝置使用相同網站網址、相同 Google 帳號、相同 Cloud 專案及同一個 Drive 資料夾，再按立即同步。Safari 分頁與加入主畫面的 PWA 可能使用不同本機資料空間，因此都需要確認 Google 連線與資料夾設定。

### Google 授權與同步原則

- 本機修改先保存在 IndexedDB，Google 失效不會讓筆記消失。
- 授權 token 只保存在分頁記憶體，過期或重開時可能需按「連線 Google」。不承諾永久免操作登入。
- 有效授權時，修改後短暫延遲自動同步；開啟中的程式也定期檢查遠端變更。
- 程式關閉時不會在背景強制續授權私人 Drive；未同步修改需下次開啟再完成。
- 多裝置筆記衝突保留副本供你確認。資料夾刪除只隱藏分類關係，筆記保留。
- 避免手動刪除 Drive 內的 `changes-*.json`；它們是同步變更記錄。雲端備份可另保留最新五份。

## 9. 安裝 iPhone／iPad／Windows PWA

- **iPhone／iPad**：Safari 開啟正式 HTTPS 網址 → 分享 → 加入主畫面。首次先連網開啟，等待資料載入與離線快取完成。
- **Windows**：Edge 或 Chrome 開啟正式網址，使用網址列或選單的安裝應用程式功能。
- 圖表按右上角全螢幕圖示，再轉動手機即可橫向檢視；手機若啟用方向鎖定，需自行關閉。
- 不要直接雙擊 `index.html` 以 `file://` 開啟；模組、資料下載、Google 授權與 Service Worker 需要 HTTP／HTTPS。

## 10. 更新程式與備份

1. 手機使用者收到新版扁平 ZIP 後，解壓並將全部檔案上傳同一個 repository 根目錄，等待「股序：發布與更新」成功。
2. 保留 market-data 分支與 `.github/workflows/update-data.yml`。若新版有修改工作流程，依新版說明重新貼上 YAML。
3. 開啟 PWA 會自動檢查版本；也可在設定頁按「檢查程式更新」。更新不清除個人分類與筆記。
4. 開發者需要修改原始碼時，將 SOURCE.zip 解壓到獨立工作資料夾；依其中 README.md 還原初始資料、測試及建置，再產生完整扁平發佈包。不要只上傳 TSX 原始碼期待瀏覽器自行編譯。

誤刪後可到設定的 **回收區** 按還原。舊備份採合併模式，不會自行取消較新的刪除；需要恢復已刪項目時，先合併備份，再到回收區選擇還原。

建議在設定頁定期 **匯出完整備份**。可使用 **建立雲端備份** 保留最新五份；還原採變更合併，不以舊檔整份覆蓋新記錄。匯出含公開 Client ID、受限制的 Picker API Key 與資料夾 ID，不含登入 token；仍請當成你的個人檔案保存。

## 11. 常見問題

|狀況|處理方式|
|網站 404|確認 Pages Source 選 GitHub Actions，並查看部署工作是否成功；網址的 repository 名稱要正確。|
|找不到自動工作|確認 `.github/workflows/update-data.yml` 在 main 分支的正確路徑，而非多包一層資料夾。|
|data 工作 push 被拒絕|確認 Actions workflow permissions 允許 write；若分支保護限制 bot，依你的規則調整 market-data 分支。|
|首頁仍是前一日|看設定頁各來源交易日及最後整理時間；非交易日正常保留前日。交易日則查看 Actions 與官方是否尚未發布。|
|K 線較短／技術篩選沒有結果|查看個股歷史筆數及缺漏，執行歷史補建；不足 20／60／240 日不能產生對應指標。|
|來源 403／307／429 或逾時|保留已取得資料，等待下一輪；不要快速反覆執行或繞過來源限制。|
|Google origin_mismatch|Web Client 的 JavaScript origin 要與網站相同，HTTPS，且不含專案路徑。|
|Google access_denied|確認正在使用已列入 Test users 的帳號，scope 為 drive.file，API 與 OAuth 同屬一個專案。|
|Picker API Key 錯誤|確認啟用 Google Picker API、網站 referrer 包含目前網域、專案編號是數字且與 OAuth 同專案。|
|資料夾 404／403|先用 Picker 選取該資料夾、確認登入相同帳號且有寫入權限；不要公開資料夾來處理權限錯誤。|
|Google 彈出視窗沒出現|允許此網站彈出視窗；若提示 Google 剛載入，再按一次連線。|
|換裝置沒看到筆記|用相同網址／帳號／資料夾完成同步，並確認原裝置的待同步筆數為 0。|
|更新後反覆載入|先匯出個人備份，再檢查部署檔案是否完整。不要先清空所有網站資料。|

## 12. 驗收清單

- 首頁有真實資料日期、上市上櫃行情，無模擬即時數據。
- 搜尋台積電等股票可進入技術分析，已有日線期間能正確顯示。
- 新增分類、加入股票、編輯筆記，重新整理後仍保留。
- 儲存條件、快照、匯出 CSV 與備份可用。
- 兩裝置同帳號／同資料夾可同步，衝突內容可見。
- Actions 手動更新成功後自動發布，設定頁能看到新的整理時間。
- 設定頁有版本資訊及檢查更新；手機底部導覽不遮擋內容。

## 官方參考

- GitHub Pages 建立與限制：https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits
- Actions 排程限制：https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule
- Google Drive 授權範圍：https://developers.google.com/workspace/drive/api/guides/api-specific-auth
- Google Picker：https://developers.google.com/workspace/drive/picker/guides/overview
- Google token flow：https://developers.google.com/identity/oauth2/web/guides/use-token-model

版本：V1_2／1.2.0。說明日期：2026-10-29。

手機上傳限制：https://docs.github.com/en/repositories/working-with-files/managing-files/adding-a-file-to-a-repository
