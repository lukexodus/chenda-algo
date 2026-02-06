# Chenda - Perishable Goods E-Commerce Platform - Project Context

## Project Overview
Building an ecommerce platform for perishable goods that uses a **combined algorithm** integrating:
1. **Proximity Algorithm** - Haversine distance calculation
2. **Shelf Life Algorithm** - Percentage of remaining shelf life

## Core Features
- **Users**: Buyers and Sellers (unified user type with `type` field)
- **Products**: Created by sellers with expiration tracking
- **Algorithm**: Weighted scoring system (proximity + shelf life), user-adjustable
- **Filters**: Max radius (50km default), minimum freshness threshold (optional)
- **Display Modes**: Ranking (combined score) OR Filter+Sort (by price/distance/freshness)

## Tech Stack & Database
- **Database**: PostgreSQL + PostGIS for geospatial queries
- **Shelf Life Data**: USDA FoodKeeper database (catalog.data.gov/dataset/fsis-foodkeeper-data)
- **Development**: JavaScript objects for prototyping, migrate to PostgreSQL later

## Database Schema (PlantUML ERD Created)

```
Users (id, name, email, type, location POINT, preferences JSONB)
  ↓ 1:N
Products (id, seller_id, product_type_id, days_already_used, listed_date, price, quantity, location POINT)
  ↓ N:1
Product_Types (id, name, default_shelf_life_days)
```

## USDA FoodKeeper Data Structure

**Relevant Fields from Product Sheet:**
- Storage conditions: Pantry, Refrigerate, Freeze (each has Min/Max/Metric)
- After Opening (DOP prefix): DOP_Pantry, DOP_Refrigerate, DOP_Freeze
- Metrics: "Days", "Weeks", "Months" (need conversion to days)
- Example: Butter DOP_Refrigerate = 1-2 Months after opening

**Storage Condition Fallback Priority:**
1. DOP_Refrigerate (refrigerated after opening) - most common for perishables
2. Refrigerate (unopened refrigerated)
3. DOP_Pantry (pantry after opening)
4. Pantry (unopened pantry)
5. Freeze options (last resort)

## Algorithm Details

**Shelf Life Calculation:**
- Total shelf life = Product_Type.default_shelf_life_days (from USDA)
- Seller specifies: `days_already_used` (how much shelf life consumed before listing)
- Remaining % = (total - days_used) / total × 100
- Expired products filtered out completely

**Proximity Calculation:**
- Haversine formula (straight-line distance)
- Filter by max_radius (default 50km, user-adjustable)

**Combined Score:**
- Normalize proximity: 0-100 (closer = higher)
- Normalize freshness: 0-100 (fresher = higher)  
- Combined = (proximity_weight × proximity_score) + (shelf_life_weight × shelf_life_score)
- Default weights: 50/50, user-adjustable 0-100%

**User Controls:**
- Adjustable weights (proximity vs shelf life priority)
- Max delivery radius
- Optional minimum freshness threshold filter
- Toggle between ranking mode vs filter+sort mode

## Current Progress - Phase 1

**Completed:**
- [x] Task 1.1: Downloaded USDA FoodKeeper database
- [x] Task 1.2: Explored database structure

**Next Steps:**
- [ ] Task 1.3: Extract relevant fields from USDA data
- [ ] Task 1.4: Transform to product_types format (convert metrics to days, apply fallback logic)
- [ ] Task 1.5: Define user object structure
- [ ] Task 1.6: Define product object structure
- [ ] Task 1.7: Create mock data sets (5-10 users, 20-30 products)

**Pending Decision for Task 1.3-1.4:**
- Use MAX shelf life value or AVERAGE(min, max)? (e.g., Butter 1-2 months → 60 days or 45 days?)

## Full Implementation Plan (7 Phases)

1. **Phase 1**: Setup & Mock Data (in progress)
2. **Phase 2**: Core calculation functions (Haversine, shelf life %, expiration date)
3. **Phase 3**: Filtering logic (expired, radius, freshness threshold)
4. **Phase 4**: Scoring & ranking system (normalize, weight, combine)
5. **Phase 5**: Sorting & display logic (ranking vs filter+sort modes)
6. **Phase 6**: Main algorithm integration (pipeline: filter → score → sort)
7. **Phase 7**: Testing & validation (edge cases, weight adjustments, real USDA products)

## Key Design Decisions Made

- Sellers specify storage condition when listing (refrigerated/pantry/frozen)
- Sellers input `days_already_used` (not specific expiration date)
- System calculates actual expiration based on: listed_date + (total_shelf_life - days_used)
- Products past expiration are filtered out completely
- User location stored as coordinates {lat, lng}
- Mock data uses real USDA product types (milk, eggs, lettuce, meat, etc.)

