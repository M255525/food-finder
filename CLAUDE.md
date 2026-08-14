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
- **沒有真實地點查詢**：不呼叫任何地圖或商家資料庫 API，`buildRecommendationPrompt()` 把使用者填的地點文字、餐期、距離偏好、人數、預算、美食類型、自由文字，整理成一段提示詞，直接請 LLM 扮演「熟悉台北/新北美食的在地朋友」生成建議清單（JSON：`{places:[{name,category,distanceHint,priceHint,ratingHint,locationHint,reason}]}`）。**所有欄位都是 AI 依訓練知識生成的文字，不是即時資料**，`distanceHint`／`priceHint`／`ratingHint` 都是 AI 自己估的，不是測量或查表得出；`ratingHint`（評價星等）明確要求模型「不確定或沒有相關資訊請直接寫『沒有』，不要虛構數字」，前端也直接用 `p.ratingHint || '沒有'` 顯示，缺值時一律誠實標「沒有」而不是留空或猜測。
- 因為資料來源不再可信、不可驗證，**誠實揭露是這個版本最重要的設計要求**：頂部警語（含服務範圍限定台北/新北）、頁面中段固定顯示的「使用須知／創作者資料」卡片（`.about-panel`，不藏在 manual.html 裡才看得到）、每個結果卡片的「AI 建議」標籤、每張卡片的「🔍 上網查證這家店是否還在營業」連結（純文字 Google 搜尋連結，`https://www.google.com/search?q=店名+地點`，不需要任何金鑰）、`manual.html` 的警語段落，多處都要一致強調「這是 AI 生成、不是即時資料、出發前請自行查證」，修改文案時不要把這個語氣改弱。
- Prompt 裡明確要求「若沒把握，寧可少列幾家、標註請自行查證，也不要編造不存在的店名或捏造精確地址電話評價」——這是唯一能降低（不是消除）幻覺風險的做法，改 prompt 時保留這段指示。

## 定位方式：輸入地址，或先選路線再選捷運站

拿掉了原本的「使用目前定位」（GPS）：沒有 Google Geocoder 可以把座標反查成地名，而讓 LLM 自己解讀原始經緯度既不可靠、對使用者也不透明，所以直接不做，只留兩種**純文字**的地點輸入方式。

捷運站選擇是**兩層 `<select>`**（`#mrtLineSelect` → `#mrtStationSelect`），不是單一個 138 站平鋪的下拉/datalist——使用者回饋單層清單太混亂。`MRT_LINE_ORDER` 常數固定路線顯示順序（板南線／淡水信義線／松山新店線／中和新蘆線／文湖線／環狀線／安坑輕軌／淡海輕軌，依台北捷運官方慣用順序，不是字母序），選路線後才動態用 `s.lines.indexOf(line)` 篩出該路線站別填進第二個 select（預設 `disabled`，避免使用者跳過選路線直接亂選）。轉乘站（例如「市政府」同時屬於板南線／文湖線的鄰站中山國中一帶，或「中山」同時在淡水信義線與松山新店線）**會在多條路線的清單裡各自出現一次**，這是刻意的、符合真實轉乘站邏輯，不是資料重複的 bug。

`mrt-stations-data.js` 裡的 `lat`/`lng` 欄位**目前完全沒被讀取**——只用 `name`／`lines` 組出選單文字與最終地點字串（例如「捷運市政府站」，注意結尾故意不加「附近」，因為 `renderBatches()` 組合顯示文字時會自己補一次「附近」，兩邊都加會變成「站附近 附近」的重複語病，這是已經修過的坑，改動地點字串組合邏輯時要小心這個接合點）。保留座標欄位是為了萬一之後又要接回真實地圖服務時不必重查一次 Overpass；不要因為看起來沒用就刪掉這兩個欄位。

## 「都不滿意就繼續搜尋」的分批機制

`batches`（陣列，每筆 `{index, places}`）與 `excludedNames`（已出現過的店名，`Array`）是模組層級狀態。第一次按「請 AI 幫我推薦」（`runSearch(false)`）會重置這兩者並打第一次 LLM 請求；按「🔄 都不滿意？繼續搜尋其他餐飲」（`runSearch(true)`）沿用同一組 `lastCtx` 篩選條件，但把 `excludedNames` 一併寫進 `buildRecommendationPrompt()` 的排除清單段落，請模型不要重複推薦。**兩層防重複**：prompt 明文要求不重複（第一層，仰賴模型配合）＋前端收到回應後再用 `places.filter(p => excludedNames.indexOf(p.name) === -1)` 濾掉萬一還是重複的店名（第二層，防止模型沒完全照做）。`renderBatches()` 把所有批次**累加**渲染（不是用新批次覆蓋舊批次），每批前面加一個「🔎 第 N 批建議」的 `.batch-heading` 分隔——使用者要求「給出搜尋頁碼」，這裡用「第幾批」取代傳統分頁頁碼，因為 AI 生成的清單沒有真正的資料庫分頁概念，「第幾批」比較誠實地反映「這是又問了 AI 一次」而不是「翻到資料庫第 2 頁」。

