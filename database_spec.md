# Baking Lab App - Database & API Specifications

This document outlines the database schema (Google Sheets) and the HTTP API specifications (Google Apps Script) used by the Baking Lab application.

---

## 1. System Architecture

The application uses a serverless backend powered by **Google Apps Script (GAS)**, which acts as an API gateway communicating with a **Google Spreadsheet** as the primary database.

- **Frontend Client**: `index.html` (interacts via `fetch(API_URL, ...)`)
- **Backend API**: `db.gs` (deployed as a Web App in Google Apps Script)
- **Database**: Google Sheets (containing sheets: `Users`, `Recipes`, `Records`, `Purchases`, and `Options`)

---

## 2. Database Schema (Google Sheets)

The active Google Spreadsheet contains the following sheets and column structures:

### 2.1. `Users` Sheet
Stores authentication credentials for the application.

| Column | Header | Type | Description |
|---|---|---|---|
| A | `username` | String | User's login ID / username |
| B | `password` | String | Plaintext password |

### 2.2. `Recipes` Sheet
Stores standard target recipes. If a recipe with the same name is saved, it is overwritten.

| Column | Header | Type | Description |
|---|---|---|---|
| A | `recipe_name` | String | The unique name of the recipe (Primary Key) |
| B | `target_recipe` | String | Ingredients and ratio details (e.g., text, JSON, or markdown list) |
| C | `target_temp_up` | Number | Target oven upper heating temperature (°C) |
| D | `target_temp_down` | Number | Target oven lower heating temperature (°C) |
| E | `target_time` | Number | Target baking duration (minutes) |
| F | `target_steps` | String | Description of standard preparation steps |

### 2.3. `Records` Sheet
Stores details of actual baking experiments/sessions.

| Column | Header | Type | Description |
|---|---|---|---|
| A | `id` | String | Unique record identifier (e.g. timestamp or UUID) |
| B | `date` | String | Date of the baking experiment (YYYY-MM-DD) |
| C | `recipe_name` | String | The standard recipe name this record is based on |
| D | `actual_recipe` | String | Ingredients and ratios actually used |
| E | `actual_temp_up` | Number | Actual oven upper heating temperature (°C) |
| F | `actual_temp_down` | Number | Actual oven lower heating temperature (°C) |
| G | `actual_time` | Number | Actual baking duration (minutes) |
| H | `experiment_purpose` | String | The main objective/purpose of this specific bake (e.g. adjustment notes) |
| I | `yield_amount` | String/Number | The output yield quantity |
| J | `result_notes` | String | Post-baking evaluation, taste notes, and sensory results |
| K | `total_cost` | Number | Total cost calculated for the ingredients used |
| L | `oven_spec` | String | Identifier or specifications of the oven used |

### 2.4. `Purchases` Sheet
Tracks ingredient purchase logs to help estimate inventory costs.

| Column | Header | Type | Description |
|---|---|---|---|
| A | `id` | String | Unique purchase identifier |
| B | `date` | String | Date of purchase (YYYY-MM-DD) |
| C | `ingredient` | String | Ingredient name |
| D | `weight` | Number | Weight/quantity purchased (e.g. in grams) |
| E | `price` | Number | Purchase price (currency amount) |
| F | `brand` | String | Brand of the ingredient |
| G | `store` | String | Store where purchased |

### 2.5. `Options` Sheet
Contains predefined option configurations for drop-down lists (e.g., matching ingredients, brands, stores) used by the UI.

| Column | Header | Type | Description |
|---|---|---|---|
| - | `ingredient` | String | Preset ingredient names |
| - | `brand` | String | Preset ingredient brands |
| - | `store` | String | Preset purchase stores |

---

## 3. Backend API Specifications (`db.gs`)

All API interactions are served by `doGet` and `doPost` entry points.

### 3.1. Fetch Database State (`GET /`)

Retrieves the complete state of the database, parsed into JSON arrays. Data rows are returned in **reverse chronological order** (most recent row first).

- **Method**: `GET`
- **Response Format**: `application/json`
- **Response Body**:
  ```json
  {
    "recipes": [
      {
        "recipe_name": "...",
        "target_recipe": "...",
        "target_temp_up": 180,
        "target_temp_down": 180,
        "target_time": 25,
        "target_steps": "..."
      }
    ],
    "records": [...],
    "purchases": [...],
    "options": [...]
  }
  ```

---

### 3.2. Update/Write Database (`POST /`)

- **Method**: `POST`
- **Request Headers**: `Content-Type: application/json`
- **Base Response (Success)**:
  ```json
  {
    "status": "success"
  }
  ```
- **Base Response (Error)**:
  ```json
  {
    "status": "error",
    "message": "<Error Details>"
  }
  ```

#### Action: `login`
Verifies user credentials.
- **Request Payload**:
  ```json
  {
    "action": "login",
    "username": "<username>",
    "password": "<password>"
  }
  ```
- **Custom Responses**:
  - Success: `{"status": "success", "message": "login_ok"}`
  - Failure: `{"status": "error", "message": "帳號或密碼錯誤"}`

#### Action: `save_recipe`
Creates or updates a recipe row. Overwrites if the `recipe_name` already exists.
- **Request Payload**:
  ```json
  {
    "action": "save_recipe",
    "recipe_name": "...",
    "target_recipe": "...",
    "target_temp_up": 180,
    "target_temp_down": 180,
    "target_time": 30,
    "target_steps": "..."
  }
  ```

#### Action: `save_record`
Appends a new baking experiment record.
- **Request Payload**:
  ```json
  {
    "action": "save_record",
    "id": "rec_123456",
    "date": "2026-08-01",
    "recipe_name": "...",
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

#### Action: `update_record`
Updates an existing baking experiment record by `id`. Preserves the original `id`, `date`, and `recipe_name`.
- **Request Payload**:
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

#### Action: `save_purchase`
Appends a new purchase log.
- **Request Payload**:
  ```json
  {
    "action": "save_purchase",
    "id": "pur_123456",
    "date": "2026-08-01",
    "ingredient": "...",
    "weight": 1000,
    "price": 80,
    "brand": "...",
    "store": "..."
  }
  ```

#### Action: `update_purchase`
Updates an existing purchase entry by `id`.
- **Request Payload**:
  ```json
  {
    "action": "update_purchase",
    "id": "pur_123456",
    "ingredient": "...",
    "weight": 1000,
    "price": 85,
    "brand": "...",
    "store": "..."
  }
  ```

#### Action: `delete_purchase`
Deletes a purchase entry by `id`.
- **Request Payload**:
  ```json
  {
    "action": "delete_purchase",
    "id": "pur_123456"
  }
  ```
