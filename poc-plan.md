# PoC — Kombinovaný plánovač výletů na Madeiře

## Overview

Proof-of-concept webová aplikace pro plánování výletů na Madeiře kombinující veřejnou dopravu a turistické trasy.

**Přidaná hodnota oproti Google Maps:** Google Maps umí autobus nebo pěší chůzi zvlášť. Tato aplikace plánuje celý výlet dohromady — najde pěší trasy mezi dvěma body, uživatel si vybere jednu, a aplikace pak naplánuje dopravu tam i zpět. Pro turisty kteří neznají oblast a chtějí jednoduché "jak se dostanu na start trasy a zpátky?"

**UX flow — dva kroky:**

**Krok 1 — Vyhledání pěší trasy:**
1. Uživatel zadá **startovací bod** (odkud chce začít pěší tůru) a **cílový bod** (kde chce tůru skončit)
2. Aplikace vyhledá v OSM pěší trasy procházející nebo spojující tyto dva body
3. Zobrazí několik variant tras (název, délka, převýšení)
4. Uživatel si jednu variantu vybere

**Krok 2 — Plánování dopravy:**
1. Uživatel zadá **výchozí bod** (ubytování) a **čas odjezdu**
2. Aplikace naplánuje:
   - 🚌 Hromadná doprava: ubytování → startovací bod tůry
   - 🥾 Vybraná pěší trasa
   - 🚌 Hromadná doprava: cílový bod tůry → ubytování

**Stack:** Next.js (App Router) · TypeScript · shadcn/ui (preset `b5JgPIY6C`) · Google Maps Directions API (transit mode) · OpenStreetMap Overpass API (turistické trasy)

**Deploy:** Cloudflare Workers via OpenNext.js (`@opennextjs/cloudflare`) + Wrangler CLI

**Rozsah PoC:** Dvoustupňový plánovač — OSM vyhledávání tras + itinerář hromadná doprava+trasa+hromadná doprava. Žádná autentizace, žádné ukládání, žádný real-time tracking.

**Omezení:** Google Maps Transit pokrytí Madeiry je omezené — výsledky autobusů mohou být prázdné pro lokální linky. PoC toto odkryje pro budoucí rozhodnutí o zdroji dat.

**Credentials:** Cloudflare tokeny a přihlašovací údaje jsou uloženy v `~/.env` na vývojářově stroji. Před každým příkazem který tyto credentials využívá (wrangler deploy, wrangler secret put, atd.) je nutné si nechat potvrdit od uživatele, že chce příkaz spustit.

---

## Sub-Tasks

---

### 1. Scaffold projektu

**Intent:** Inicializovat Next.js projekt pre-konfigurovaný pro Cloudflare Workers pomocí `create-cloudflare`, nainstalovat shadcn/ui pomocí daného presetu.

**Expected Outcomes:**
- Funkční Next.js projekt se spustitelným `npm run dev` (local Next.js dev server)
- `wrangler.jsonc`, `open-next.config.ts` a skripty (`build`, `deploy`, `preview`) připraveny
- shadcn/ui inicializováno s presetem `b5JgPIY6C`
- Základní adresářová struktura (`app/`, `components/`, `lib/`)

**Todo List:**
- [ ] Scaffoldovat projekt: `npm create cloudflare@latest . -- --framework=next` — vygeneruje Next.js + OpenNext Cloudflare pre-config včetně `wrangler.jsonc`
- [ ] Přidat do `next.config.mjs` volání `initOpenNextCloudflareForDev()` pro lokální dev s Cloudflare runtime
- [ ] Spustit shadcn init s presetem: `npx shadcn init --preset b5JgPIY6C`
- [ ] Přidat potřebné shadcn komponenty: `button`, `input`, `card`, `badge`, `separator`, `scroll-area`, `stepper` (nebo vlastní step indikátor)
- [ ] Ověřit že `npm run dev` funguje

**Relevant Context:**
- Kořen repozitáře je prázdný (jen LICENSE), vše se inicializuje od nuly.
- `create-cloudflare --framework=next` vygeneruje `wrangler.jsonc` s `main: ".open-next/worker.js"`, `ASSETS` binding, `WORKER_SELF_REFERENCE` service binding a compatibility flags `nodejs_compat`.
- Lokální env proměnné jdou do `.dev.vars` (ne `.env.local`), v produkci se používají Wrangler secrets.

**Status:** `[ ] pending`

---

### 2. Krok 1 — Vyhledání pěších tras z OSM

**Intent:** Uživatel zadá startovací a cílový bod tůry. Aplikace vyhledá v OpenStreetMap pěší trasy které tyto body spojují nebo leží v jejich blízkosti, a zobrazí několik variant k výběru.

**Expected Outcomes:**
- Formulář pro zadání startovacího bodu (textová adresa nebo název místa) a cílového bodu
- Route Handler `app/api/trails/route.ts` přijme start + cíl, dotáže Overpass API a vrátí relevantní trasy
- Každá trasa obsahuje: název, délku (km), převýšení (pokud dostupné v OSM), souřadnice skutečného startu a konce trasy
- Komponenta `components/trail-results.tsx` zobrazí varianty tras — uživatel kliknutím vybere jednu
- Výběr trasy posouvá uživatele do Kroku 2

