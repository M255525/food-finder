# CLAUDE.md — 巷口美食通 (food-finder)

台北/新北三餐美食建議網站。單一 `index.html`，無建置步驟、無框架、無後端。直接開啟或用靜態伺服器託管即可。**本資料夾是獨立 git 儲存庫**（commit identity 本地設定 `Mark Tsai <tsaimark@gmail.com>`）。

主要使用族群是**樂齡與上班族**，因此整體設計刻意往「越少東西要設定越好」「文字越大越好」傾斜，不是預設的通用做法，改動時請維持這個方向。

## 檔案

- `index.html` — 主程式：設定面板（單一 LLM 金鑰 BYOK）、搜尋表單、AI 推薦生成與渲染邏輯全部在這一個檔案裡
- `mrt-stations-data.js` — 台北捷運＋新北捷運（環狀線／安坑輕軌／淡海輕軌）共 138 站，`window.MRT_STATIONS = [...]`
- `manual.html` — 操作手冊，含金鑰申請步驟、隱私說明、創作者資料（與 `sbir-generator`／`icap-generator`／`Prompt`／`phoenix-loan-generator` 的創作者經歷區塊同一份，更新其中一邊時同步其餘各邊）

## 核心架構：純 LLM 生成推薦，不接 Google Maps/Places API

**這個專案曾經實作過一版接 Google Places API (New) 的即時查詢版本，後來整個拿掉了。** 原因是使用者明確表示「只會使用一般語言模型的 API key」——目標族群是樂齡與上班族，要求他們去 Google Cloud Console 開專案、開通兩個 API、綁信用卡、設定 HTTP referrer 限制，這件事本身的門檻就足以讓這群人放棄使用，跟 `expense-tracker-pwa` 拿掉 Google Sheet+Apps Script 雲端同步是同一種取捨（見該專案 `CLAUDE.md`：「一般使用者實測回饋部署門檻太高」）。所以現在的版本：

- **只有一把 API 金鑰**：使用者自選 Claude／OpenAI／Gemini／OpenRouter 其中之一，設定面板只有這一組欄位（provider/model/key），存在 `localStorage` key `foodFinderSettings`。
- **沒有真實地點查詢**：不呼叫任何地圖或商家資料庫 API，`buildRecommendationPrompt()` 把使用者填的地點文字、餐期、距離偏好、人數、預算、美食類型、自由文字，整理成一段提示詞，直接請 LLM 扮演「熟悉台北/新北美食的在地朋友」生成建議清單（JSON：`{places:[{name,category,distanceHint,priceHint,locationHint,reason}]}`）。**所有欄位都是 AI 依訓練知識生成的文字，不是即時資料**，`distanceHint`／`priceHint` 是 AI 自己估的，不是測量或查表得出。
- 因為資料來源不再可信、不可驗證，**誠實揭露是這個版本最重要的設計要求**：頂部警語、每個結果卡片的「AI 建議」標籤、每張卡片的「🔍 上網查證這家店是否還在營業」連結（純文字 Google 搜尋連結，`https://www.google.com/search?q=店名+地點`，不需要任何金鑰）、`manual.html` 的警語段落，四處都要一致強調「這是 AI 生成、不是即時資料、出發前請自行查證」，修改文案時不要把這個語氣改弱。
- Prompt 裡明確要求「若沒把握，寧可少列幾家、標註請自行查證，也不要編造不存在的店名或捏造精確地址電話」——這是唯一能降低（不是消除）幻覺風險的做法，改 prompt 時保留這段指示。

## 定位方式：只剩「輸入地址」與「選捷運站」

拿掉了原本的「使用目前定位」（GPS）：沒有 Google Geocoder 可以把座標反查成地名，而讓 LLM 自己解讀原始經緯度既不可靠、對使用者也不透明，所以直接不做，只留兩種**純文字**的地點輸入方式。`mrt-stations-data.js` 裡的 `lat`/`lng` 欄位因此**目前完全沒被讀取**——只用 `name`／`lines` 組出下拉選單文字（例如「捷運市政府站附近」），保留座標欄位是為了萬一之後又要接回真實地圖服務時不必重查一次 Overpass；不要因為看起來沒用就刪掉這兩個欄位。

## 無障礙／樂齡文字大小

`html{font-size:118%}` 是全站預設放大的基準（不是 100%），所有元件字級再疊加 rem 為單位。右上角「小／中／大」三顆按鈕（`#fontSizeGroup`）直接調整 `document.documentElement.style.fontSize`（100%/118%/136%），存 `localStorage` key `foodFinderFontScale`，預設 `md`。互動元件（chip／按鈕／輸入框）的內距刻意用 `em` 不是 `px`，這樣文字放大時觸控熱區也會跟著變大，不是只有字變大但按鈕大小不變——之後新增互動元件時延用同一個單位慣例。所有按鈕/chip 最小高度 44px 起跳（觸控無障礙常見門檻）。

## LLM BYOK

`callLLM()` 實作與 `sbir-generator`／`icap-generator`／`Prompt` 同一套模式（Claude 用 `x-api-key`+`anthropic-dangerous-direct-browser-access`、Gemini 用 `x-goog-api-key`、OpenAI/OpenRouter 用 Bearer；429/500/503/529 自動重試 1 次），修改時可互相參照。因為這把金鑰現在是**唯一的資料引擎、不是選填的加值功能**，`index.html` 載入時若偵測到 `localStorage` 沒有金鑰會自動彈出設定面板（`if(!loadSettings().llmKey) openSettings();`），搜尋按鈕本身也會擋掉沒填金鑰的請求並顯示錯誤——不要把這個 mandatory 檢查改回「選填」邏輯。

## 頂部跑馬燈

與 `ai-video-studio` 系列共用同一個授權伺服器與 Google Sheet，做法比照一般頁面（`position:fixed` + `body.has-marquee{padding-top:34px}`，跑馬燈列高度因整體字級放大而加高），`localStorage` key 為 `foodFinderMarquee`。本工具沒有序號登入機制，頁面載入時直接 POST 空序號取得 `marquee` 欄位。

## 預覽

port **8787**（定義於 `.claude/launch.json`，名稱 `food-finder`）。以 Preview MCP 的 `preview_start` 啟動，或 `python -m http.server 8787 --directory 小工具/food-finder`。

**測試時注意**：用一把格式正確但無效的金鑰（例如 `sk-ant-fake-test-key`）對著真實服務商端點打，可以驗證錯誤處理路徑（例如 Claude 會回傳真實的 401 `invalid x-api-key`），不需要一把真的有額度的金鑰就能確認請求格式與錯誤訊息顯示正確；但無法驗證 AI 實際生成內容的品質與 JSON 格式穩定性，這部分需要一把真的金鑰實測。
