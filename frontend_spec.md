# 烘焙實驗室 App - 前端 UI 與行為規格說明書

本說明書詳細規範 **T³ 減法實驗室 - 烘焙紀錄** 前端 Web 應用程式 (`index.html`) 的使用者介面佈局、使用者權限、狀態管理、互動功能與計算演算法。

---

## 1. 概述與技術棧 (Overview & Technology Stack)

- **核心框架**：HTML5, Vanilla JavaScript (ES6+), Bootstrap 5.3 (CSS & JS bundle)。
- **樣式設計**：自訂 CSS，支援響應式佈局 (Responsive Layout)、彈出視窗對話框 (Modal Overlays)、卡片組件、拖拉排序樣式及基於角色的功能切換。
- **資料同步**：透過非同步 `fetch` 呼叫 Google Apps Script Web App API URL (`db.gs`)。

---

## 2. 身份驗證與權限控制 (Authentication & Permission Control)

本應用程式使用 `localStorage`（支援 `sessionStorage` 備援機制，鍵值為 `baking_lab_auth`）來跨瀏覽器分頁與工作階段維持登入狀態。

### 2.1. 存取角色

#### **A. 訪客 / 公開模式 (Visitor / Public Mode - 未驗證)**
- **頂部導覽列 (Header)**：顯示「🔑 登入」按鈕。
- **實驗紀錄庫檢視 (`#recordsContainer`)**：
  - 可檢視食譜摺疊面板與歷史實驗紀錄。
  - 可使用頂部自動補全搜尋框搜尋食譜。
  - 唯讀欄位：`#admin-recipe-section` 內的所有表單輸入框均被 CSS 停用 (`pointer-events: none`)。
- **隱藏功能**：
  - 摺疊面板摘要中的品嚐紀錄與評語 (`result_notes`)。
  - 摺疊面板卡片中的總成本標籤。
  - 食譜與紀錄的儲存/編輯按鈕。
  - 採買紀錄區塊 (`#admin-purchase-section`)。

#### **B. 管理員 / 會員模式 (Administrator / Member Mode - 已驗證)**
- **頂部導覽列 (Header)**：顯示「🚪 登出」按鈕。
- **解鎖功能**：
  - 完整食譜編輯與新增權限 (`save_recipe`)。
  - 完整實驗紀錄編輯與新增權限 (`save_record`, `update_record`)。
  - 食材拖拉重排順序 (Drag-and-Drop)。
  - 食譜複製工具、烘焙百分比 (`Baker's %`) 與重量轉換計算器。
  - 具備蛋品智慧換算的食材成本計算器。
  - 完整採買紀錄存取權限 (`save_purchase`, `update_purchase`, `delete_purchase`)。

### 2.2. 登入 Modal (`#login-overlay`)
- 點擊「🔑 登入」按鈕或透過程式觸發開啟。
- 支援 Enter 鍵快捷導航（於使用者名稱框按 Enter 聚焦至密碼框；於密碼框按 Enter 直接送出表單）。
- 呼叫 API Action `login` 進行驗證。

---

## 3. 核心模組與功能 (Core Modules & Features)

### 3.1. 食譜版本管理與搜尋自動補全
- **元件**：`#recipe_main_name`（主品項名稱）+ `#recipe_version_name`（版本名稱）搭配 `#autocomplete-results` 懸浮下拉選單。
- **食譜命名規範**：
  - `recipe_name` 組合格式為 `主品項名稱 - 版本名稱`（例如：`芋頭酥 - 經典原味`、`芋頭酥 - 減糖30%`）。
  - 輔助函式 `parseRecipeFullName` 與 `buildRecipeFullName` 可乾淨地拆解與組合食譜標題與版本標籤。
- **行為**：
  - 於 `#recipe_main_name` 輸入或聚焦時：依字串匹配過濾 `dbRecipes` 並顯示含版本標籤的結果清單。
  - 點擊搜尋結果：自動將該版本的食譜詳細內容載入至編輯器 (`loadRecipeDetails`)。
  - 「🗑️ 清除」按鈕：重置所有食譜輸入框與表單選擇狀態。

### 3.2. 食譜版本升格與基準對照 Modal
- **基準對照 Modal (`#baselineCompareModal`)**：
  - 於實驗表單點擊「👁️ 對照原始基準比例」按鈕。
  - 開啟彈出對話框，並排顯示原始基準烘焙百分比 (`Baseline %`) 與本次實驗烘焙百分比 (`Actual %`)，並自動標示百分點差異 (`%p`)。
