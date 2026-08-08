# CLAUDE.md — 巷口美食通 (food-finder)

台北/新北三餐美食建議網站。單一 `index.html`，無建置步驟、無框架、無後端。直接開啟或用靜態伺服器託管即可。**本資料夾是獨立 git 儲存庫**（commit identity 本地設定 `Mark Tsai <tsaimark@gmail.com>`）。

## 檔案

- `index.html` — 主程式：設定面板（Google Maps／LLM 金鑰 BYOK）、搜尋表單、Google Places 查詢與渲染邏輯全部在這一個檔案裡
- `mrt-stations-data.js` — 台北捷運＋新北捷運（環狀線／安坑輕軌／淡海輕軌）共 138 站，`window.MRT_STATIONS = [...]`
- `manual.html` — 操作手冊，含金鑰申請步驟、隱私說明、創作者資料（與 `sbir-generator`／`icap-generator`／`Prompt`／`phoenix-loan-generator` 的創作者經歷區塊同一份，更新其中一邊時同步其餘各邊）

## 核心架構決策：改用 Places API (New)，不是 legacy PlacesService

原始設計曾採用 `google.maps.places.PlacesService.nearbySearch()`（多數網路教學仍在用的舊版），**但在 Playwright 對著真實 Google 伺服器打一個假金鑰測試時，Chrome console 直接跳出 Google 官方警告：`PlacesService is not available to new customers as of March 1st, 2025`**——2026 年才申請 Google Cloud 專案的使用者屬於「新客戶」，這條路完全不通，不是可以忽略的過時警告。因此整個查詢邏輯改寫成 `google.maps.places.Place.searchNearby()`（新版 Places API 的 JS 類別，透過同一個 `<script src=...&libraries=places>` 載入即可用，不需要改用 `importLibrary()` 非同步載入模式）。

新舊版差異，改動時務必記得：

- **新版 Nearby Search 沒有 `keyword` 全文檢索參數**，只能用 `includedTypes`（Table A 型別陣列，OR 條件）做伺服器端篩選。`TYPE_MAP_CHIP` 把「咖啡店/烘焙/日式/中式」四個美食類型 chip 對應到乾淨的 Table A 型別（`cafe`/`bakery`/`japanese_restaurant`/`chinese_restaurant`）；其餘 chip（火鍋/麵食/西式/素食/小吃）與自由文字/LLM 解析出的關鍵字**沒有對應的 Table A 型別**，改成抓回結果後在前端用 `matchesKeyword()` 對店名／分類做子字串比對（無 LLM 時才會硬篩選；有 LLM 時交給 `buildSummaryPrompt()` 的語意排序處理，避免子字串比對誤刪掉語意相關但用詞不同的店家）。
- **半徑限制改用 `locationRestriction: {center, radius}`**（圓形、硬性），不是舊版的 `radius` 參數，但語意等價，一樣是「不會出現超出範圍的店家」。
- **價位與營業中都沒有伺服器端篩選參數**（新版 Nearby Search 只回傳這兩個欄位、不能拿來篩選），改成抓回結果後前端 `matchesBudget()`／`matchesOpenNow()` 後篩選。
- **`priceLevel` 從舊版的 0-4 整數改成字串 enum**（`FREE`/`INEXPENSIVE`/`MODERATE`/`EXPENSIVE`/`VERY_EXPENSIVE`），`PRICE_LABELS`／`PRICE_LEVEL_GROUPS` 兩份對照表都是字串 key，不要誤植回整數索引。
- 回傳的 `Place` 物件是類別實例（`place.displayName`／`place.location`／`place.priceLevel`…都是屬性存取），不是舊版那種普通 JSON（`place.name`／`place.geometry.location`／`place.price_level`）——`toCardModel()` 對 `displayName`／`location` 都做了防禦性型別判斷（可能是字串也可能是 `{text}` 物件、`.lat`/`.lng` 可能是方法也可能是數字），這是刻意保留的相容寫法，不是多餘的判斷。
- `place.googleMapsURI` 新版直接回傳可用的 Google 地圖連結，不用再自己拼 `place_id:` 網址（但仍保留 fallback 拼法以防某些回應缺這個欄位）。

## MRT 車站資料來源

`mrt-stations-data.js` 的座標**不是手動輸入**，是用 OpenStreetMap Overpass API 查詢台北市／新北市境內 `railway=station` 節點（`network` 篩選「臺北捷運」「新北捷運」），同名站多筆節點（不同月台/出入口）取平均座標，共 138 站；路線標籤（`lines` 欄位）是拿查詢結果比對台北捷運/新北捷運官方路網人工標註的，Overpass 本身沒有可靠的路線分組資料。座標僅供搜尋中心點使用，精度不是站體出入口等級——如果之後要更新或擴充站點，重新查一次 Overpass 比手動改座標可靠。

有一個名為「行天宮」的 OSM 節點被排除掉了：座標落在松江南京站與中山國小站之間，不是真實存在過的捷運站（可能是誤標的地標點），保留在清單裡會讓使用者選到一個查無此站的「站名」。

## LLM BYOK

金鑰、provider、model 存 `localStorage` key `foodFinderSettings`（與 Google Maps 金鑰同一個物件）。`callLLM()` 實作與 `sbir-generator`／`icap-generator`／`Prompt` 同一套模式（Claude 用 `x-api-key`+`anthropic-dangerous-direct-browser-access`、Gemini 用 `x-goog-api-key`、OpenAI/OpenRouter 用 Bearer；429/500/503/529 自動重試），修改時可互相參照。兩個呼叫點：`buildParsePrompt()`（自由文字→關鍵字 JSON）、`buildSummaryPrompt()`（候選清單→最多 5 筆推薦+理由 JSON，**明確要求只能從清單裡挑選**，避免模型編造清單以外的店家）。兩者任一失敗都要 catch 住、顯示警告 banner、退回規則式邏輯，不能讓 AI 失敗擋住基本查詢功能。

## 頂部跑馬燈

與 `ai-video-studio` 系列共用同一個授權伺服器與 Google Sheet，做法比照一般頁面（`position:fixed` + `body.has-marquee{padding-top:30px}`），`localStorage` key 為 `foodFinderMarquee`。本工具沒有序號登入機制，頁面載入時直接 POST 空序號取得 `marquee` 欄位。

## 預覽

port **8787**（定義於 `.claude/launch.json`，名稱 `food-finder`）。以 Preview MCP 的 `preview_start` 啟動，或 `python -m http.server 8787 --directory 小工具/food-finder`。

**測試時注意**：Places/Maps 功能需要一把真的 Google Maps API 金鑰才能完整測試；用假金鑰測試時 Google 會回傳 `gm_authFailure`（腳本載入後才觸發）或 `Place.searchNearby()` 直接 reject（`API key not valid`），這兩條錯誤處理路徑都已對著真實 Google 伺服器驗證過會正確顯示中文錯誤訊息，不會讓頁面卡住或丟出未捕捉的例外。
