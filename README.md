# Dashboard Requirements

### ✅ **1. Additional Data Points for the Dashboard**

In addition to the ones you've listed, consider adding these:

| Category | Metric | Purpose |
| --- | --- | --- |
| **Stock & Inventory** | Opening Stock (Yesterday) | To compare with today’s values for drift detection |
|  | Closing Stock (Today) | Forecasting, planning, and alerts |
|  | Reorder Alert / Low Stock Indicator | Operational decision-making |
| **Purchases** | Purchase Frequency (Last 7 days) | Identify demand pattern |
|  | Avg Purchase Cost per Unit | Track cost drift over time |
| **Prep & Yield** | Prep Frequency | Understand production patterns |
|  | Standard vs Actual Yield (%) | Benchmark actuals vs targets |
| **Sales** | Dish-level Consumption (Top 5) | Ingredient-to-dish mapping |
| **Waste & Loss** | Estimated Theoretical Waste | Based on yield and sales |
|  | Unaccounted Waste (Expected - Actual) | Detect theft, misreporting, etc. |
| **Performance** | Stock Turnover Rate | How efficiently stock is used |
|  | Cost of Waste (₹) | Translate weight to ₹ for impact |

---

### ✅ **2. Backend Logic / Trajectory of Each Data Point**

| Metric | Backend Logic |
| --- | --- |
| Total Stock Unprepared | `Opening stock + Purchases - Prep Quantity - Spoilage` |
| Total Stock Prepared | Sum from Production Prep sheet (ingredient-wise) |
| Last Purchase | Latest entry in Purchase sheet based on date |
| Dishes Sold Yesterday | Count of dishes (from Sales sheet) grouped by yesterday’s date |
| Total Quantity Sold | Sum of ingredient usage from dish-to-ingredient mapping (if available) |
| Average Yield | `(Total Output from Prep) / (Total Input Raw Material)` over last X days |
| Spoiled Waste | From Waste Sheet or inferred from unaccounted delta (input - output - sales quantity) |
| Low Stock Alert | Triggered when `Stock Unprepared < X` where X is threshold |
| Estimated Waste | `Expected Prep - Actual Prep` based on defined yield benchmarks |
| Cost of Waste | `Spoiled Quantity * Average Cost per Unit` from recent purchases |

---

### ✅ 3**. Where Should You Store the 3 Datasets?**

| Option | Why Use It |
| --- | --- |
| **Google Sheets** (current) | Good for prototyping, editable by non-devs. Use gspread or Sheets API to access |
| **BigQuery** | Recommended for production. Easy SQL queries, great for large data volumes |
| **Firestore / Realtime DB** | Good for mobile UI sync, but not optimal for complex reconciliation logic |
| **PostgreSQL / MySQL** | Best for relational data and joins, if running on your own infra |

**Recommended**:

- Continue reading daily from Google Sheets → Load to **BigQuery** using scheduled ETL (Python + Cloud Scheduler or Apps Script).
- Run reconciliation queries in BigQuery.
- Serve results via API to the dashboard frontend.

---

### ✅ 4**. Should You Use AI Workflows or Agents?**

**Not for dashboard logic itself**, but **yes** in specific areas:

| Use Case | Recommendation |
| --- | --- |
| Predict missing entries | AI model (lightweight regression/ML) |
| Alert for yield anomalies | AI/ML model trained on past trends |
| Auto-fill ingredient-to-dish map | CLIP-based model or NLP assistance |
| Natural language querying | RAG system (LangChain + Sheets/SQL) |
| Conversational UI ("What was wasted yesterday?") | AI agent + LLM for querying data |

So:

- Use **traditional logic + SQL** for core reconciliation.
- Use **AI/Agents** for insights, assistance, anomaly detection, and conversational UX.

# Backend Logics

Absolutely! Here's the **enhanced backend logic table** including two new columns: **Data Source** (which sheet/table to extract from) and **Visualization** (suggested type of visual representation).

---

### 📦 Stock or Purchase Related Data Points

| **Metric** | **Definition** | **Backend Logic / Formula** | **Data Source** | **Visualization** |
| --- | --- | --- | --- | --- |
| Current Stock | How much is in the storage now | `Total Purchases - Total Prepped - Total Waste` | Purchase Sheet, Prep Sheet, Waste Sheet | Static Number or Line Trend |
| Prepared Stock | How much was prepped | `SUM(Prep Quantity)` | Production Prep Sheet | Static Number |
| Last Buy | Last date of purchase | `MAX(Purchase Date)` | Purchase Sheet | Static Date Text |
| Stock Period | How long item has been in storage | `Today - Last Purchase Date` | Purchase Sheet | Static Number (days) |

---

### 🍳 Prep Related Data Points

| **Metric** | **Definition** | **Backend Logic / Formula** | **Data Source** | **Visualization** |
| --- | --- | --- | --- | --- |
| Average Yield | Purchase to prep efficiency (%) | `(Total Prepped / Total Purchased) × 100` over a date range | Purchase Sheet, Prep Sheet | Line Chart or Gauge |
| Spoiled Waste | Material wasted after prep | `Purchased - (Prepped + Stock)` | Purchase, Prep, Stock | Static Number or Bar Graph |

---