- **升格實驗紀錄為主食譜版本 (`promoteRecordToRecipeVersion`)**：
  - 於當前實驗表單或歷史實驗卡片點擊「🌟 升格為經典配方版本」按鈕。
  - 跳出對話框提示輸入版本名稱（預設為 `${experiment_purpose} 版`）。
  - 自動根據實際執行的克數算出各區塊食材的烘焙百分比，並將其作為新的主食譜版本儲存至 `dbRecipes`。

### 3.3. 多分區食譜編輯器 (Multi-Section Recipe Editor)
- **容器**：`#target-recipe-container`（目標配方）與 `#actual-recipe-container`（實驗配方）。
- **多分區支援**：食譜可劃分為多個區塊（例如：`主麵團`、`中種`、`內餡`、`表面裝飾`）。
- **列控制項**：
  - **拖拉控制手把 (☰)**：單一區塊內的食材列拖拉重排。
  - **輸入框**：`食材` (名稱)、`%` (烘焙百分比)、`克數` (重量 g)、`品牌` (品牌名稱，選填)。
- **序列化語法**：
  將 UI 介面中的多分區食材列編譯為結構化文字進行儲存：
  ```markdown
  ## 主麵團
  高筋麵粉 | 100% | 500g | (日清高筋) | $120/1000g
  水 | 65% | 325g
  ```

### 3.4. 雙向單位計算器 (Dual-Way Unit Calculators)

#### **A. `g` 算 `%` (`calcPctFromGrams`)**
1. 讀取各食材的克數重量。
2. 提示使用者輸入 100% 基準麵粉的總重量。
3. 計算公式：`烘焙百分比 % = (食材重量 / 基準麵粉重量) * 100`。

#### **B. `%` 算 `g` (`calcGramsFromPct`)**
1. 加總所有食材的烘焙百分比。
2. 提示使用者輸入預期成品淨重（不含損耗）。
3. 提示使用者選擇損耗率 (0%, 5%, 10%)。
4. 計算總目標毛重：`目標總重 = 預期淨重 * (1 + 損耗率 / 100)`。
5. 計算基準單位重量：`基準單位重量 = 目標總重 / 百分比總和`。
6. 計算個別食材重量：`食材重量 = Math.round(食材百分比 % * 基準單位重量)`。

### 3.5. 剩餘食材回推計算器 (`#remainingScaleModal`)
- **入口**：點擊原始食譜或實驗食譜編輯器中的「⚖️ 剩餘食材回推」按鈕。
- **流程與演算法**：
  1. 解析當前食譜編輯器中的食材與基準重量。
  2. 渲染食材清單並提供剩餘庫存克數的輸入框。
  3. 計算個別食材的縮放比例：`Scale_i = 剩餘克數_i / 基準重量_i`。
  4. 找出瓶頸食材（最小比例）：`MinScale = min(Scale_i)`。
  5. 計算預估可做產量：`預估產量 = 原始產量 * MinScale`。
  6. 計算所有區塊食材按比例縮放後的所需克數：`縮放重量 = Math.round(基準重量 * MinScale * 10) / 10`。
  7. **「一鍵套用至配方」**：將縮放後的重量更新至當前食譜編輯器的 `.ing-weight` 欄位中。

### 3.6. 訂單與多分區材料需求計算器 (`#orderCalculatorModal`)
- **入口**：點擊頂部導覽列的「📦 訂單試算」。
- **單顆/單份產量與規格控制**：
  - `order_target_count`：總目標製作份數（例如 `50 顆`）。
  - `order_unit_name`：自訂單位名稱（`顆`、`盒`、`份`）。
  - 區塊單顆重量 (`order_sec_piece_gram`)：指定每個區塊每顆所需的精確重量（例如：塔皮 `35.0g/顆`、檸檬餡 `40.0g/顆`）。
- **成分比例與單區塊用量規格 tracking**：
  - **即時區塊比例與規格標籤**：於每個區塊卡片下方動態渲染食材比例 (%) 與單顆用量 (`例如：低筋麵粉 57.1% (20.0g/顆)`)。
  - **材料加總彙整演算法**：
    - `區塊總所需重量 = 目標份數 * 單顆重量`。
    - `食材所需重量 = 區塊總所需重量 * (食材基準重量 / 區塊基準總重)`。
    - 合併跨區塊的相同食材名稱：`食材總重量 = Sum(食材所需重量)`。
    - 結合最新採買紀錄 (`dbPurchases`) 與蛋品計價演算法 (`resolveEggPricing`) 計算預估食材總成本。
- **詳細訂單產品區塊規格標籤 (📋 訂單產品區塊規格標籤)**：
  - 渲染詳細區塊卡片與表格，展示：`食材名稱`、`成分比例 (%)`、`單顆/單份用量 (g/顆)` 與 `全案總需重量 (g)`。