## 手冊連結與頁內「關於」區塊

`manual.html` 的連結**不只放在 footer**——header 右上角有一個文字＋圖示都有的 `.manual-link`（「📖 操作手冊」，新分頁開啟），確保不用先點設定齒輪才看得到。使用須知與創作者資料也**直接嵌在首頁**（`.about-panel`，在搜尋結果區塊下方、footer 上方，一律顯示不需要展開），不是只寫在 manual.html 裡才看得到——這兩個位置都要保留，不要因為 manual.html 已經有更完整版本就把首頁這份簡版拿掉。

## 無障礙／樂齡文字大小

`html{font-size:118%}` 是全站預設放大的基準（不是 100%），所有元件字級再疊加 rem 為單位。右上角「小／中／大」三顆按鈕（`#fontSizeGroup`）直接調整 `document.documentElement.style.fontSize`（100%/118%/136%），存 `localStorage` key `foodFinderFontScale`，預設 `md`。互動元件（chip／按鈕／輸入框）的內距刻意用 `em` 不是 `px`，這樣文字放大時觸控熱區也會跟著變大，不是只有字變大但按鈕大小不變——之後新增互動元件時延用同一個單位慣例。所有按鈕/chip 最小高度 44px 起跳（觸控無障礙常見門檻）。

## LLM BYOK

`callLLM()` 實作與 `sbir-generator`／`icap-generator`／`Prompt` 同一套模式（Claude 用 `x-api-key`+`anthropic-dangerous-direct-browser-access`、Gemini 用 `x-goog-api-key`、OpenAI/OpenRouter 用 Bearer；429/500/503/529 自動重試 1 次），修改時可互相參照。因為這把金鑰現在是**唯一的資料引擎、不是選填的加值功能**，`index.html` 載入時若偵測到 `localStorage` 沒有金鑰會自動彈出設定面板（`if(!loadSettings().llmKey) openSettings();`），搜尋按鈕本身也會擋掉沒填金鑰的請求並顯示錯誤——不要把這個 mandatory 檢查改回「選填」邏輯。

## 頂部跑馬燈

與 `ai-video-studio` 系列共用同一個授權伺服器與 Google Sheet，做法比照一般頁面（`position:fixed` + `body.has-marquee{padding-top:34px}`，跑馬燈列高度因整體字級放大而加高），`localStorage` key 為 `foodFinderMarquee`。本工具沒有序號登入機制，頁面載入時直接 POST 空序號取得 `marquee` 欄位。

## 瀏覽次數計數器

