# 烘焙實驗室 App - 資料庫與 API 規格說明書 (Database & API Specifications)

本文件詳細說明 **T³ 減法實驗室 - 烘焙紀錄** 應用程式所採用的 Google Sheets 資料庫欄位結構以及 Google Apps Script (GAS) 的 HTTP API 介面規格。

---

## 1. 系統架構說明 (System Architecture)

本應用程式採用無伺服器 (Serverless) 架構，後端由 **Google Apps Script (GAS)** 驅動，作為 API 閘道器與作為主資料庫的 **Google 試算表 (Google Spreadsheet)** 進行非同步資料傳輸。

- **前端用戶端**：`index.html`（透過 `fetch(API_URL, ...)` 進行非同步請求）
- **後端 API**：`db.gs`（部署為 Google Apps Script Web App）
- **主資料庫**：Google Sheets（包含工作表：`Users`、`Recipes`、`Records`、`Purchases`、`Options` 與 `Orders`）

---

## 2. 資料庫欄位結構 (Database Schema)

目前的 Google 試算表中包含以下工作表及其欄位架構：

### 2.1. `Users` 工作表
儲存系統管理員之登入帳號與密碼。

| 欄位 | 標題 (Header) | 資料型態 | 說明 |
|---|---|---|---|
| A | `username` | 字串 (String) | 使用者登入帳號 |
| B | `password` | 字串 (String) | 登入密碼 (明碼) |

### 2.2. `Recipes` 工作表
儲存經典 master 配方與配方版本 (例如 `芋頭酥 - 經典原味`、`芋頭酥 - 減糖30%`)。若儲存相同配方名稱，則進行覆蓋更新。

| 欄位 | 標題 (Header) | 資料型態 | 說明 |
|---|---|---|---|
| A | `recipe_name` | 字串 (String) | 配方唯一識別名稱，格式為 `主品項 - 版本名稱`（如 `芋頭酥 - 經典原味`）或純主品項 |
| B | `target_recipe` | 字串 (String) | 依 Baker's % 烘焙百分比與基準克數序列化之食材明細字串 |
| C | `target_temp_up` | 數字 (Number) | 預設烤箱上火溫度 (°C) |
| D | `target_temp_down` | 數字 (Number) | 預設烤箱下火溫度 (°C) |
| E | `target_time` | 數字 (Number) | 預設烘焙時間 (分鐘) |
| F | `target_steps` | 字串 (String) | 標準製作步驟說明文字 |

### 2.3. `Records` 工作表
儲存每次實際烘焙實驗與製作的紀錄細節。

| 欄位 | 標題 (Header) | 資料型態 | 說明 |
|---|---|---|---|
| A | `id` | 字串 (String) | 實驗紀錄唯一 ID (時間戳記或 UUID) |
| B | `date` | 字串 (String) | 烘焙實驗日期時間 (YYYY/MM/DD HH:mm:ss) |
| C | `recipe_name` | 字串 (String) | 本次實驗所基於的經典配方名稱 |
| D | `actual_recipe` | 字串 (String) | 本次實驗實際秤重使用的食材克數明細 |
| E | `actual_temp_up` | 數字 (Number) | 實際烤箱上火溫度 (°C) |
| F | `actual_temp_down` | 數字 (Number) | 實際烤箱下火溫度 (°C) |
| G | `actual_time` | 數字 (Number) | 實際烘焙時間 (分鐘) |
| H | `experiment_purpose` | 字串 (String) | 本次實驗目的/任務 (如 `減糖30%`, `顧客訂單`, `早餐`) |
| I | `yield_amount` | 字串/數字 | 本次成品產出數量 |
| J | `result_notes` | 字串 (String) | 測試結果評估、口感心得與風味紀錄 |
| K | `total_cost` | 數字 (Number) | 本次烘焙所使用的食材預估總成本 ($) |
| L | `oven_spec` | 字串 (String) | 所使用的烤箱設備規格 (如 `Panasonic 38L`, `中部電機`) |

### 2.4. `Purchases` 工作表
記錄食材採買與進貨紀錄，用於精算烘焙成本。

| 欄位 | 標題 (Header) | 資料型態 | 說明 |
|---|---|---|---|
| A | `id` | 字串 (String) | 採買紀錄唯一 ID |
| B | `date` | 字串 (String) | 採買日期 (YYYY/MM/DD) |
| C | `ingredient` | 字串 (String) | 食材名稱 |
| D | `weight` | 數字 (Number) | 採買包裝重量/數量 (克數 g) |
| E | `price` | 數字 (Number) | 採買金額 ($) |
| F | `brand` | 字串 (String) | 食材品牌 |
| G | `store` | 字串 (String) | 購買店家/通路 |

### 2.5. `Options` 工作表
包含前端下拉選單預設之常用選單項目（如食材名稱、品牌、店家）。

| 欄位 | 標題 (Header) | 資料型態 | 說明 |
|---|---|---|---|
| - | `ingredient` | 字串 (String) | 預設常用食材名稱 |
| - | `brand` | 字串 (String) | 預設常用食材品牌 |
| - | `store` | 字串 (String) | 預設常用店家名稱 |

### 2.6. `Orders` 工作表
儲存訂單需求試算紀錄、各區塊單顆份量規格與全案材料反推加總文字。

| 欄位 | 標題 (Header) | 資料型態 | 說明 |
|---|---|---|---|
| A | `id` | 字串 (String) | 訂單紀錄唯一 ID |
| B | `date` | 字串 (String) | 訂單計算日期時間 (YYYY/MM/DD HH:mm:ss) |
| C | `order_name` | 字串 (String) | 訂單名稱 / 客戶名稱 |
| D | `recipe_name` | 字串 (String) | 對應的品項配方名稱 |
| E | `sections_json` | 字串 (String) | 包含目標總顆數、單位與各區塊單顆克數的 JSON 字串 |
| F | `total_materials_json` | 字串 (String) | 包含全案採買清單與區塊規格明細的文字 JSON 字串 |
| G | `note` | 字串 (String) | 訂單備註事項 |