**Todo List:**
- [ ] Vytvořit TypeScript typ `Trail` (`id`, `name`, `distance`, `ascent`, `startLat`, `startLon`, `endLat`, `endLon`, `tags`)
- [ ] Vytvořit `lib/overpass.ts` s funkcí `searchTrails(startLat, startLon, endLat, endLon)` — dotaz na Overpass API pro pěší relace (`route=hiking`) v bboxu ohraničujícím zadané body (s rozumným bufferem ~5 km)
- [ ] Geocoding vstupních adres na souřadnice: použít **Nominatim API** (`https://nominatim.openstreetmap.org/search?q=...&format=json`) — zdarma, bez klíče, pokrývá Madeiru
- [ ] Vytvořit `app/api/trails/route.ts` — POST handler, přijme `{ startAddress, endAddress }`, geocoduje adresy, zavolá Overpass, vrátí seznam tras
- [ ] Vytvořit `components/trail-search-form.tsx` — dva inputy (Start tůry, Cíl tůry) + tlačítko Vyhledat trasy
- [ ] Vytvořit `components/trail-results.tsx` — seznam karet s variantami tras; kliknutí = výběr + přechod na Krok 2
- [ ] Přidat loading a empty stav ("Nenalezeny žádné trasy mezi zadanými body")

**Relevant Context:**
- Overpass API endpoint: `https://overpass-api.de/api/interpreter`
- Overpass QL pro trasy v bbox (bbox = min/max souřadnic start+cíl + buffer):
  ```
  [out:json];
  relation["route"="hiking"](bbox_south,bbox_west,bbox_north,bbox_east);
  out body;
  ```
- Nominatim geocoding: `GET https://nominatim.openstreetmap.org/search?q=Funchal&format=json&limit=1` → vrátí `lat`, `lon`
- Nominatim vyžaduje `User-Agent` header v requestu (pravidla použití)
- Filtrování relevance: trasy jejichž geometrie leží v blízkosti zadaných bodů (pro PoC stačí bbox filtr)
- Overpass a Nominatim jsou zdarma, bez API klíče

**Status:** `[ ] pending`

---

### 3. Krok 2 — Plánování hromadné dopravy (Google Maps API)

**Intent:** Po výběru trasy uživatel zadá výchozí bod (ubytování) a čas odjezdu. Aplikace naplánuje dopravu tam (ubytování → start trasy) i zpět (konec trasy → ubytování) pomocí Google Maps Directions API.

**Expected Outcomes:**
- `.dev.vars.example` se vzorem pro `GOOGLE_MAPS_API_KEY=`
- Route Handler `app/api/directions/route.ts` provede dvě volání Google Maps API (tam + zpět) a vrátí oba itineráře
- API klíč nikdy neprojde do klienta
- V produkci se klíč nastaví přes `wrangler secret put GOOGLE_MAPS_API_KEY`

**Todo List:**
- [ ] Vytvořit `.dev.vars.example` s `GOOGLE_MAPS_API_KEY=`
- [ ] Ověřit že `.dev.vars` je v `.gitignore`
- [ ] Vytvořit `lib/directions.ts` s TypeScript typy (`DirectionsRequest`, `DirectionsRoute`, `TransitStep`)
- [ ] Vytvořit `app/api/directions/route.ts` — POST handler:
  - Přijme `{ accommodationAddress, trailStartLat, trailStartLon, trailEndLat, trailEndLon, departureTime }`
  - Volání 1 (tam): `origin=accommodationAddress` → `destination=trailStartLat,trailStartLon`, `mode=transit`, `departure_time`
  - Volání 2 (zpět): `origin=trailEndLat,trailEndLon` → `destination=accommodationAddress`, `mode=transit`, `departure_time` = odhadovaný čas dokončení tůry (departure_time + délka trasy v minutách)
  - Vrátí `{ toTrail: DirectionsRoute, fromTrail: DirectionsRoute }`
- [ ] Přidat základní validaci vstupů

**Relevant Context:**
- Na Cloudflare Workers jsou env proměnné dostupné přes `process.env` (OpenNext mapuje z CF `env` objektu)
- Lokální dev: `.dev.vars` → `getPlatformProxy()` → `process.env`
- Produkce: `wrangler secret put GOOGLE_MAPS_API_KEY`
- Google Directions API: `GET https://maps.googleapis.com/maps/api/directions/json?origin=...&destination=...&mode=transit&departure_time=...&key=...`
- `departure_time` musí být Unix timestamp (ne v minulosti)
- Odhad času zpáteční dopravy: `departureTime + (trail.distance / 4km/h * 60)` — hrubý odhad tempa pro PoC

**Status:** `[ ] pending`

---

### 4. Hlavní UI — dvoustupňový wizard

