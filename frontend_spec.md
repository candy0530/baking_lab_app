# Baking Lab App - Frontend Specification (UI & Behavior)

This document outlines the user interface layout, user permissions, state management, interactive features, and calculation algorithms of the **T³ 減法實驗室 - 烘焙紀錄** frontend web application (`index.html`).

---

## 1. Overview & Technology Stack

- **Frameworks**: HTML5, Vanilla JavaScript (ES6+), Bootstrap 5.3 (CSS & JS bundle).
- **Styling**: Custom CSS with responsive layouts, modal overlays, card components, drag-and-drop styling, and role-based feature toggling.
- **Data Synchronization**: Asynchronous `fetch` calls against Google Apps Script Web App API URL (`db.gs`).

---

## 2. Authentication & Permission Control

The app uses `sessionStorage` (`key: baking_lab_auth`) to store login state.

### 2.1. Access Roles

#### **A. Visitor / Public Mode (Unauthenticated)**
- **Header**: Displays "🔑 登入" button.
- **Repository View (`#recordsContainer`)**:
  - Can view recipe accordions and past experiment logs.
  - Can search recipes using the top autocomplete search input.
  - Read-only fields: All form inputs inside `#admin-recipe-section` are disabled via CSS (`pointer-events: none`).
- **Hidden Features**:
  - Tasting notes (`result_notes`) in the summary accordion.
  - Total cost badges in accordion cards.
  - Save/Edit buttons for recipes and records.
  - Purchasing log section (`#admin-purchase-section`).

#### **B. Administrator / Member Mode (Authenticated)**
- **Header**: Displays "🚪 登出" button.
- **Unlocked Capabilities**:
  - Full editing and creation of recipes (`save_recipe`).
  - Full editing and creation of experiment records (`save_record`, `update_record`).
  - Drag-and-drop reordering of ingredients.
  - Recipe copy tools, Baker's % and weight conversion calculators.
  - Ingredient cost calculator with smart egg pricing.
  - Complete access to purchasing logs (`save_purchase`, `update_purchase`, `delete_purchase`).

### 2.2. Login Modal (`#login-overlay`)
- Opened via "🔑 登入" button or triggered programmatically.
- Enter key navigation (press Enter in username focuses password; press Enter in password submits form).
- Validates via API action `login`.

---

## 3. Core Modules & Features

### 3.1. Recipe Search & Autocomplete
- **Component**: `#recipe_name` input with `#autocomplete-results` floating list.
- **Behavior**:
  - On input or focus: Filters `dbRecipes` by string match (case-insensitive fuzzy search).
  - Clicking a result loads the recipe details into the editor (`loadRecipeDetails`).
  - "🗑️ 清除" button: Resets all recipe form inputs and selection states.

### 3.2. Multi-Section Recipe Editor
- **Container**: `#target-recipe-container` (Original) and `#actual-recipe-container` (Experiment).
- **Multi-Section Support**: Recipes can be partitioned into sections (e.g. `主麵團`, `中種`, `內餡`, `表面裝飾`).
- **Row Controls**:
  - **Drag Handle (☰)**: Drag-and-drop row reordering within a section.
  - **Inputs**: `食材` (Ingredient Name), `%` (Baker's Percentage), `克數` (Weight in grams), `品牌` (Brand name, optional).
- **Serialization Syntax**:
  Compiles multi-section UI rows into formatted string storage:
  ```markdown
  ## 主麵團
  高筋麵粉 | 100% | 500g | (日清高筋) | $120/1000g
  水 | 65% | 325g
  ```

### 3.3. Dual-Way Unit Calculators

#### **A. `g`算`%` (`calcPctFromGrams`)**
1. Reads ingredient gram weights.
2. Prompts user for 100% base flour total weight.
3. Computes `Baker's % = (Ingredient Weight / Base Weight) * 100`.

#### **B. `%`算`g` (`calcGramsFromPct`)**
1. Sums up all Baker's percentages.
2. Prompts user for expected total net weight (without waste).
3. Prompts user for loss ratio (0%, 5%, 10%).
4. Calculates gross total target weight: `Target Total = Expected Net * (1 + Loss Ratio / 100)`.
5. Calculates base weight unit: `Base Unit = Target Total / Sum of Percentages`.
6. Computes individual ingredient weight: `Weight = Math.round(Baker's % * Base Unit)`.

---

## 4. Baking Experiment Records Workflow

### 4.1. Recipe Target & Copy
- Selecting a recipe populates target temperatures (upper/lower), target baking time, and target preparation steps.
- **"⬇️ 複製配方" Button (`copyOriginalToActual`)**: Copies target recipe structure, temperatures, and times directly into the actual experiment form.

### 4.2. Experiment Record Form (`#recordForm`)
- **Fields**:
  - `experiment_purpose`: Purpose/Focus (with preset datalist options: `試做`, `減糖`, `減油`).
  - `actual_recipe`: Custom ingredient table for this specific bake.
  - `actual_temp_up` & `actual_temp_down`: Oven temperatures (°C).
  - `actual_time`: Baking time (minutes).
  - `oven_spec`: Oven equipment selection (`Panasonic 38L`, `中部電機`).
  - `yield_amount`: Output yield count.
  - `result_notes`: Taste test and texture evaluation.
  - `total_cost`: Automatically stored calculated ingredient cost.

### 4.3. Smart Ingredient Cost Calculator (`calculateCost`)
1. Fetches latest purchase history from `Purchases` sheet.
2. Matches each ingredient row with purchase logs by `name` and optional `brand`.
3. **Egg Pricing Conversion Algorithm (`resolveEggPricing`)**:
   - Handles shell eggs (`雞蛋`, `蛋`, `中型蛋`, etc.) and converts to specific egg parts:
     - **Whole Egg (全蛋/蛋液)**: Shell egg 60g yields ~50g liquid. Multiplier `(60 / 50)`.
     - **Egg Yolk (蛋黃/蛋黃液)**: Shell egg ~20g yields ~17g yolk. Multiplier `(20 / 17)`.
     - **Egg White (蛋白/蛋白液)**: Shell egg ~40g yields ~34g white. Multiplier `(40 / 34)`.
4. Renders itemized cost summary table with status badges (`table-warning` for missing purchase records).

---

## 5. Experiment Repository Accordion (`#recordsContainer`)

- Groups records by `recipe_name`.
- Displays experiment count badge per recipe.
- Shows standard recipe preparation steps (`target_steps`).
- Click card to enter edit mode (loads record into form and scrolls smoothly to editor).

---

- Form fields: `pur_ingredient`, `pur_brand`, `pur_weight`, `pur_price`, `pur_store`.
- Dynamic autocomplete datalists (`#ingredient-list`, `#brand-list`, `#store-list`):
  - `#ingredient-list` and `#store-list`: Populated from `Options` sheet and purchase history.
  - `#brand-list`: Dynamically generated via `updateBrandDatalist(ingredientName)` from `Purchases` sheet logs (`dbPurchases`) in reverse chronological order:
    - **Ingredient Match**: Queries `dbPurchases` for brand names associated with `ingredientName`.
    - **Egg Derivative Normalization**: When `ingredientName` is an egg derivative (`全蛋`, `蛋白`, `蛋黃`, `蛋液`, etc.), automatically maps and searches for brands recorded under shell eggs (`雞蛋`, `蛋`, `中型蛋`).
    - **Fallback Chain**: If no ingredient-specific purchase brand exists, returns all distinct brands in `dbPurchases`. If `dbPurchases` is empty, falls back to `Options` sheet brands.
- Full CRUD support: Create, Edit (populates form with cancel button), and Delete with confirmation prompt.