### 🧾 Sales Related Data Points

| **Metric** | **Definition** | **Backend Logic / Formula** | **Data Source** | **Visualization** |
| --- | --- | --- | --- | --- |
| Number of Dishes Sold Yesterday | Dish count containing the item | `COUNT(dishes)` where `date = yesterday` and ingredient in dish | Sales Sheet | Static Number |
| Total Quantity Sold Yesterday | Total item usage from dishes | `SUM(dish_qty_sold × item_qty_per_dish)` for `date = yesterday` | Sales Sheet, Mapping Table | Static Number or Bar Chart |
| Dish-level Consumption (Top 5) | Top dishes that used this item | Group dishes by item usage, order DESC, limit 5 | Sales Sheet, Mapping Table | Bar Chart or Pie Chart |

---

### 📈 Forecasting Related Data Points

| **Metric** | **Definition** | **Backend Logic / Formula** | **Data Source** | **Visualization** |
| --- | --- | --- | --- | --- |
| Purchase Forecast | Next buy date and quantity needed | If `Stock < Threshold`, then `Forecast Qty = Avg Daily Use × X Days` | All sources (stock, sales) | Static Text + Line Marker |
| Supply Forecast | Days until current stock runs out | `Current Stock / Avg Daily Usage` | Stock Calc, Sales Sheet | Static Number or Line Chart |
| Preparation Forecast | Expected prep quantity based on projected sales | `Forecast Sales × Avg Ingredient Usage per Dish` | Sales + Prep Sheet | Static Number or Forecast Line |

---

### 📊 Visualization Related Data Points

| **Metric** | **Definition** | **Backend Logic / Formula** | **Data Source** | **Visualization** |
| --- | --- | --- | --- | --- |
| Prep Frequency | How often the item was prepped | `COUNT(prep events)` grouped by date | Prep Sheet | Line Chart or Heatmap |
| Reorder Point Indicators | Points where stock nears reorder threshold | Overlay line on stock level trend when `Stock < Threshold` | Stock Calculation | Line Chart with Threshold Line |
| Purchase Frequency | How often item is purchased | `COUNT(purchase events)` grouped by date | Purchase Sheet | Bar Chart or Calendar Plot |

## 🧩 OVERALL UX GOALS

| **User** | **Goals** |
| --- | --- |
| **Chef** | See what to prep, what's low on stock, yesterday’s performance, spoilage |
| **Owner** | Understand inventory health, cost efficiency, wastage, sales, forecasts |

---

## 🧱 STRUCTURE OF THE DASHBOARD

### 1. **Home Screen (Ingredient Overview)**

| **Element** | **Mobile View** | **Web View** |
| --- | --- | --- |
| Search Bar | Full-width at top | Top-left, sticky |
| Filter Tabs | Scrollable tabs: All / Low Stock / High Waste | Top bar tabs with icons |
| Ingredient Cards | 2 per row (name, unit, key metric tags) | Grid layout, 4 per row with full metrics |
| Alerts (e.g. Reorder) | Inline chip with red/yellow icon | Alert column or badges |

🔹 **Tap on card → Ingredient Detail Page**

---

### 2. **Ingredient Detail Page**

### Top Header:

- Ingredient Name
- Unit (kg, g, etc.)
- Reorder status tag

### Section-wise Scroll View:

| **Section** | **Components** | **Mobile UX** | **Web UX** |
| --- | --- | --- | --- |
| **Stock Section** | - Current Stock (static)- Last Buy (date)- Stock Period- Reorder Indicator (graph) | Static cards + Line chart | Grid cards + Line chart with thresholds |
| **Prep Section** | - Prepared Stock- Average Yield (gauge)- Spoiled Waste- Prep Frequency (line chart) | Cards + mini chart toggle | Cards + Charts side by side |
| **Sales Section** | - Dishes Sold Yesterday- Quantity Sold- Top 5 Dish Consumption (bar/pie) | Scrollable cards + Pie toggle | Grid cards + Table + Pie chart |
| **Forecast Section** | - Purchase Forecast (static)- Supply Forecast (days)- Prep Forecast | Static + optional chart | Cards + Forecast trend line |

🔹 Bottom Button (Mobile): “View Purchase & Prep History”

---

### 3. **Comparison & Analytics Page**

| **Feature** | **Mobile View** | **Web View** |
| --- | --- | --- |
| Top 5 Wasted Ingredients | Horizontal bar / pie | Side-by-side bar chart |
| Low Stock Items | Scrollable list | Table or heat map |
| Most Prepped Ingredients | Line or bar chart | Comparison matrix |
| Purchase Trends | Compact line graph | Full-width trendline per ingredient |

🔹 Filters:

- Date Range
- Category (Veg / Dairy / Spices)

---

### 4. **Component Library Recommendation & Analytics Page**

| Component | Library (Web) | Library (Mobile / React Native) |
| --- | --- | --- |
| Cards, Tabs, Modals | ShadCN + Tailwind / MUI | React Native Paper / NativeBase |
| Charts | Recharts / Chart.js | Victory Native / React Native Charts |
| Navigation | React Router + Sidebar | React Navigation |
| Responsiveness | Tailwind responsive classes | Flex layouts + SafeAreaView |
