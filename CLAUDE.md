# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

- **Projekt:** CraftIQ
- **Was es tut:** Handwerker-App — Kostenvoranschläge per KI generieren, Kunden verwalten, Rechnungen tracken, Termine planen
- **Stand:** MVP fertig — alle 5 Tabs funktionieren, KI-Integration via Anthropic Claude
- **Jetzt:** Schulprojekt W&R, Start-Up Pitch-Demo
- **Deployment:** lokal / GitHub Pages
- **Design-Richtung:** Dark, Navy-Blau — Cormorant Garamond + Figtree, Hellblau-Akzent `#219CE8`

## Entwicklung

Kein Build-Step, kein Package-Manager. Einfach `index.html` im Browser öffnen oder via Live Server. PWA-Cache bei Asset-Änderungen invalidieren: Cache-Name `craftiq-v1` in `sw.js` erhöhen.

## Stack

- Single-file HTML/CSS/JS PWA (`index.html`) — alles in einer Datei
- Kein Framework, kein Build-Step
- Daten: ausschließlich localStorage (`craftiq_*` Keys)
- KI: Anthropic API direkt → `claude-haiku-4-5-20251001`
- API Key: in `.env` als `API_KEY` — hardcoded im JS als `const API_KEY` (Demo-Modus, kein Backend)
- PDF: `window.open(blob:url)` + `window.print()` — Print-CSS inline via `getPDFStyles()`

## JS-Architektur

Alles in einem einzigen `<script>`-Block am Ende von `index.html`. Kein Modul-System.

**Globaler State** (`appState`):
```js
{ currentTab, editingCustomerId, editingQuoteId, editingAptId, quoteItems[], quoteFilter, invoiceFilter }
```

**Render-Pattern:** Jeder Tab hat eine `render<Tab>()` Funktion, die `innerHTML` eines `#tab-<name>` Elements komplett neu setzt. Kein Diff/VDOM. `renderTab(tab)` dispatcht auf die richtige Render-Funktion.

**Datenfluss:**
1. Lesen: direkt aus `localStorage.getItem('craftiq_*')` + `JSON.parse`
2. Schreiben: `JSON.stringify` + `localStorage.setItem` → danach `renderTab(currentTab)` neu aufrufen
3. KI-Call: `generateQuoteItems(description)` → Anthropic API → parst JSON aus Response → befüllt `appState.quoteItems`

**Modal-System:** `openModal(id)` / `closeModal(id)` toggle `.open` auf `.modal-overlay`. Jedes Modal hat eine eigene `open<Entity>Modal(id)` Funktion die State schreibt + Modal öffnet.

**PDF-Generierung:** `printQuotePDF(quoteId)` / `printInvoicePDF(invoiceId)` bauen vollständiges HTML-Dokument mit `getPDFStyles()` (Inline-Print-CSS), öffnen Blob-URL in neuem Tab, rufen `window.print()` auf.

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
--bg: #0f1318          /* Haupt-Hintergrund */
--bg-elevated: #161b22 /* Modals, Bottom-Nav */
--surface: #1c2430     /* Cards */
--surface-2: #1e2a38   /* Hover-States */
--accent: #219CE8      /* Primärfarbe — Buttons, aktive Nav, Links */
--gold: #ED9434        /* Seltener Akzent — Gesamtbetrag in PDFs */
--text: #ffffff
--muted: #7a9bb8
```

## PWA

- `manifest.json` + `sw.js` — Cache-Name `craftiq-v1`
- Bei Änderungen an `index.html` oder `icon.png`: Cache-Name in `sw.js` erhöhen, sonst sehen Nutzer alte Version
- `safe-area-inset-*` im CSS für iPhone Notch

## KI-Integration

`generateQuoteItems(description)` in `index.html` — sendet Auftragsbeschreibung an Claude Haiku, erwartet JSON-Array mit Positionen zurück. Bei Fehler: Toast mit Meldung, manuelle Positionseingabe bleibt verfügbar. Das Modell und der API-Key sind direkt als Konstanten im JS-Block hardcoded (Demo — kein Produktiveinsatz).