**Intent:** Sestavit celé UI jako dvoustupňový wizard: Krok 1 (hledám trasu) → Krok 2 (plánuji dopravu) → Výsledek (kompletní itinerář).

**Expected Outcomes:**
- `components/trip-wizard.tsx` — hlavní client komponenta řídící stav wizardu (aktuální krok, vybraná trasa, výsledky)
- **Krok 1:** formulář start+cíl tůry → seznam variant tras → výběr
- **Krok 2:** formulář ubytování+čas → zobrazení kompletního itineráře
- `components/itinerary.tsx` — zobrazí tři sekce: 🚌 Tam / 🥾 Trasa / 🚌 Zpět
- Vizuální indikátor kroků (Step 1 / Step 2)
- Prázdné stavy a error stavy pro oba kroky

**Todo List:**
- [ ] Vytvořit `components/trip-wizard.tsx` — client component se stavem:
  - `step: 1 | 2`
  - `trailSearchResult: Trail[]`
  - `selectedTrail: Trail | null`
  - `itinerary: { toTrail, fromTrail } | null`
  - `loading`, `error`
- [ ] Sestavit Krok 1 UI: `TrailSearchForm` + `TrailResults` (ze Sub-task 2), po výběru trasy přechod na krok 2
- [ ] Sestavit Krok 2 UI: input pro adresu ubytování + datetime picker (výchozí = zítra 9:00), tlačítko Naplánovat dopravu
- [ ] Implementovat `onPlanTransport` — zavolá `POST /api/directions` s ubytováním + souřadnicemi vybrané trasy
- [ ] Vytvořit `components/itinerary.tsx` — tři sekce jako Cards:
  - 🚌 **Tam:** linka, čas odjezdu, čas příjezdu na start trasy, přestupy
  - 🥾 **Trasa:** název, délka, odhadovaná doba
  - 🚌 **Zpět:** linka, čas odjezdu z konce trasy, čas příjezdu domů
- [ ] Přidat tlačítko "← Zpět na výběr trasy" v Kroku 2
- [ ] Přidat loading skeleton a error stav (včetně specifické hlášky když Google Maps nemá transit data)
- [ ] Sestavit `app/page.tsx` — nadpis + `TripWizard`

**Relevant Context:**
- shadcn komponenty: `Input`, `Button`, `Card`, `CardHeader`, `CardContent`, `Badge`, `Separator`
- Wizard stav žije v `TripWizard` — předává se dolů jako props do child komponent
- Pokud Google Maps vrátí prázdné `routes[]`: "Nepodařilo se najít spoje hromadnou dopravou. Data pro Madeiru mohou být neúplná."
- Krok 2 musí znát souřadnice vybrané trasy (startLat/Lon, endLat/Lon) z Kroku 1

**Status:** `[ ] pending`

---

### 5. Cloudflare deploy + finalizace

**Intent:** Nasadit PoC na Cloudflare Workers a zdokumentovat celý workflow pro tým.

**Expected Outcomes:**
- Aplikace nasazena na Cloudflare Workers a přístupná přes `*.workers.dev` URL
- `README.md` pokrývá lokální vývoj i deploy workflow
- `.dev.vars.example` správně vyplněný

**Todo List:**
- [ ] Spustit `wrangler secret put GOOGLE_MAPS_API_KEY` *(vyžaduje potvrzení)*
- [ ] Spustit `npm run deploy` (`opennextjs-cloudflare build && opennextjs-cloudflare deploy`) *(vyžaduje potvrzení)*
- [ ] Ověřit end-to-end průchod na produkční URL: Krok 1 → výběr trasy → Krok 2 → itinerář (nebo prázdný stav)
- [ ] Napsat `README.md`: prerekvizity (Node.js, Wrangler CLI, Cloudflare účet, Google Maps API klíč), `npm install` + `npm run dev`, postup pro deploy
- [ ] Zkontrolovat že `npm run build` proběhne bez chyb

**Relevant Context:**
- `npm run deploy` = `opennextjs-cloudflare build && opennextjs-cloudflare deploy`
- `npm run preview` = lokální preview s Wrangler (simuluje Workers runtime)
- Wrangler secrets jsou v Workers runtime dostupné přes `process.env`
- Credentials jsou v `~/.env` na vývojářově stroji — před spuštěním wrangler příkazů vždy potvrdit

**Status:** `[ ] pending`

---

## Open Questions

- **Relevance OSM tras:** Overpass bbox filtr vrátí trasy v oblasti — pro PoC dostačující. Přesnější filtrování (trasy skutečně procházející startem/cílem) je složitější a patří za PoC.
- **Google Maps pokrytí Madeiry:** Pokud API vrátí prázdné výsledky, bude potřeba zvážit GTFS nebo jiný zdroj. Toto PoC odkryje.
- **Délka tras z OSM:** Některé relace nemusí mít `distance` tag — zobrazit "délka neznámá" nebo odhadnout z geometry (mimo rozsah PoC).
- **Odhad času zpáteční dopravy:** Používáme fixní tempo 4 km/h — nepřesné, ale pro PoC dostačující.
