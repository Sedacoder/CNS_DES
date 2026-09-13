# Stratizen Food Finder — Project Guide

**Database and Enterprise Systems — Group Project**
Team: 6 members | Timeline: ~3 months | Stack: Python + Flask, MySQL, HTML/CSS/JS

---

## 1. What We're Building (Recap)

A web app where Strathmore students ("Stratizens") can search for food by item (e.g. "shawarma") or by location (on-campus vs off-campus), browse restaurant menus, place an order for pickup, and see live-updated stock. Vendors get their own dashboard to manage their menu, quantities, and incoming orders.

---

## 2. Full Feature Breakdown

### Student-facing features

| Feature | What it does | Notes |
|---|---|---|
| Sign up / Log in | Student creates an account, logs in with email + password | Consider restricting to Strathmore email domain |
| Search by food item | Type "shawarma" → see every vendor offering it | Uses `LIKE` matching on menu item names |
| Search/filter by location | Filter to on-campus or off-campus vendors, maybe by specific area | Needs a `campus_status` or location field on vendors |
| Browse restaurant page | See a vendor's full menu, prices, live stock | One page per vendor |
| Cart | Add one or more items before placing an order | Can be session-based, doesn't need its own DB table if kept simple |
| Place order | Submits cart as an order to the vendor | Triggers the stock-decrement transaction |
| Order status tracking | See "preparing" → "ready for pickup" | Simple status field, updated by vendor |
| Bestsellers | See top-selling items per restaurant | Derived from aggregating past orders — build this last |
| Order history (optional stretch) | Student sees their past orders | Nice-to-have, not core |

### Vendor-facing features

| Feature | What it does | Notes |
|---|---|---|
| Vendor login | Separate role from student | Same `users` table, different `role` value |
| Menu management | Add, edit, remove menu items; set price and category | |
| Stock management | Update available quantity per item | Can be manual or auto-updated by orders |
| Incoming orders view | See new orders as they come in | Refresh or poll every few seconds is enough — no need for websockets |
| Mark order ready | Updates order status so student knows to collect | |

### System-level / behind-the-scenes features

- **Authentication & sessions** — who's logged in, what role they have, what they're allowed to do.
- **Role-based access control** — a student shouldn't be able to hit a vendor's "edit menu" endpoint, and vice versa. This needs to be checked server-side in Flask, not just hidden in the frontend.
- **Stock consistency** — when an order is placed, quantity must decrease correctly even if two students order at once (see challenges below).
- **Search** — the `LIKE`-based query covered earlier; can later add filtering by category, price range, "open now."

---

## 3. Database Design

### One database, not one per restaurant

Use a **single MySQL database** for the whole system, with a `vendor_id` foreign key tying menu items and orders to a specific restaurant. This is called **multi-tenancy via a shared schema** — it's the standard approach, and it's what makes cross-vendor search possible in the first place. If you had one database per restaurant, a single search for "shawarma" would have to query every single restaurant's separate database and merge the results — much slower and much harder to maintain, especially as you add more vendors. One shared database with foreign keys is simpler, faster to query, and easier for six people to work on together (one schema, one place to reason about).

### Suggested tables

```
users
  id (PK)
  name
  email
  password_hash
  role            -- 'student' or 'vendor'

vendors
  id (PK)
  user_id (FK -> users.id)      -- the account that manages this vendor
  name
  location_description
  campus_status                 -- 'on-campus' or 'off-campus'

categories                      -- optional, but nice for filtering
  id (PK)
  name                          -- e.g. 'Fast food', 'Swahili dishes', 'Drinks'

menu_items
  id (PK)
  vendor_id (FK -> vendors.id)
  category_id (FK -> categories.id, nullable)
  item_name
  price
  quantity_available

orders
  id (PK)
  student_id (FK -> users.id)
  vendor_id (FK -> vendors.id)
  status                        -- 'placed', 'preparing', 'ready', 'collected'
  created_at

order_items
  id (PK)
  order_id (FK -> orders.id)
  menu_item_id (FK -> menu_items.id)
  quantity_ordered
  price_at_order_time            -- store this so later price changes don't rewrite history
```

Bestsellers is just a query, not a table:
```sql
SELECT menu_items.item_name, SUM(order_items.quantity_ordered) AS total_sold
FROM order_items
JOIN menu_items ON order_items.menu_item_id = menu_items.id
WHERE menu_items.vendor_id = %s
GROUP BY menu_items.id
ORDER BY total_sold DESC
LIMIT 5;
```

---

## 4. What's Actually Going to Be Hard (Be Careful Here)

