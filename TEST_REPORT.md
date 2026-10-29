# V1_2 手機扁平版驗證

日期：2026-10-29；程式版本 1.2.0。

- TypeScript 型別檢查與正式 Vite 建置成功。
- 14 項 Node 測試、16 項 Python 測試通過，含篩選、同步合併、指標、備份、資料處理與 ZIP 驗證。
- 新增瀏覽器／Python 分片演算法對照（9,003 個股票 ID）、行情打包還原、Pages 檔案白名單與錯誤分片拒絕測試。
- Service Worker 測試確認 market.json、history-XX.json 不會被程式快取攔截；專案快取隔離與離線說明保留。
- 工作流程 YAML 已解析檢查；正式網站為同層檔案，GitHub Actions 使用來源 ZIP 還原 Python 工作目錄。

交付前另外驗證全量歷史打包還原、從 SOURCE.zip 還原後的 Python 測試、Pages 產物檔案路徑，以及最終 ZIP 的 CRC／逐檔 SHA-256／單一外層資料夾。詳見外層 RELEASE_MANIFEST.json。

限制：本次重構沒有取得使用者 GitHub repository 或 Google OAuth 設定，因此未在使用者帳號實際部署、排程或登入 Drive。附帶行情保留 V1_1 的官方資料日期與缺漏；本次沒有宣稱更新為即時行情或補齊五年資料。手機實際選檔介面依系統與瀏覽器而異。

實際交付檢查：2,340 檔個股歷史與行情快照還原前後 JSON 完全一致；解開 SOURCE.zip 後的 16 項 Python 測試及 Pages staging 成功。HTTP 專案子路徑逐一讀取 48 個參照與資料檔，內容長度一致。正式扁平包共 56 檔（含完整性清單），無子資料夾，單檔均小於 25 MiB。
