# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Smedicalis boutique: a static, French-language catalogue site for a medical-equipment seller in Bamako, Mali, served at `smedicalis.com` (see `CNAME`). Customers browse prices and order through WhatsApp; there is no backend, no payment, no database.

The shop is **`index.html`** (HTML + two `<style>` blocks + one inline `<script>`), with the catalogue data in **`produits.js`** and an owner-only catalogue editor in **`admin.html`**. There is no build step, no package manager, no test suite and no linter. To preview locally, serve the folder (e.g. `python3 -m http.server`) and open it in a browser; check mobile width, since most customers are on phones. `logo.png` is the favicon/OG image; the header logo is a base64 data URI inlined in `index.html`.

User-facing text, comments and identifiers are in French — keep it that way.

## Architecture of the inline script

Everything is global functions and constants, rendered by string-building HTML into containers (`#grid`, `#cats`, `#cart-items`…) and wired with inline `onclick` attributes. Any text that comes from user input or OCR must go through `esc()` before being inserted.

Key data (edit these to change the catalogue, not the rendering code):
- `WA_NUMBER` — Smedicalis WhatsApp number (international format without `+`). Cart orders go to this number; the header link hardcodes the same number separately.
- `PRODUITS` — the catalogue: `{id, nom, prix (FCFA), cat, desc, photo}`. It is read from `window.CATALOGUE`, defined in `produits.js` (loaded before the inline script); `index.html` no longer embeds the list. `produits.js` must stay `window.CATALOGUE = ` followed by strict JSON, one product per line, because `admin.html` parses and rewrites it. `id` is a stable number referenced by kits and saved carts — never renumber or reuse ids. Categories (`CATS`) are derived from the `cat` values.
- `KITS` — bundles for students (`public:'etudiant'`) and professionals, each a list of `[productId, qty]`. Kit prices are computed from catalogue prices minus `REMISE_ETUDIANT` / `REMISE_PRO`, so they follow price changes automatically. Kit ids are strings starting with `kit-`; `article(cle)` resolves either a product id or a kit id, and cart keys use the same scheme.
- Photos — a product's `photo` field (path under `images/`); the `PHOTOS` map in `index.html` is only for kit photos. Wikimedia photos must stay credited in `credits.html`. Without a photo, `visuel()` draws an SVG illustration chosen by the first matching regex in `ICONES` (on the accent-stripped name), falling back to `CAT_ICONE`, coloured by `CAT_TEINTE`. New categories need entries in `CAT_ICONE`/`CAT_TEINTE`/`CAT_CHIP_ICONS`.

Features:
- **Catalogue & cart**: filtering by category/audience, search (on precomputed `p._q`, normalised by `dvNorm`), sorting; cart checkout (`envoyerWhatsApp`) validates customer name/phone/location and opens WhatsApp (`api.whatsapp.com/send?phone=WA_NUMBER`) with the order text. Quote and share buttons instead open `wa.me/?text=` with no recipient so the customer picks one.
- **Devis rapide** (`dv*` functions, `#devis-overlay`): paste a shopping list or photograph one (OCR via Tesseract.js lazy-loaded from jsDelivr), fuzzy-match lines to products (`dvTrouver`/`dvScore`, stop-words in `DV_STOP`), then share by WhatsApp or print a quote as PDF (`dvPDF` opens a printable window).

State is persisted in `localStorage` only through `lsGet`/`lsSet` (which tolerate private mode): `smed_cart`, `smed_devis`, `smed_cli` (customer details), `smed_bienvenue` (welcome overlay dismissed). Saved carts drop ids no longer in the catalogue, so removing a product is safe.

## Catalogue editor (`admin.html`)

Standalone page for the shop owner: search, add, edit, delete products and upload a photo (resized to 900 px in the browser). It commits straight to `main` through the GitHub Contents API (`produits.js`, plus new files under `images/`) using a fine-grained personal access token that the owner pastes in; the token lives only in that browser's `localStorage` (`smed_admin_cle`). So `main` can change outside pull requests — fetch before working, and never reformat `produits.js`.

## Repository

`.claude/settings.json` enables Claude Code plugins (supabase, vercel, oh-my-claudecode, chrome-devtools-mcp) from their marketplaces.