---

**Status**: Ready to continue with Task 1.3 - Extract USDA data with storage condition fallback logic

# Chenda - Additional Context (Continuation)

## USDA Data Transformation Complete - New Findings

### Real Dataset Statistics
- **Actual USDA products**: 661 total
- **Successfully transformed**: 613 products (92.7%)
- **Perishables curated** (<30 days): 180 products
- **Skipped**: 48 products (no shelf life data)

### Storage Condition Reality (from real data)
```
refrigerated_opened    235 products (38.3%) - Most common
pantry_opened          149 products (24.3%)
refrigerated           128 products (20.9%)
pantry                  71 products (11.6%)
frozen_opened           21 products (3.4%)
frozen                   9 products (1.5%)
```

### Shelf Life Distribution (from real data)
```
0-7 days        230 products (37.5%) - Highly perishable
8-30 days        88 products (14.4%) - Perishable  
31-90 days       44 products (7.2%)
91-180 days      45 products (7.3%)
181-365 days    105 products (17.1%)
365+ days       101 products (16.5%)
```

### Implementation Details Confirmed

**Transformer Logic (working)**:
- Fallback chain successfully applied to 613 products
- Metrics converted: Months×30, Weeks×7, Days×1, Years×365
- Average calculation: (min+max)/2 rounded to days
- Unknown metrics detected: "Hours" (8 products - not supported)

**Data Quality Issues Found**:
1. 48 products have no shelf life data in any storage condition
2. Some products use "Hours" metric (not in our conversion table)
3. Metrics like "When Ripe", "Indefinitely", "Not Recommended" exist but aren't quantitative

**Files Created**:
- `product_types_full.json` - 613 products, complete catalog
- `product_types_perishables.json` - 180 products, ≤30 days shelf life, categories: Dairy(7), Meat(10), Poultry(15), Fruits(18), Vegetables(19)
- `usda_data_transformer.js` - Reusable transformation logic
- `shelf-life-core.json` - Source data from fsis.usda.gov/shared/data/EN/foodkeeper.json

### Sample Real Products (for mock data reference)
```
Highly Perishable (1-7 days):
- Cream (whipped): 1 day
- Eggs (raw whites/yolks): 3 days  
- Eggnog: 4 days
- Milk (opened): 6 days

Moderately Perishable (8-30 days):
- Buttermilk: 11 days
- Cottage cheese: 14 days
- Eggs (in shell): 28 days
- Shredded cheese: 30 days

Long shelf life:
- Parmesan (grated): 360 days
- Butter: 45 days
```

### Updated Data Structure (confirmed working)
```javascript
{
  id: 21,
  name: "Eggs",
  name_subtitle: "in shell",
  category_id: 7,
  keywords: "Eggs, fresh eggs",
  default_shelf_life_days: 28,  // AVERAGE(3,5) weeks = 28 days
  default_storage_condition: "refrigerated_opened",
  shelf_life_source: {
    min: 3,
    max: 5,
    metric: "Weeks"
  }
}
```

**Key Decision Confirmed**: Use `product_types_perishables.json` (180 products) for mock data generation in Task 1.7, as these are the most relevant items (≤30 days shelf life from Dairy, Meat, Poultry, Fruits, Vegetables categories).

---

# Chenda - Updated Context (Tasks 2.1-2.2)

## Phase 2 Progress: Core Calculation Functions

### ✅ Completed Tasks

**Task 2.1: Haversine Distance Calculator** ✓
- File: `haversine_calculator.js`
- 3 main functions + helpers
- Real Metro Manila coordinates tested
- Performance: 300 distances in 1ms

**Task 2.2: Shelf Life Calculator** ✓
- File: `shelf_life_calculator.js`
- 8 functions (calculations + filtering)
- All 30 mock products analyzed
- 0 expired products, 74.1% avg freshness

---

### Core Implementation Files
1. **haversine_calculator.js** - Distance calculations
   - `calculateDistance(point1, point2, unit='km')`
   - `calculateDistanceRounded(point1, point2, unit, decimals=2)`
   - `calculateDistanceBatch(origin, destinations, unit)`
   - Supports: km, miles, meters
   - Validation: lat (-90 to 90), lng (-180 to 180)

2. **shelf_life_calculator.js** - Freshness tracking
   - `calculateRemainingShelfLife(total, used)`
   - `calculateFreshnessPercent(total, used, decimals=2)`
   - `calculateExpirationDate(listedDate, remaining)`
   - `isExpired(expirationDate, currentDate)`
   - `calculateShelfLifeMetrics(product, currentDate)` - All-in-one
   - `calculateShelfLifeMetricsBatch(products, currentDate)`
   - `filterExpiredProducts(products, currentDate)`
   - `filterByFreshness(products, minPercent)`

