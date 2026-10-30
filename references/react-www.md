# React-in-Frappe templates (from peace_valley)

Replace `myapp`, `mypage`, `ma` (prefix) and the site/port.

## frontend/vite.config.js
```js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

// Frappe dev server (bench start). Override with FRAPPE_URL=http://host:port yarn dev
const FRAPPE_URL = process.env.FRAPPE_URL || "http://myclient.localhost:8019";

export default defineConfig({
	plugins: [react()],
	base: "/assets/myapp/frontend/",
	build: {
		outDir: "../myapp/public/frontend",
		emptyOutDir: true,
		manifest: true,
		chunkSizeWarningLimit: 2000,
		rollupOptions: { input: "src/main.jsx" },
	},
	server: {
		port: 8082,
		proxy: { "^/(api|app|assets|files|login|private)": { target: FRAPPE_URL, changeOrigin: false } },
	},
});
```
Deps: react, react-dom, react-router-dom, antd 5 (+ `@ant-design/v5-patch-for-react-19` on React 19), `@ant-design/icons`, dayjs.

## myapp/www/mypage.py
```python
import json
import os

import frappe
from frappe import _

no_cache = 1

MANIFEST = os.path.join(frappe.get_app_path("myapp"), "public", "frontend", ".vite", "manifest.json")
ASSET_BASE = "/assets/myapp/frontend/"


def get_context(context):
	if frappe.session.user == "Guest":
		frappe.local.flags.redirect_location = "/login?redirect-to=" + frappe.request.path
		raise frappe.Redirect
	context.no_breadcrumbs = 1
	context.title = "My Page"
	context.boot = {"csrf_token": frappe.sessions.get_csrf_token()}
	if not os.path.exists(MANIFEST):
		frappe.throw(_("Portal is not built. Run `bench build --app myapp`."))
	with open(MANIFEST) as f:
		entry = json.load(f)["src/main.jsx"]
	context.script = ASSET_BASE + entry["file"]
	context.styles = [ASSET_BASE + css for css in entry.get("css", [])]
	return context
```

## myapp/www/mypage.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="utf-8">
	<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
	<title>{{ title }}</title>
	{% for href in styles %}<link rel="stylesheet" href="{{ href }}">
	{% endfor %}
</head>
<body>
	<div id="root"></div>
	<script>window.ma_boot = {{ boot | tojson }};</script>
	<script type="module" src="{{ script }}"></script>
</body>
</html>
```

## hooks.py
```python
add_to_apps_screen = [{"name": "myapp", "logo": "/assets/myapp/images/logo.png", "title": "My App", "route": "/mypage"}]
after_install = "myapp.install.after_install"
after_migrate = "myapp.install.after_migrate"   # roles, settings defaults, workspace blocks
website_route_rules = [{"from_route": "/mypage/<path:app_path>", "to_route": "mypage"}]
```
React side: `<BrowserRouter basename="/mypage">` (or a hash router for WebView dashboards).

## frontend/src/lib/api.js
```js
const csrf = () => (window.ma_boot || {}).csrf_token;

function errorText(data, fallback) {
	if (data?._server_messages) {
		try {
			return JSON.parse(data._server_messages).map((m) => JSON.parse(m).message).join("\n").replace(/<[^>]+>/g, "");
		} catch { /* fall through */ }
	}
	if (data?.exception) return data.exception.split(":").slice(1).join(":").trim() || data.exception;
	return data?.exc_type || fallback;
}

export async function call(method, args = {}, { write = false } = {}) {
	const url = `/api/method/myapp.api.${method}`;
	const res = write
		? await fetch(url, {
				method: "POST",
				headers: { "Content-Type": "application/json", Accept: "application/json", "X-Frappe-CSRF-Token": csrf() },
				body: JSON.stringify(args),
			})
		: await fetch(`${url}?${new URLSearchParams(Object.entries(args)
				.filter(([, v]) => v !== undefined && v !== null && v !== "")
				.map(([k, v]) => [k, typeof v === "object" ? JSON.stringify(v) : v]))}`, { headers: { Accept: "application/json" } });
	if (res.status === 403 && res.headers.get("content-type")?.includes("html")) {
		window.location.href = "/login?redirect-to=" + encodeURIComponent(window.location.pathname);
	}
	const data = await res.json().catch(() => ({}));
	if (!res.ok) throw new Error(errorText(data, res.statusText));
	return data.message;
}

export const post = (method, args) => call(method, args, { write: true });
```
File uploads: POST FormData to `/api/method/upload_file` with `is_private=1` for payment/identity attachments.

## Demo registry (demo.py)
```python
REGISTRY = "ma_demo.json"

def _reg_path(): return frappe.get_site_path("private", REGISTRY)
def _load():
	p = _reg_path()
	return json.load(open(p)) if os.path.exists(p) else {}
def _save(reg):
	with open(_reg_path(), "w") as f: json.dump(reg, f, indent=1)

def make():
	if _load():
		print("Demo data already loaded; run delete first"); return
	reg = {dt: [] for dt in (...)}
	# create, append each name to reg[doctype], _save(reg) as you go
	frappe.db.commit()

def delete():
	reg = _load()
	# frappe.db.delete children → parents, plus Comment/Version rows; delete_doc for Users
	frappe.db.commit(); os.remove(_reg_path())
```

## Test runner (tests/test_flow.py)
```python
RESULTS = []
def check(code, label, cond, detail=""): RESULTS.append((code, label, bool(cond), detail))

def run():
	with patch.object(frappe.db, "commit", lambda *a, **k: None), patch.object(frappe.db, "rollback", lambda *a, **k: None), \
			patch("frappe.enqueue", lambda *a, **k: None):
		try:
			frappe.set_user("Administrator"); _run()
		finally:
			frappe.set_user("Administrator"); frappe.db.sql("rollback")
	failed = [r for r in RESULTS if not r[2]]
	for code, label, ok, detail in RESULTS:
		print(f"{'PASS' if ok else 'FAIL'}  {code:6} {label}{'  -> ' + str(detail) if detail and not ok else ''}")
	print(f"\n{len(RESULTS) - len(failed)}/{len(RESULTS)} passed")
	return len(failed)
```
