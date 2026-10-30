---
name: spiralcode-frappe-app
description: The user's (Spiralcode) house way of building client projects on Frappe/ERPNext - a custom Frappe app whose doctypes live in Frappe and whose UI is a React (Vite + antd) single-page app served from a www page, built from a client BRD/brief, with no hardcoded data, removable demo data, bench-execute test suites, and per-project dashboards cloned from a previous one. Use whenever the user starts or extends a client prototype/app/dashboard on a bench (Peace Valley, Malayil, Petra, Waves, Asians, DemoApp, DPlus style), hands over a BRD/docx/brief, asks for "doctype data flow", "React UI", "dashboard like the last one", "demo data", "test cases", or screenshots of the UI.
---

# Spiralcode Frappe + React client app

How the user builds client projects. Follow these defaults unless the user says otherwise; pair with the `frappe-*` skills for framework detail.

## 0. Before touching code
- **Find the coordinates**: bench dir (`~/<Client>-bench`), site (`<client>.localhost`), web port (80xx) and redis ports (110xx/130xx), the app name, the module name. Check memory first; each project has a memory file.
- **The user runs `bench start` in their own VS Code terminal.** Never start a second one (it kills theirs, exit 241). If nothing is running and it's needed, ask, or start only redis + `bench serve --port <port>` on a bench where nothing runs.
- **Treat restored sites as real data** (e.g. Malayil was restored from live). No destructive SQL or seed data on them without asking.
- New bench site that's empty: run the ERPNext setup wizard via API (Company, abbr, INR/India, FY Apr-Mar), `bench use <site>` if `default_site` is unset, then say what Company/abbr/admin password you chose.

## 1. Scope from the BRD
- The client gives a BRD/brief (usually a .docx in Windows `Downloads`). **Build what the boss asked plus the BRD's names (statuses, actions, labels)** — not the whole BRD. List what you're deliberately leaving out (payments, approvals, OTP, notifications…) and confirm.
- Use the client's real name/branding (BRDs misspell: "Piece Valley" → Peace Valley). Take theme colours from the client's logo/site.
- Keep one app per concern: a booking site does not go inside the ERP app (`malayil` vs `malayil_booking`), a dashboard is its own app (`<project>_dashboard`). Ask the user for the app name.

## 2. Doctypes
- Whenever you describe or propose doctypes ("data flow", "what fields"), give a **full field table**: fieldname (snake_case) | fieldtype (exact Frappe type) | label | options (Link target / Select values) | reqd / unique / permlevel. The user creates or reviews them in Desk from this table.
- Doctypes live in the Frappe app (JSON under `<module>/doctype/`), may be generated from a script, logic in module `.py` files, not in the React app.
- Frappe can't nest a child table in a child row — use two tables linked by a key column.
- Never name a Workspace the same as a DocType (the workspace route hides the list).
- Multi-company clients: put Company (+ Cost Center when asked) on every transaction, and filter Cost Center by Company with `doctype_js`.
- Sensitive fields (PAN, Aadhaar…): permlevel 1 and masked in the API for low roles. Create roles in `after_migrate`.
- Use statuses from the BRD verbatim; submittable doctypes for anything that's "confirmed"/"receipted".

## 3. Backend API (`<app>/api.py`)
- `@frappe.whitelist()` methods; the React client calls them as `/api/method/<app>.api.<fn>`.
- **Avoid `frappe.get_doc`**: read with `frappe.get_all` / `frappe.db.get_value` / `frappe.qb` / `frappe.db.sql` (parameterised), write with `frappe.db.set_value`. Use get_doc only to insert, submit or cancel (controller hooks must run), once, and say why.
- Check permissions explicitly in whitelisted methods (`frappe.has_permission`, role checks) since get_all skips them.
- Anything that can be double-booked/double-run: idempotency keys or unique keys + row locks (`FOR UPDATE` on the parent resource first, then `LOCK IN SHARE MODE` reads), and a concurrency test with real commits.
- Guard optional custom fields from other apps with a `_has_field` check — custom-field patches are often never run.