---

## 3. 後端 API 規格說明 (`db.gs`)

所有 API 請求均由 `doGet` 與 `doPost` 進入點進行處理。

### 3.1. 讀取完整資料庫狀態 (`GET /`)

一次性取得資料庫所有工作表內容，解析為 JSON 陣列。回傳的資料列均已按 **時間倒序** 進行排序（最新資料在最前）。

- **請求 HTTP 方法**：`GET`
- **回應格式**：`application/json`
- **回應 JSON 範例**：
  ```json
  {
    "recipes": [...],
    "records": [...],
    "purchases": [...],
    "options": [...],
    "orders": [...]
  }
  ```

---

### 3.2. 寫入與更新資料庫 (`POST /`)

- **請求 HTTP 方法**：`POST`
- **請求標頭**：`Content-Type: application/json`
- **基本成功回應**：
  ```json
  {
    "status": "success"
  }
  ```
- **基本失敗回應**：
  ```json
  {
    "status": "error",
    "message": "<錯誤詳細資訊>"
  }
  ```

#### 動作 (Action): `login`
驗證使用者登入帳號密碼。
- **請求 Payload**：
  ```json
  {
    "action": "login",
    "username": "<帳號>",
    "password": "<密碼>"
  }
  ```
- **回應說明**：
  - 成功：`{"status": "success", "message": "login_ok"}`
  - 失敗：`{"status": "error", "message": "帳號或密碼錯誤"}`

#### 動作 (Action): `save_recipe`
建立或更新經典配方列。若 `recipe_name` 已存在則直接覆蓋更新。
- **請求 Payload**：
  ```json
  {
    "action": "save_recipe",
    "recipe_name": "芋頭酥 - 經典原味",
    "target_recipe": "## 主麵團\n高筋麵粉 | 100% | 500g...",
    "target_temp_up": 180,
    "target_temp_down": 180,
    "target_time": 30,
    "target_steps": "1. 麵粉過篩..."
  }
  ```

#### 動作 (Action): `save_record`
新增一筆全新的烘焙實驗紀錄。
- **請求 Payload**：
  ```json
  {
    "action": "save_record",
    "id": "rec_123456",
    "date": "2026-09-12 14:00:00",
    "recipe_name": "芋頭酥 - 經典原味",
    "actual_recipe": "## 主麵團\n高筋麵粉 | 500g...",
    "actual_temp_up": 175,
    "actual_temp_down": 175,
    "actual_time": 28,
    "experiment_purpose": "減糖30%實驗",
    "yield_amount": "20顆",
    "result_notes": "甜度適中，口感酥脆",
    "total_cost": 150,
    "oven_spec": "Panasonic 38L"
  }
  ```

#### 動作 (Action): `update_record`
依 `id` 更新既有的烘焙實驗紀錄，保留原有的 `id` 與建立日期。
- **請求 Payload**：
  ```json
  {
    "action": "update_record",
    "id": "rec_123456",
    "actual_recipe": "...",
    "actual_temp_up": 175,
    "actual_temp_down": 175,
    "actual_time": 28,
    "experiment_purpose": "...",
    "yield_amount": "...",
    "result_notes": "...",
    "total_cost": 150,
    "oven_spec": "..."
  }
  ```

#### 動作 (Action): `save_purchase`
新增一筆進貨採買紀錄。
- **請求 Payload**：
  ```json
  {
    "action": "save_purchase",
    "id": "pur_123456",
    "date": "2026-09-12",
    "ingredient": "無鹽奶油",
    "weight": 500,
    "price": 180,
    "brand": "發酵奶油",
    "store": "烘焙材料行"
  }
  ```

#### 動作 (Action): `update_purchase`
依 `id` 更新既有的採買紀錄。
- **請求 Payload**：
  ```json
  {
    "action": "update_purchase",
    "id": "pur_123456",
    "ingredient": "無鹽奶油",
    "weight": 500,
    "price": 185,
    "brand": "發酵奶油",
    "store": "烘焙材料行"
  }
  ```

#### 動作 (Action): `delete_purchase`
依 `id` 刪除採買紀錄。
- **請求 Payload**：
  ```json
  {
    "action": "delete_purchase",
    "id": "pur_123456"
  }
  ```

#### 動作 (Action): `save_order`
新增一筆訂單需求與材料反推計算紀錄。
- **請求 Payload**：
  ```json
  {
    "action": "save_order",
    "id": "ord_123456",
    "date": "2026-09-12 14:00:00",
    "order_name": "2026中秋訂單 (張小姐)",
    "recipe_name": "芋頭酥 - 經典原味",
    "sections_json": "{\"target_count\":50,\"unit_name\":\"顆\",\"sections\":[{\"section_title\":\"油皮\",\"piece_gram\":35}]}",
    "total_materials_json": "【2026中秋訂單】\n...",
    "note": "預計9/20交貨"
  }
  ```

#### 動作 (Action): `update_order`
依 `id` 更新既有的訂單計算紀錄。
- **請求 Payload**：
  ```json
  {
    "action": "update_order",
    "id": "ord_123456",
    "order_name": "...",
    "recipe_name": "...",
    "sections_json": "...",
    "total_materials_json": "...",
    "note": "..."
  }
  ```

#### 動作 (Action): `delete_order`
依 `id` 刪除訂單紀錄。
- **請求 Payload**：
  ```json
  {
    "action": "delete_order",
    "id": "ord_123456"
  }
  ```
