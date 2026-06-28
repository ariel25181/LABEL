# Receipt App Refactor Specification

## Purpose

This specification defines a modular architecture and behavior contract for the current single-page receipt/product tracker app in `Downloads/index.html`.
It is intended as a base for refactors where the code is changed by adjusting the spec and mapping it into new modules, rather than patching the app piecemeal.

## Goals

- Separate responsibilities into focused modules.
- Keep the existing UI behavior while making code easier to maintain.
- Encapsulate receipt scanning and manual item entry with a common validation model.
- Use explicit state shape definitions so future changes can be driven by data structure updates.
- Preserve the current app flows: product entry, receipt scan, draft validation, purchase confirmation.

## High-level Architecture

### Suggested module boundaries

1. `state.js`
   - Global application state object
   - CRUD helpers for state updates
   - Persistence helpers for local/session storage

2. `ui.js`
   - DOM selectors and binding helpers
   - Safe rendering functions for each major screen/section
   - Toast and error UI utilities

3. `receipt.js`
   - Receipt scanner flow and state
   - Gemini image upload and response parsing
   - Per-item validation workflow
   - Draft save gating

4. `products.js`
   - Manual item entry and catalog search
   - `addItemToList`, `editChipItem`, `renderChips`
   - Autocomplete and name suggestion logic

5. `validation.js`
   - Status transitions for `pending`, `validating`, `validated`
   - Validation rule definitions
   - AI prompt contract and result parser

6. `api.js`
   - Supabase catalog lookup and session save functions
   - Gemini request wrapper
   - Generic fetch/error normalization

7. `main.js`
   - Initialization
   - Event wiring
   - High-level route and overlay control

### Why this split?

- `ui.js` is only responsible for DOM updates and event wiring.
- `state.js` prevents direct global variable mutation from business logic.
- `receipt.js` isolates all scanner-specific complexity.
- `products.js` centralizes item entry and chip rendering.
- `validation.js` allows the same validation rules to be reused for receipt items and manual items.

## Core Data Models

### Item

```js
{
  id: string,
  source: 'manual' | 'catalog' | 'receipt',
  product_id: string,
  product_name: string,
  category: string,
  size: string,
  unit_price: number,
  quantity: number,
  line_total: number,
  notes: string,
  status: 'pending' | 'validating' | 'validated',
  validationMessage: string,
  catalog_product_id?: string,
}
```

### ReceiptDraft

```js
{
  items: Item[],
  total: number,
  status: 'editing' | 'reviewing' | 'readyToSave',
  lastScanId: number,
}
```

### AppState

```js
{
  auth: { loggedIn: boolean, user: object | null },
  activeStore: Store | null,
  currentItems: Item[],
  receiptDraft: ReceiptDraft,
  config: { geminiApiKey: string, supabaseUrl: string, supabaseAnonKey: string },
  ui: { currentOverlay: string | null },
  session: { [storeId: string]: { store: Store, items: Item[], total: number } },
}
```

## UI Contract

### UI segments and element responsibilities

- `openReceiptScanner()` opens file picker and does not require Gemini key immediately.
- `receipt-input` is the hidden file input.
- `ov-receipt` is the receipt review overlay.
- `receipt-results` renders receipt line items with validation controls.
- `receipt-footer` contains `receipt-validate-btn` and `receipt-load-btn`.
- `items-added` and `chips-container` show manual/current item chips.
- `confirm-prods-btn` enables only when item list is valid for purchase.

### Expected behaviors

- A receipt scan can be started without a Gemini key, but a user-facing warning is shown if the key is missing when processing the photo.
- Receipt items are shown as editable rows.
- Each scanned item is validated one-by-one using AI.
- `receipt-load-btn` becomes enabled only after all scanned items are validated.
- Manual-added items share the same item `status` and tooltip/visual state semantics.
- The draft preview is visible once a receipt is present.

## Receipt Scanner Flow

1. User clicks camera button.
2. `openReceiptScanner()` resets scan state and opens `receipt-input`.
3. User selects or captures an image.
4. `handleReceiptPhoto()` uploads/resizes the image and calls Gemini.
5. Parsed items are normalized into `receiptDraft.items`.
6. The overlay shows the list of items with status dots.
7. `startReceiptValidation()` validates the next unvalidated item.
8. If an item is valid, it becomes `validated`; otherwise it stays `pending` with a message.
9. `receipt-load-btn` is enabled when `receiptDraft.items.every(item => item.status === 'validated')`.
10. `finalizeReceiptDraft()` transforms validated receipt items into `currentItems` and closes the overlay.

## Validation Semantics

### Validation status lifecycle

- `pending` = newly added / edited / not approved yet.
- `validating` = actively being checked by Gemini.
- `validated` = good to save and use in purchase.

### Shared validation requirement

- Manual entries should start as `pending` if free-form.
- Catalog-seeded manual entries can default to `validated` if they contain full product detail.
- Receipt-scanned items should always start `pending` and move to `validated` only after a successful validation pass.

### Draft gating rule

- The `save draft` action is only available when every receipt-scanned item is `validated`.
- A different action should be used to allow the user to keep the receipt draft open if there are still pending items.

