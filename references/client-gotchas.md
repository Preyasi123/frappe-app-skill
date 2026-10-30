# Gotchas hit on past client projects

## Setting up a site
- `bench install-app` marks the app's patches as done **without running them**, so custom-field patches (`create_custom_fields`) must be run manually: `bench --site <site> execute <app>.patches.<patch>.execute`.
- An empty site can't create anything without a Company, so finish the setup wizard first.
- `Contact.is_billing_contact` missing breaks every Customer save; fix it with `erpnext.setup.install.create_address_and_contact_custom_fields`.
- If `default_site` is unset, plain `localhost:<port>` returns 404, so run `bench use <site>`.
- A doctype that was added but never migrated needs `bench migrate`.

## Frappe behaviour
- If a doctype has no autoname, passing `"name"` in the dict does nothing; you have to set `doc.name = x; doc.flags.name_set = True`.
- Time fields read back as `00:00:00.000000`, so "not checked out" means NULL **or** midnight. Use `time_to_sec(col) > 0` and never a string compare.
- Coordinates stored as `Data` (e.g. `custom_latitude`) must be cast before any numeric compare.
- Sales Person is a tree, so exclude `is_group=1` (the root "Sales Team").
- ERPNext **Quotation has no `sales_team`** (Sales Order and Customer do). Attribute a quotation through the customer's `Customer.sales_team`.
- Under REPEATABLE READ, a plain SELECT after locking can see a stale snapshot. Lock the parent `FOR UPDATE` first, then read with `LOCK IN SHARE MODE`. Lock in a consistent order to avoid deadlocks.
- Some client apps commit inside hooks (waves2, petra block logic), so rollback won't undo them and you must clean up by hand.
- A workspace with the same name as a doctype hides the doctype's list route.

## Waves-family field-sales models (they differ per client, so re-check each time)
| | waves (vajram) | asians (asian_food_waves) | malayil | petra |
|---|---|---|---|---|
| Visit target | Customer | **Shop** (Customer = distributor) | Customer (doctype is "Shop Checkin") | no field sales |
| Staff key | Employee `custom_type` | Sales Person (bridge `Sales Person.employee`) | Sales Person | Sales Person via Customer.sales_team |
| Grouping | Territory | Route | Route | district |
| Route Map child | — | `Route Map Childs` | `Route Map Child` | — |
| checkout_time | real (midnight = open) | UI custom field only; fall back to `total_time > 0` | real | — |
| Orders | Sales Order | Sales Order | Sales Order `custom_sales_person` | Quotation → SO, expiry blocks dealer |

- `Daily Location.daily_distance` is a **running cumulative total**, so take MAX per (employee, date), never SUM. Per-ping deltas (`custom_distance_from_last`) can be summed.
- Petra expiry is `Quotation.valid_till` but `Sales Order.custom_reservation_expiry_date`. Only quotations block dealers.
- Petra: a quotation can't be born expired. Insert it live, then back-date with `db.set_value`, then run Petra's own blocker.

## Landing page instead of Desk
- `/` uses the `home_page` hook, but a Role's `home_page` overrides it.
- After login, a System User goes to System Settings `default_app` → that app's route in `add_to_apps_screen`.
- `after_install` won't fire if the app is already installed, so call the setter manually.
