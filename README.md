# 股序 V1_2｜手機扁平版

五頁台股研究 PWA：首頁、篩選、技術分析、分類與設定。版本 1.2.0。

1. 將本資料夾裡的全部檔案直接上傳 GitHub repository 根目錄。不要只上傳外層 ZIP。
2. Settings → Pages → Deploy from a branch → main → /(root)，先發布網站。
3. 開啟網站的 MOBILE.html，複製工作流程，依 SETUP.html 在 GitHub 建立 `.github/workflows/update-data.yml`，再改用 GitHub Actions 發布與自動更新。

詳見 [完整設定說明](SETUP.md)。不需要手機安裝 Node、Python 或 Git。

SOURCE.zip 是完整可編輯原始碼；PIPELINE_STATE.zip 是公開來源快取與續傳狀態。兩個檔案保持 ZIP 格式一起上傳，不用再解壓。它們由 GitHub Actions 還原使用。

market.json 與 32 個 history-XX.json 是初始公開行情；不是即時報價。後續行情會保存在 market-data 分支並由 Actions 直接部署。私人分類與筆記留在本機，選擇啟用後同步到你的 Google Drive。

若沿用舊版專案，先停用舊資料與發布工作，避免覆寫本版。工作流程必須位於 `.github/workflows/`，根目錄的 update-data.workflow.yml 只是供手機複製的範本。

ZIP 已附 RELEASE_MANIFEST.json 逐檔雜湊，可使用 SOURCE.zip 中的 scripts/release_archive.py 驗證完整性。
