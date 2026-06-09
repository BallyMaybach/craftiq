# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

- **Projekt:** CraftIQ
- **Was es tut:** Handwerker-App — Kostenvoranschläge per KI generieren, Kunden verwalten, Rechnungen tracken, Termine planen
- **Stand:** MVP fertig — alle 5 Tabs funktionieren, KI über Supabase Edge Function
- **Jetzt:** Schulprojekt W&R, Start-Up Pitch-Demo
- **Deployment:** GitHub Pages → https://ballymaybach.github.io/craftiq/
- **Design-Richtung:** Dark, Navy-Blau — Cormorant Garamond + Figtree, Hellblau-Akzent `#219CE8`

## Entwicklung

Kein Build-Step, kein Package-Manager. `index.html` direkt im Browser öffnen oder via Live Server.

PWA-Cache invalidieren bei Asset-Änderungen: Cache-Name `craftiq-v1` in `sw.js` erhöhen.

Deployen: `git add . && git commit -m "..." && git push` → GitHub Pages baut automatisch.

## Stack

- Single-file HTML/CSS/JS PWA (`index.html`) — alles in einer Datei
- Daten: ausschließlich localStorage (`craftiq_*` Keys)
- KI: Supabase Edge Function `craftiq-ai` als Proxy → Anthropic `claude-haiku-4-5-20251001`
- PDF: `window.open(blob:url)` + `window.print()` — Print-CSS inline via `getPDFStyles()`

## KI-Integration

`AI_ENDPOINT` in `index.html` zeigt auf `https://yyxnhwkdwcgpqajeknfe.supabase.co/functions/v1/craftiq-ai`.

Die Edge Function liegt im Supabase-Projekt `Bally-OS-Daten` (id: `yyxnhwkdwcgpqajeknfe`). Der Anthropic API Key ist dort als Secret `ANTHROPIC_API_KEY` hinterlegt — **nicht** im Code.

`generateWithAI()` sendet `{ description }` an den Endpoint, der Anthropic-Response kommt 1:1 zurück. Bei Fehler: Toast + manuelle Positionseingabe bleibt verfügbar.

## JS-Architektur

Alles in einem einzigen `<script>`-Block am Ende von `index.html`. Kein Modul-System.

**Globaler State** (`appState`):
```js
{ tab, editingCustomerId, editingQuoteId, editingAppointmentId, quoteFilter, invoiceFilter }
```
`quoteItems[]` ist eine separate globale Variable (aktive Positionen im Angebots-Editor).

**Render-Pattern:** Jeder Tab hat `render<Tab>()`, die `innerHTML` von `#tab-<name>` komplett neu setzt. `renderTab(tab)` dispatcht. Kein Diff.

**Datenfluss:**
1. Lesen: `store.get(key, default)` → `localStorage.getItem('craftiq_' + key)` + `JSON.parse`
2. Schreiben: `store.set(key, val)` → danach `renderTab(appState.tab)` aufrufen
3. KI-Call: `generateWithAI()` → Edge Function → parst JSON aus Response → befüllt `quoteItems`

**Modal-System:** `openModal(id)` / `closeModal(id)` toggled `.open` auf `.modal-overlay`. Jedes Modal hat `open<Entity>Modal(id?)` die State setzt + Modal öffnet.

**PDF-Generierung:** `printQuotePDF(quoteId)` / `printInvoicePDF(invoiceId)` bauen vollständiges HTML mit `getPDFStyles()`, öffnen Blob-URL in neuem Tab, rufen `window.print()` auf.

## Datenstruktur (localStorage)

```
craftiq_customers      [{id, name, email, phone, address, notes, created_at}]
craftiq_quotes         [{id, customer_id, title, description, items:[{beschreibung,einheit,menge,einzelpreis}], total, status, created_at}]
craftiq_invoices       [{id, quote_id, customer_id, number, amount, status, due_date, paid_date, created_at}]
craftiq_appointments   [{id, customer_id, title, date, time, notes, created_at}]
craftiq_settings       {firma, inhaber, strasse, plz, ort, telefon, email, iban, steuernummer}
craftiq_invoiceCounter {year, count}
```

Quote-Status: `entwurf` → `verschickt` → `angenommen` / `abgelehnt`
Invoice-Status: `offen` → `bezahlt`

## Design-Tokens

```css
--bg: #0f1318
--bg-elevated: #161b22
--surface: #1c2430
--surface-2: #1e2a38
--accent: #219CE8
--gold: #ED9434
--text: #ffffff
--muted: #7a9bb8
```

## PWA

`manifest.json` + `sw.js`, Cache-Name `craftiq-v1`. Bei Änderungen an `index.html` oder `icon.png` Cache-Name erhöhen. `safe-area-inset-*` im CSS für iPhone Notch.