- **雲端同步 (`Orders` 工作表)**：
  - 儲存訂單資料，包含：`order_name`、`recipe_name`、`sections_json`（包含目標份數、單位名稱、區塊單顆克數與食材規格）、`total_materials_json`（包含完整規格明細）與 `note`。
  - 透過 API Actions `save_order`, `update_order`, `delete_order` 支援完整 CRUD。
- **「📋 複製完整採買與規格明細」**：一鍵產生並複製包含訂單規格（比例與單顆重量）及總採買彙整需求的完整文字說明。

---

## 4. 烘焙實驗紀錄流程 (Baking Experiment Records Workflow)

### 4.1. 食譜目標與複製
- 選擇食譜時，系統會自動填入目標上下火溫度、目標烘焙時間與目標製作步驟。
- **「⬇️ 複製配方」按鈕 (`copyOriginalToActual`)**：將目標食譜結構、溫度與時間直接複製至當前實驗表單中。

### 4.2. 實驗紀錄表單 (`#recordForm`)
- **欄位**：
  - `experiment_purpose`：實驗目的/重點（提供預設下拉選項：`試做`、`減糖`、`減油`）。
  - `actual_recipe`：本次烘焙實驗的實作食材表格。
  - `actual_temp_up` 與 `actual_temp_down`：烤箱上下火溫度 (°C)。
  - `actual_time`：烘焙時間（分鐘）。
  - `oven_spec`：烤箱設備選擇（`Panasonic 38L`、`中部電機`）。
  - `yield_amount`：實際成品數量。
  - `result_notes`：品嚐與口感質地紀錄。
  - `total_cost`：自動計算並儲存的食材總成本。

### 4.3. 智慧食材成本計算器 (`calculateCost`)
1. 從 `Purchases` 工作表擷取最新採買紀錄。
2. 比對每個食材列的 `名稱` 與選填的 `品牌`。
3. **蛋品計價轉換演算法 (`resolveEggPricing`)**：
   - 處理帶殼蛋（`雞蛋`、`蛋`、`中型蛋` 等），並精確轉換為特定蛋品部位：
     - **全蛋 / 蛋液**：帶殼蛋 60g 約可得 50g 蛋液，換算倍率 `(60 / 50)`。
     - **蛋黃 / 蛋黃液**：帶殼蛋 ~20g 蛋黃約可得 17g 淨蛋黃，換算倍率 `(20 / 17)`。
     - **蛋白 / 蛋白液**：帶殼蛋 ~40g 蛋白約可得 34g 淨蛋白，換算倍率 `(40 / 34)`。
4. 渲染分項成本明細表，並標示狀態（若無採買紀錄則顯示 `table-warning` 警告標籤）。

---

## 5. 實驗紀錄庫摺疊面板 (`#recordsContainer`)

- 按 `recipe_name` 食譜名稱進行分組。
- 顯示各食譜對應的實驗紀錄數量標籤。
- 展示標準食譜製作步驟 (`target_steps`)。
- 點擊紀錄卡片可進入編輯模式（自動將紀錄載入至表單並平滑捲動至編輯區域）。

---

## 6. 採買紀錄管理 (`#purchaseModal`)

- **Modal 存取入口**：可隨時透過以下途徑開啟：
  - 頂部導覽列按鈕 (`🛋️ 購買紀錄`)。
  - 成本計算摘要卡片按鈕 (`🛠️ 開啟購買紀錄管理`)。
  - 未標價食材警示旁的按鈕 (`➕ 補新增`)，點擊時會自動預填食材名稱與品牌。
- **表單欄位**：`pur_ingredient` (食材)、`pur_brand` (品牌)、`pur_weight` (重量)、`pur_price` (價格)、`pur_store` (購買店家)。
- **動態自動補全下拉選單** (`#ingredient-list`, `#brand-list`, `#store-list`)：
  - `#ingredient-list` 與 `#store-list`：取自 `Options` 工作表與歷史採買紀錄。
  - `#brand-list`：由 `updateBrandDatalist(ingredientName)` 根據 `Purchases` 工作表紀錄 (`dbPurchases`) 按時間倒序動態產生：
    - **食材匹配**：查詢 `dbPurchases` 中與 `ingredientName` 相關的品牌名稱。
    - **蛋品衍生詞正規化**：當 `ingredientName` 為蛋品衍生詞（`全蛋`、`蛋白`、`蛋黃`、`蛋液`等）時，自動對照並搜尋帶殼蛋（`雞蛋`、`蛋`、`中型蛋`）的採買紀錄。
    - **備援機制**：若查無該食材特定品牌，則回傳 `dbPurchases` 中所有不重複的品牌；若 `dbPurchases` 為空，則採用 `Options` 工作表預設品牌。
- **完整 CRUD 支援**：新增、編輯（載入內容至 Modal 表單）與刪除（附帶確認提示）。