## 4. React UI
Templates in `references/react-www.md`. Shape:
- `apps/<app>/frontend/` (Vite, React, antd 5, react-router) builds to `<app>/public/frontend` with `manifest: true` and `base: "/assets/<app>/frontend/"`.
- `www/<page>.py` reads the Vite manifest and injects script/css + a boot object (`window.<prefix>_boot` with csrf_token, user, brand); `www/<page>.html` is the shell. Controller filenames need underscores (`stays.py`, `waves_dashboard.py`).
- `website_route_rules` `/<page>/<path:app_path>` → `<page>` for client routes; `add_to_apps_screen` entry; login-required redirect in `get_context`.
- Short unique prefix per app for CSS classes and boot global (`pv-`/`pv_boot`, `md-`/`md_boot`).
- Thin `lib/api.js`: GET for reads, POST + `X-Frappe-CSRF-Token` for writes, parse `_server_messages` for errors, redirect to `/login` on 403.
- Build: `npx vite build` inside `frontend/` (or `bench build --app <app>`).
- Mobile WebView use: support `?embed=1` to hide chrome.
- Making the app the site landing page: `home_page` hook **and** System Settings `default_app` (set in `after_install`, route must be in `add_to_apps_screen`), with an `unset_as_default_app` to hand Desk back.

## 5. Data rules (the user has repeated these)
- **No hardcoded data** — branding, lists, numbers come from DB/settings (`<App> Settings` single or `hooks.py` brand dict / site_config). Data is entered through Desk forms.
- **No demo data unless asked.** When asked, ship `<module>/demo.py` with `make()` / `delete()` that records every created name in a registry `sites/<site>/private/<prefix>_demo.json`, refuses to `make` twice, and `delete` removes exactly what it made (plus Comments/Versions). Run via `bench --site <site> execute <app>.<module>.demo.make`. For dashboard apps that must stay data-free, demo scripts go outside `apps/` (`<bench>/demo-data/seed_x.py` / `cleanup_x.py`).
- Demo logins use a fake domain (`*@<client>.demo`) and one shared password; tell the user.

## 6. Tests
- Single runnable suite: `bench --site <site> execute <app>.<module>.tests.test_flow.run` that prints `PASS/FAIL  <code> <label>` and `N/M passed`.
- Map checks to BRD acceptance-criteria codes (AC01…) plus API smoke tests.
- Patch `frappe.db.commit`/`rollback` and `frappe.enqueue` (redis may be down), roll back in `finally`. Concurrency tests are the exception: real commits, self-cleaning.
- Report the real pass count; never claim green you didn't see.

## 7. Verify the UI
- Screenshots with headless Windows Chrome from WSL: `"/mnt/c/Program Files/Google/Chrome/Application/chrome.exe" --headless=new --window-size=1400,900 --screenshot=<path> <url>`; for logged-in pages use a temporary same-origin login page in `public/` and **delete it afterwards**. One run at a time; min viewport ~500px.
- No logo supplied → remove favicon/icon links rather than leave 404s, and say so.

## 8. Cloning a per-project dashboard
The user wants one dashboard app per project using the same shell (waves → asians → petra → malayil). See `references/client-gotchas.md`.
- Clone the closest previous app, rename prefix/boot global/API base/`HOST_APP` module.
- **Re-read the host app's doctypes before writing any query** — every client's model differs (Shop vs Customer, Sales Person vs Employee, singular vs plural child doctype, Data vs Float coordinates, which field holds expiry).
- Grep the frontend for the previous project's domain nouns; components read API keys by name and fail silently as `undefined`.
- Drop features the client has no data for (Petra: no locations → no map, drop leaflet).

## 9. Finishing
- Save/update the project memory file: coordinates, run commands (tests, demo make/delete, build), credentials you set, decisions, gotchas.
- Bugs found in the client's other apps: report them, don't fix them unasked.
- End with what was built, what's untested (e.g. no data for the map half), and what's pending from the BRD.