### Integration Test Files
3. **haversine_integration_test.js** - 6 test scenarios with mock data
4. **shelf_life_integration_test.js** - 10 test scenarios with mock data

---

## Real Data Insights from Integration Tests

### Distance Statistics (Haversine)
- Total buyer-product pairs: 210
- Average distance: **9.64 km**
- Distance distribution:
  - 0-5 km: 26.2% (Very close)
  - 5-10 km: 35.7% (Moderate) ← Most common
  - 10-15 km: 19.0%
  - 15-20 km: 11.9%
  - 20-50 km: 7.1%

### Seller Coverage (Haversine)
- Juan's Fresh Market (Pasig): 6/7 buyers reachable (85.7%)
- Tindahan ni Aling Nena (Taguig): 7/7 buyers reachable (100%)
- Manila Bay Groceries (Manila): 6/7 buyers reachable (85.7%)
- Linda's Dairy Corner (Valenzuela): 6/7 buyers reachable (85.7%)

### Freshness Statistics (Shelf Life)
- Total products: 30
- Expired: 0
- Average freshness: **74.1%**
- Average days remaining: **11.2 days**
- Freshness breakdown:
  - ≥90% fresh: 3 products (10%)
  - ≥70% fresh: 20 products (66.7%)
  - <50% fresh: 1 product (3.3%)
  - Expiring within 3 days: 2 products

### Seller Performance (Shelf Life)
- Linda's Dairy Corner: 86.4% avg ⭐ Best
- Juan's Fresh Market: 74.4% avg
- Manila Bay Groceries: 72.6% avg
- Tindahan ni Aling Nena: 68.3% avg

### Product Type Analysis (Shelf Life)
- Freshest: Yogurt (90.9% avg)
- Lowest: Dips/Sour cream (64.3% avg)
- Most listings: Yogurt, Cheese (4 each)

---

## Key Technical Decisions Made

### Haversine Calculator
1. **Object parameters**: `{lat, lng}` format (matches data structure)
2. **Multiple units**: km (default), miles, meters
3. **Earth radius**: 6371 km (mean radius)
4. **Precision**: Full internally, 2 decimals for display
5. **Batch function**: 1-to-many optimization

### Shelf Life Calculator
1. **Dual approach**: Granular functions + all-in-one convenience
2. **Date handling**: Accept both Date objects and ISO 8601 strings
3. **Precision**: 2 decimals for freshness (82.14%)
4. **Separate filters**: Expired vs freshness threshold (composable)
5. **Validation**: Strict with descriptive error messages

---

## Algorithm Integration Points

### Current Usage Pattern
```javascript
// 1. Calculate distances (Task 2.1)
const distance = calculateDistance(buyer.location, product.location);

// 2. Calculate freshness metrics (Task 2.2)
const metrics = calculateShelfLifeMetrics({
  total_shelf_life_days: productType.default_shelf_life_days,
  days_already_used: product.days_already_used,
  listed_date: product.listed_date
});

// 3. Filter products
const active = filterExpiredProducts(products);
const inRange = active.filter(p => p.distance_km <= buyer.max_radius_km);
const fresh = filterByFreshness(inRange, buyer.min_freshness_percent);

// 4. Next: Normalize scores (Task 2.3)
// 5. Next: Calculate combined scores (Task 2.4)
```

---

## Next Phase Requirements

### Task 2.3: Score Normalization (Pending)
Need to implement:
- `normalizeProximityScore(distance_km, max_radius_km)` → 0-100
- `normalizeFreshnessScore(freshness_percent)` → 0-100 (already 0-100, may just return as-is)

### Task 2.4: Combined Score Calculator (Pending)
Need to implement:
- `calculateCombinedScore(proximity_score, freshness_score, proximity_weight, freshness_weight)` → 0-100
- Input: Two normalized scores + user weights
- Output: Weighted average

---

## Updated Architecture Notes

### Package Diagram
- **Application Layer**: Frontend Interface (coupled, no REST API)
- **Business Logic Layer**: Search Service, User Management, Ranking Algorithm
- **Data Access Layer**: Repositories, PostgreSQL + PostGIS
- **External Systems**: USDA FoodKeeper (one-time import)

### Component Diagram
- **Algorithm Component** contains 5 sub-components:
  1. Haversine Distance ✓ (Task 2.1)
  2. Shelf Life Calculator ✓ (Task 2.2)
  3. Score Calculator (Tasks 2.3-2.4)
  4. Filter Engine (Phase 3)
  5. Ranking Engine (Phase 4)

---