`#visitCounter`（頁尾）用免費第三方服務 [CountAPI](https://countapi.mileshilliard.com)（`countapi.mileshilliard.com/api/v1/hit/foodfinder_tsaimark_visits`）記錄**累計頁面載入次數**，不是去重後的訪客數，純展示用途、無認證。`localStorage`（key `foodFinderVisitCount`）快取上次讀到的數字，網路請求失敗時靜默顯示快取值或留空，不拋錯。

**這是 `mentors-panel` 既有做法的直接沿用**（見該專案 CLAUDE.md「瀏覽次數計數器」一節），但**服務商不同**：mentors-panel 原本用的 CounterAPI（`api.counterapi.dev/v1/...`）在本次實作時實測已經回傳 `410 Gone`——v1 已停用，v2 改成需要註冊帳號＋API 金鑰，不再支援匿名呼叫，代表 mentors-panel 那顆計數器現在也已經悄悄失效中（前端設計成失敗即靜默不報錯，使用者/開發者都不會發現）。food-finder 改用另一個目前還活著、格式類似但回傳欄位是 `value` 不是 `count` 的服務（CountAPI，`countapi.mileshilliard.com`，同樣免註冊免金鑰）。**這是公開共用的 key 命名空間**，key 要夠獨特（目前用 `foodfinder_tsaimark_visits`），不要改成 `visits` 這種會撞名的通用字串。若之後這個服務也停用或不穩定，換成別的免費 counter 服務只需要改 `COUNTER_URL` 這一行與回應欄位名稱。

## 預覽

port **8787**（定義於 `.claude/launch.json`，名稱 `food-finder`）。以 Preview MCP 的 `preview_start` 啟動，或 `python -m http.server 8787 --directory 小工具/food-finder`。

## 加入主畫面（PWA，2026-08-14 新增）

比照工作區 `expense-tracker-pwa` 的做法加上安裝支援：`manifest.json`＋`icons/`（米色 `#fdf7ef` 背景、橘色 `#c8551f`「食」字圖示，對應 `--bg`／`--accent`）＋`service-worker.js`。**不追求離線可用**（本工具核心是即時打 LLM API），SW 採 network-first＋同源快取備援，跨網域請求（LLM API、CountAPI 計數器）一律略過不進快取，不需要每次改動升版 `CACHE_NAME`。安裝按鈕（`#installBtn`，📲）放在 `header.topbar` 的 `.topbar-tools`（跟「📖 操作手冊」「⚙️ 設定」同排、同款 `.icon-btn` 樣式），符合本工具「樂齡與上班族越少東西要設定越好」的一貫方向——安裝入口一開頁就看得到，不用先找設定選單。本工具沒有 `showToast`，點擊未支援安裝時改把按鈕 `title` 換成提示文字。邏輯是獨立 `<script>`，跟頂部跑馬燈、瀏覽次數計數器互不相依。已用 Playwright 實測：Chromium 確實判定本頁可安裝並觸發 `beforeinstallprompt`、SW 成功註冊為 `activated`。


**iOS／iPadOS／macOS 相容性補強（2026-08-14 同日追加）**：Safari（含 iOS 上的 Chrome/Firefox，底層都是 WebKit）**永遠不會觸發 `beforeinstallprompt`**，原本的按鈕邏輯在這些瀏覽器上一律落入「瀏覽器不支援」這句話，其實是誤導——蘋果裝置本來就能加入主畫面，只是要透過分享選單手動操作，不像 Chrome/Edge 有自動彈窗。修法：安裝腳本新增 `isIOSDevice`（`/iPad|iPhone|iPod/` 或 `navigator.platform==='MacIntel' && maxTouchPoints>1`——後者是因為 iPadOS 13+ 預設偽裝成 Mac 桌面版 UA，要用觸控點數才分得出來是 iPad 還是真的 Mac）與 `isMacDesktop && isSafariEngine`（macOS 桌面版 Safari 17+ 是「檔案→加入 Dock」，跟手機的分享選單操作不同）兩種判斷，各自顯示對應的操作指引文字，不再顯示「不支援」；`isStandalone`（`matchMedia('(display-mode: standalone)')` 或 iOS 專有的 `navigator.standalone`）為真時直接隱藏按鈕（已經是安裝後開啟，不需要再顯示安裝按鈕）。`<head>` 同步補上 `apple-touch-icon`（180×180 專用尺寸，`icons/apple-touch-icon.png`，純色不透明背景）＋ `apple-mobile-web-app-capable`／`mobile-web-app-capable`（兩個都要，前者給 Safari、後者是 Chrome 主張的新標準，只寫一個 Chrome 會在主控台噴 deprecation warning）＋ `apple-mobile-web-app-status-bar-style`／`apple-mobile-web-app-title`。這五個判斷/訊息字串在全部 9 個已加裝 PWA 的專案裡是逐字複製的同一段邏輯，日後若要調整任一處的措辭或判斷式，建議九個一起改，避免各專案的安裝體驗不一致。

**回饋機制與快取踩坑修正（2026-08-14，使用者實測回報「加入主畫面沒有功能」才發現兩層問題）**：(1) 原本無 `showToast` 時用「暫時置換按鈕文字」當提示，在工具列裡太不明顯，使用者完全沒注意到訊息出現過——改成 `window.alert(fallbackMessage())`，`deferredPrompt.prompt()` 也包 try/catch。(2) 改完使用者仍回報沒反應，追查發現 `service-worker.js` 的 `fetch(event.request)` 沒有繞過瀏覽器 HTTP 快取——GitHub Pages 對回應下 `Cache-Control: max-age=600`，10 分鐘內「network-first」名不符實，可能吃到舊版內容重新存進 Cache Storage。改成 `fetch(event.request, {cache:'reload'})` 強制略過 HTTP 快取，`CACHE_NAME` 同步升版 v1→v2 清掉已污染的快取。這是跟 `expense-tracker-pwa` 那次「install 階段 `cache.addAll()` 忘記加 `{cache:'reload'}`」同一個 bug class 的 runtime 版本，細節見 [[pwa-install-rollout]]。

## 正式部署

GitHub Pages，公開 repo <https://github.com/M255525/food-finder>，網址 <https://m255525.github.io/food-finder/>（`main` branch 名稱其實是 `master`，`/` 根目錄）。`README.md` 面向 GitHub 訪客，內容與 `manual.html` 有重疊但角色不同——README 是給「決定要不要用」的人看的簡短總覽，manual.html 是給「已經在用」的人看的完整操作手冊，兩邊都要維護但不用逐字同步。改完 `index.html`／`manual.html`／`mrt-stations-data.js` 後記得 `git push`（GitHub Pages 會自動重新部署，通常一兩分鐘內生效）。repo 是公開的，但只有程式碼——使用者的搜尋條件與 API 金鑰只存在自己瀏覽器的 localStorage，不會出現在 repo 裡。

**測試時注意**：用一把格式正確但無效的金鑰（例如 `sk-ant-fake-test-key`）對著真實服務商端點打，可以驗證錯誤處理路徑（例如 Claude 會回傳真實的 401 `invalid x-api-key`），不需要一把真的有額度的金鑰就能確認請求格式與錯誤訊息顯示正確；但無法驗證 AI 實際生成內容的品質與 JSON 格式穩定性，這部分需要一把真的金鑰實測。