## API Contracts

### `GeminiService`

- `scanReceiptImage(file: File): Promise<ScannedReceiptItem[]>`
- `validateItem(item: Item): Promise<ValidationResult>`
- `parseGeminiTextToItems(rawText: string): ScannedReceiptItem[]`

### `SupabaseService`

- `searchProducts(query: string): Promise<CatalogProduct[]>`
- `lookupNameSuggestions(prefix: string): Promise<CatalogProduct[]>`
- `saveStoreSession(storeId: string, items: Item[]): Promise<void>`

### Gemini prompt guarantees

- All Gemini prompts must ask for JSON-only responses.
- The code must sanitize and parse the first bracketed JSON object/array.
- If Gemini returns invalid JSON, the app shows an inline error and lets the user retry.

## Refactor Recipe

### Step 1: Define the new state object

- Replace scattered globals (`currentItems`, `scannedReceiptItems`, `receiptScanGen`) with a single `appState` object.
- Create helpers like `getState()`, `setState(partial)`, `updateState(path, value)`.

### Step 2: Move UI selectors into `ui.js`

- Create functions such as `getEl(id)` and `showOverlay(id)`.
- Build `renderReceiptResults(state.receiptDraft)` and `renderProductChips(state.currentItems)`.

### Step 3: Extract receipt workflow

- Move logic from `handleReceiptPhoto`, `renderReceiptResults`, `startReceiptValidation`, and `finalizeReceiptDraft` into `receipt.js`.
- Keep a small adapter in `index.html` that forwards DOM events.

### Step 4: Extract product entry logic

- Move `addItemToList`, `editChipItem`, `cancelEditItem`, `renderChips` into `products.js`.
- Use the item model defined above.

### Step 5: Standardize validation

- Create `validation.js` with:
  - `validateReceiptItem(item)`
  - `markItemPending(item, message)`
  - `isItemReadyForSave(item)`

### Step 6: Wire the app from `main.js`

- `init()` loads config, binds buttons, and calls `renderApp()`.
- `renderApp()` decides which overlay is active and updates counts.

## Recommended Refactor Milestones

1. Keep current UI and behavior unchanged.
2. Replace one global at a time with the new state object.
3. Extract and test the receipt scan flow in isolation.
4. Ensure `renderReceiptResults()` covers only scan overlay updates.
5. Convert manual item chip rendering to use shared status/tooltip styles.
6. Validate that `receipt-load-btn` is disabled until all items are validated.
7. Move Gemini prompt strings into named constants.

## Future-proofing

- Represent item status and validation state explicitly in the data model.
- Avoid inline string HTML generation where possible.
- Keep DOM entry points stable and use lightweight renderers.
- Use `data-*` attributes instead of `onclick` inline event handlers for new modules.
- Keep `receiptDraft` separate from `currentItems` until the user confirms.

## Minimal UI Contract for Receipt Scanner

- `openReceiptScanner()`
  - Should only trigger `receipt-input.click()`.
  - Must not early-return due to Gemini key absence.

- `handleReceiptPhoto(file)`
  - Should display the receipt overlay immediately.
  - If `geminiApiKey` is missing, show a warning block inside `receipt-results` and keep scan controls visible.

- `renderReceiptResults(items)`
  - Should render one row per `item`.
  - Each row must show a status dot, editable fields, and the line total.
  - The validate button must be visible unless the item list is empty.

- `startReceiptValidation()`
  - Should validate only one pending item at a time.
  - After completion, re-render the receipt overlay and enable save only when all items are validated.

- `finalizeReceiptDraft()`
  - Should copy validated receipt items into `currentItems` with `source: 'receipt'`.
  - Should clear `receiptDraft.items` and close `ov-receipt`.

## Key UX Rules

- The receipt scan button should remain accessible even if Gemini settings are missing.
- Validation must be per-item and explicit, not batch-only.
- Items created manually should behave like scanned items in the status UI.
- Save/draft actions should never permit incomplete scanned receipt items.

## Notes for implementers

- Treat `receiptDraft` as a first-class draft object instead of using `scannedReceiptItems` as a transient array.
- Use `receiptDraft.lastScanId` or similar to discard stale responses safely.
- Keep `updateScannedItem()` and `editScannedItem()` as pure state updaters with UI render calls.
- Avoid duplicating `closeReceiptScanner()` and `openReceiptScanner()` logic in multiple places.

---

## Quick start for refactoring by spec

1. Create a new `state.js` file and define `appState`.
2. Create `ui.js` with `renderReceiptOverlay(state)` and `renderProductSheet(state)`.
3. Create `receipt.js` that exports `scanReceiptPhoto`, `validateNextReceiptItem`, `saveReceiptDraft`.
4. Create `products.js` that exports `addProductItem`, `editProductItem`, `removeProductItem`.
5. Replace inline `onclick` call targets in the HTML progressively with functions that delegate into these modules.
6. Keep only enough inline JS in the HTML to preserve the app shell until the refactor is complete.

This specification is intentionally implementation-neutral: it defines the data shapes, module responsibilities, and expected flows that the app should obey during the refactor.