- **Stock race conditions.** If two students order the last 2 slices of pizza at the same moment, both requests must not succeed. The fix is wrapping "check stock → decrement stock → confirm order" in a single MySQL transaction (`START TRANSACTION` / `COMMIT`), ideally with a row lock (`SELECT ... FOR UPDATE`) so a second request has to wait for the first to finish before it reads the quantity. This is genuinely the most technically demanding part of the whole project — budget real time for it, and test it deliberately (e.g. two browser tabs ordering the same item at once).
- **SQL injection.** Always use parameterized queries (`%s` placeholders), never string-format user input directly into SQL. One careless line here is a real vulnerability, not just a style nitpick.
- **Role enforcement on the backend, not just the frontend.** Hiding a "manage menu" button from students in the UI does nothing if the underlying Flask route doesn't also check `if user.role != 'vendor': reject`. Anyone can call your API directly.
- **Session security.** Use Flask's built-in session handling and hash passwords (`werkzeug.security.generate_password_hash`) — never store plaintext passwords.
- **Search performance.** Fine to ignore early on, but once you have real data, add a MySQL index on `item_name` so `LIKE` searches stay fast.
- **Data consistency between cart and live stock.** If a student adds 2 slices to their cart and someone else buys the last one before they check out, you need to handle that gracefully at order time (re-check stock right before confirming, not just when the item was added to cart).
- **Scope creep.** Bestsellers, order history, ratings — these are all reasonable stretch features, but they should be built only after search, browse, and ordering work reliably end-to-end. Agree as a team on what's "core" vs "if we have time."
- **Six people, one schema.** Everyone touching the database means schema changes can break someone else's work. Agree on the schema early, version it (a `schema.sql` file in the repo), and treat changes to it as something the whole team is notified about — not something one person edits silently.

---

## 5. A Workflow That Fits a Team of 6

- **Branch per feature.** Each pair (see split below) works in their own Git branch, opens a pull request when a feature is ready, and at least one other teammate reviews it before merging into `main`. Avoids six people pushing straight to `main` and breaking each other's work.
- **Shared schema file.** Keep the database schema (table definitions) in a `schema.sql` file in the repo so everyone can rebuild the same database locally, and changes to it are visible in version control, not just in someone's head.
- **Short, regular check-ins.** A 15-minute sync twice a week (in person or on a call) is usually enough for a project this size — long enough to unblock people, short enough that it doesn't eat your build time.
- **Simple task board.** A GitHub Projects board or even a shared Trello with three columns (To do / In progress / Done) keeps everyone's tasks visible without needing heavyweight project management.
- **Feature-based pairing** (from your existing split):
  - **You + Leon** — backend & DB core: schema, auth, the order/stock transaction logic, API endpoints.
  - **Nicole + Emmanuel** — student-facing frontend: search, browse, cart, order status.
  - **David + Shem** — vendor-facing frontend: dashboard, menu/stock management, incoming orders.
- **Integration checkpoints.** Set a rough milestone (e.g. every 2-3 weeks) where all three pairs plug their pieces together and test the full flow end-to-end, rather than only integrating right before the deadline.

---

## 6. Learning Path & Resources

Everyone doesn't need to learn everything to the same depth — but here's a baseline path per technology.

### HTML/CSS/JS (frontend pairs especially)
- **MDN Web Docs** (developer.mozilla.org) — the standard reference for all three; good for looking things up as you go rather than reading cover to cover.
- **freeCodeCamp's Responsive Web Design + JavaScript courses** (freecodecamp.org) — free, structured, project-based.

### Python + Flask (backend pair especially, but everyone benefits from basics)
- **Official Flask Quickstart** (flask.palletsprojects.com) — short, and covers routes, templates, and request handling, which is most of what you'll use.
- **Corey Schafer's Flask tutorial series on YouTube** — widely recommended for beginners, walks through building a real app (forms, database, auth) step by step.

### MySQL / SQL (everyone, since it's the unit's focus)
- **W3Schools SQL/MySQL tutorial** (w3schools.com/sql) — good for syntax lookups and quick practice.
- **MySQL Tutorial** (mysqltutorial.org) — more thorough, covers joins, transactions, indexing specifically.
- **Practice**: recreate the schema above in a local MySQL instance and practice writing the search and bestsellers queries yourself before wiring them into Flask — it's much easier to debug SQL on its own than inside a web request.

### Suggested internal split for learning
Since you and Leon are doing the transaction/concurrency logic, it's worth the two of you specifically going a bit deeper into **MySQL transactions and locking** beyond what the rest of the team needs, since that's the trickiest technical piece in the whole project.

---

## 7. Rough Timeline (3 Months)

- **Weeks 1-2**: Finalize schema, set up repo/branches, everyone gets a local dev environment running (Flask + MySQL talking to each other).
- **Weeks 3-5**: Core features in parallel — search/browse (student pair), menu management (vendor pair), auth + basic order placement (backend pair).
- **Weeks 6-7**: Integrate — order placement wired to real stock decrement with transaction safety, test race conditions deliberately.
- **Weeks 8-9**: Order status flow, polish UI, start on bestsellers if on schedule.
- **Weeks 10-11**: Testing, bug fixing, edge cases (what happens at 0 stock, what happens if a vendor edits a menu item mid-order).
- **Week 12**: Documentation, report, presentation prep, buffer for anything slipped.
