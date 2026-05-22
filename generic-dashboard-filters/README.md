# Case Study: Generic Filterable Dashboard

## The Challenge

Design a dashboard that can filter through high-frequency data streams (like an internal admin panel tracking incoming orders or customer updates).

- **Left side:** A dynamic filter panel (search boxes, date pickers, dropdowns) built dynamically from a config object.
- **Right side:** An infinite-scrolling feed loading 20 items at a time.
- **The Catch:** The data source updates fast—around 10 new database records every single second. We need to handle deep scrolling without the UI lagging, and crucially, without duplicate entries showing up as new items hit the top of the database.

---

## 1. Scale Calculations & Performance Bottlenecks

Let's do some quick math to see how much data is hitting the browser during a normal user session and figure out where the app will actually break.

- Ingestion rate: 10 new rows/sec
- 1 minute = 600 new rows
- 1 hour = 36,000 new rows

If a single JSON object for an item averages around 1.5 KB, fetching a standard batch of 20 items is roughly 30 KB of data over the network.

Say an ops manager leaves the tab open, goes deep, and scrolls through 100 pages of history. The client-side cache will hold:
100 pages _ 20 items = 2,000 objects
2,000 objects _ 1.5 KB = ~3 MB of data in memory

**The Takeaway:**
Keeping 3 MB of raw data in a JavaScript variable is nothing; memory size isn't going to crash the tab. The actual bottleneck is the browser DOM. If you let 2,000 physical HTML table rows stay mounted at the same time, the browser will lag during scrolling. Another major issue is layout thrashing—if typing in a search box causes the right pane's long list to re-evaluate and re-render on every keystroke, the UI will freeze.

---

## 2. Core Requirements

### Functional

- **Isolated Filter Staging:** Typing a query or picking a date shouldn't instantly fire an API call. The user needs to stage their filters locally, and the network request should only execute when they click an explicit "Apply" or "Search" button.
- **Config-Driven Forms:** The filter panel layout must be completely dynamic. It should accept a schema array so the same dashboard component can be reused for `/orders`, `/customers`, or `/inventory` without rewriting form components.
- **Infinite Scroll Pagination:** Fetch and append next batches of 20 rows smoothly as the user nears the bottom of the page.

### Non-Functional

- **Consistent Data Consistency:** No duplicate rows or missing gaps when scrolling down, even if dozens of new items were inserted at the top of the database while the user was reading the first page.
- **Render Boundaries:** Interacting with the left panel inputs must have zero render-cycle impact on the right panel's active row list.
- **Frame Rate:** The list scroll behavior needs to stay close to a smooth 60 FPS, even when the underlying data array scales up.

---

## 3. The Implementation Blueprint

I'm breaking this problem down into a few clear phases for the playbook:

- **Part 1 (This file):** Traffic math, scoping boundaries, and pinpointing browser bottlenecks.
- **Part 2:** Architecture setup—separating the draft state (Jotai) from the server synchronized list view (React Query).
- **Part 3:** The pagination fix—why standard offset/limit queries break on live data feeds and how cursor tokens prevent item duplication.
- **Part 4:** API JSON request schemas and building clean error boundary handling for network dropouts.
- **Part 5:** Integrating DOM virtualization to cap the maximum number of active nodes.
