# Virtual Library Build Guide

> **What this is.** This document is a **build brief you hand to a vibe-coding platform** (Lovable, Bolt, v0, Claude Code, Cursor, Replit, Windsurf, or a human developer). Attach it in chat alongside your Goodreads export, then let the platform build the site. The user is not expected to write code — only to export their Goodreads library (Section 4) and attach that file.

> **If you're the AI platform / assistant reading this:** you are the builder. Build the site described below in your project's existing stack. React + TypeScript + Tailwind CSS is assumed; any React framework (TanStack Start, Next.js, Remix, Vite SPA) works. Translate the Vite/TanStack file paths here to your host conventions — don't fight them. Do **not** swap the design, fonts, or motion values for defaults. Do **not** invent placeholder books: if no Goodreads CSV is attached yet, stop after the core shelf scaffolding and ask the user to attach `goodreads_library_export.csv`, then run the import in Section 4 before populating `books`. Everything else in this guide is your job to implement.

This guide is **platform-agnostic**. Anywhere it says "the platform", "the AI", or "your editor", it means whichever tool received this brief. Platform-specific details (backend, AI key, how to attach a file to chat) are isolated in Section 2 so nothing else in the guide depends on them.


---

## 1. What you're building

A single-page personal bookshelf:

- A horizontal, infinitely scrolling rail of real books, each rendered in 3D with its actual cover art and a spine color sampled from that cover.
- Hover a book and it pulls toward you, with a floating metadata card (title, author, year, binding, rating, genres).
- Click a book and it physically slides out of the shelf into a large front-cover detail view, with the shelf blurred behind it.
- An AI-powered **"What are you looking for?"** search bar that understands plain English ("cozy fantasy romance with faeries"), plus genre filter pills.
- A **"Recommend a book"** button that lets visitors search real books and send you recommendations, which animate onto a second "Recommended to me" shelf.
- A typed-in heading: *Welcome to my library*.

### Design ethos

Ethereal, editorial, dreamlike — but warm rather than cold-blue:

- Warm neutral **paper tones** (oklch) for the background, not dark blue.
- Serif display type for titles, a clean sans for body, and tiny uppercase **mono** labels for metadata/dates/counts.
- Motion is slow and floating (drift, rise, soft fades). No bouncy springs.

### Fonts

Loaded once via a `<link>` in `src/routes/__root.tsx`:

- **Cormorant Garamond** (display serif)
- **Karla** (body sans)
- **Space Mono** (labels)

The exact Google Fonts URL (in `__root.tsx`):

```
https://fonts.googleapis.com/css2?family=Cormorant+Garamond:ital,wght@0,300;0,400;0,500;1,300;1,400&family=Karla:wght@300;400;500&family=Space+Mono:wght@400;700&display=swap
```

---

## 2. Prerequisites — the three things your platform must supply

The site needs exactly three platform-dependent things. Everything else is plain React code.

| # | What's needed | Why | How to satisfy it |
|---|---|---|---|
| 1 | A **React + TypeScript + Tailwind** project | The whole UI | Any React framework: TanStack Start, Next.js, Remix, or a plain Vite SPA. Keep whatever the platform gives you. |
| 2 | A **server-side place to call an AI model** | Natural-language search must hide the API key | TanStack server function, Next.js route handler / server action, Remix action, Express route — any server endpoint. |
| 3 | A **Postgres database with a public-insert table** *(optional)* | Only for the visitor "Recommend a book" shelf | Supabase, Neon, Replit DB/Postgres, or any Postgres. Skip it entirely if you don't want visitor recommendations. |

### Platform notes

- **Lovable** — new project starts on TanStack Start v1 + React 19 + Vite 7 + Tailwind v4. Enable **Lovable Cloud** for the database. The AI key `LOVABLE_API_KEY` is auto-provisioned; the gateway is `https://ai.gateway.lovable.dev/v1/chat/completions` (OpenAI-compatible).
- **Claude Code / Cursor / Windsurf** — scaffold with `npm create vite@latest -- --template react-ts` plus Tailwind, or any React framework you prefer. Add your own `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` to `.env` and use that provider in Section 5.
- **Replit** — use a React + Vite template; store the AI key in Secrets; use Replit's Postgres for the recommendations table.
- **Bolt / v0** — same as above; for v0 (Next.js), put the search endpoint in `app/api/search/route.ts`.

### Everything is optional except the shelf

The **static book data + 3D shelf + detail pull-out** is the core. The **AI search** needs item 2. The **recommendations shelf** needs item 3. Build the core first; the site is complete and beautiful without the other two.



---

## 3. The Book data model

Every book is a plain object in `src/data/books.ts`. The personal library is **one static array** — this file is the single source of truth for both the shelf and the AI search. The type:

```ts
export type Book = {
  id: string;
  title: string;
  author: string;
  genres?: string[];          // genre tags shown as filter pills
  cover: string;               // real cover art (Open Library URL)
  year: number;
  blurb: string;
  rating: number;              // 0 means unrated
  finished: string;            // e.g. "Jul 2026"
  recommender?: string;        // set only when a visitor recommended it
  publisher: string;
  binding: "hardcover" | "paperback" | "mass";
  finish: "cloth" | "gloss" | "matte";   // spine surface material
  spine: string;               // base color, sampled from the cover's left edge
  band?: string;               // accent pulled from the cover art
  ink: string;                 // lettering color
  face: "serif" | "sans" | "mono";       // lettering style
  caps?: boolean;
  width: number;               // spine width in px
  height: number;              // spine height in px
  lean: number;                // degrees of lean on the shelf
  depth: number;               // how far forward/back the book sits, px
  wear: number;                // 0–1 edge wear and ink fade
  spineImage?: string;
};
```

A representative object (from the real library):

```ts
{
  id: "tuesdays-with-morrie-an-old--0",
  title: "Tuesdays with Morrie: An Old Man, a Young Man, and Life's Greatest Lesson",
  author: "Mitch Albom",
  genres: ["Nonfiction"],
  cover: "https://covers.openlibrary.org/b/id/12560417-L.jpg",
  year: 1997,
  blurb: "Tuesdays with Morrie is a memoir by American author Mitch Albom…",
  rating: 0,
  finished: "Jul 2026",
  publisher: "Warner",
  binding: "paperback",
  finish: "matte",
  spine: "#4c2215",
  band: "#7e5741",
  ink: "#faf7f0",
  face: "serif",
  caps: false,
  width: 19,
  height: 217,
  lean: 0,
  depth: -2.7,
  wear: 0.34,
}
```

The physical fields (`binding`, `finish`, `spine`, `band`, `ink`, `face`, `caps`, `width`, `height`, `lean`, `depth`, `wear`) are what make each spine look like a real, individual book rather than a flat rectangle. You don't write them by hand — see Step 4.

---

## 4. Import your Goodreads library (AI-assisted)

This is how you load **your own books**. You do **not** build an upload button — you hand your Goodreads export to whichever AI coding tool you're using and let it generate the data file from it.

### Step 4a — Export from Goodreads

1. Go to Goodreads → **My Books**.
2. On the left, **Import and export** → **Export Library**.
3. Wait for the email, then download `goodreads_library_export.csv` to your computer.

### Step 4b — Give the AI your export and generate `src/data/books.ts`

Two things must happen together: **the AI must have the file**, and **it must have the prompt below**.

1. **Get the CSV to the AI.**
   - *Lovable / Bolt / v0 / ChatGPT-style chat:* click the **attach (paperclip)** button in the composer and select `goodreads_library_export.csv`. Confirm it appears attached before sending.
   - *Claude Code / Cursor / Windsurf / any terminal-based agent:* copy the CSV into the project folder (e.g. `./goodreads_library_export.csv`) and mention that path in your message — the agent reads it from disk.
   - *Replit:* upload the CSV into the file tree, then reference it by path in the chat.
2. **Send this prompt** with it:

```text
Here is my Goodreads export (goodreads_library_export.csv — attached, or in
the project root). Please regenerate src/data/books.ts with my real library,
following these rules:


- Read only rows whose Exclusive Shelf is "read" or "currently-reading".
- For each book: title, author, year (first publish year), publisher, rating
  (my rating; 0 if blank), and finished date from "Date Read"
  (format as "Mon YYYY").
- Fetch the real cover art and a short blurb from Open Library by ISBN/title.
  Use cover URLs like https://covers.openlibrary.org/b/id/<id>-L.jpg.
- Sample each cover's LEFT-EDGE color into `spine`, and pick a saturated
  accent from the cover into `band`. Set `ink` to dark (#241f19) if the spine
  is light, else near-white (#faf7f0).
- Derive physical props from the page count:
    binding = pages > 420 ? "hardcover" : pages < 260 ? "mass" : "paperback"
    finish  = binding === "hardcover" ? "cloth" : ~50/50 gloss/matte
    height  = hardcover ~236–254px, mass ~196–210px, paperback ~214–230px
    width   = clamp(pages * 0.055 + small jitter, 16, 58)
- Assign genres using ONLY this set: Fiction, Nonfiction, Sci-Fi,
  Mystery & Thriller, Fantasy, Romance. Map "Literary Fiction" -> Fiction.
  Default untagged nonfiction to "Nonfiction".
- Deterministically vary `lean` (-5..0), `depth` (-7..7), `wear` (0..0.35),
  `face`, and `caps` per book using a hash of the Open Library key, so the
  shelf looks lived-in.
- Fallbacks: missing cover -> a palette default color and no cover image;
  unrated -> rating 0; missing publisher -> "".
- Keep the existing `Book` type exactly. Export `books` as a single array,
  newest finished date first. Overwrite src/data/books.ts entirely.
```

The AI will parse the CSV, fetch covers/descriptions, sample spine colors from the artwork, derive physical sizing, assign genres, and overwrite `src/data/books.ts`. Because the shelf and the AI search both read from this array, your library is live the moment the file is regenerated.

**Goodreads columns that matter:** `Title`, `Author`, `ISBN`, `Exclusive Shelf`, `My Rating`, `Date Read`, `Number of Pages`, `Publisher`, `Year Published`.

**Genre set (must match the filter pills):** `Fiction`, `Nonfiction`, `Sci-Fi`, `Mystery & Thriller`, `Fantasy`, `Romance`. If you change this set later, update it in `src/data/books.ts` (the `genres` arrays) — the pills are generated dynamically from whatever genres exist in the data.

---

## 5. AI natural-language search

One server endpoint, any AI provider. It reads the same `books` array the shelf uses, so it works the moment your library exists.

**What the endpoint does** (in this project: `src/lib/librarySearch.functions.ts`, a TanStack server function; elsewhere: a Next.js route handler, Remix action, or Express `POST /api/search`):

- Reads the AI API key from env **inside the handler** — never in browser code.
- Builds a catalogue string from `books`: `id :: title :: author :: year :: genres :: blurb` (blurb truncated to 160 chars).
- Sends one chat-completion request with a system prompt saying: match the reader's natural-language request against these catalogue lines and return ONLY `{"ids":["id1","id2"]}`, best match first, at most 20; empty array if nothing fits. Request JSON output (`response_format: { type: "json_object" }` on OpenAI-compatible APIs).
- Validates returned ids against real book ids (drops anything unknown) and maps errors to friendly messages (429 → "Too many searches…", 402 → "Search credits exhausted.").

**Provider options** — all OpenAI-compatible, so only the URL, key, and model name change:

| Platform | Endpoint | Env var | Model |
|---|---|---|---|
| Lovable | `https://ai.gateway.lovable.dev/v1/chat/completions` | `LOVABLE_API_KEY` (auto-provisioned) | `google/gemini-2.5-flash` |
| OpenAI | `https://api.openai.com/v1/chat/completions` | `OPENAI_API_KEY` | `gpt-4o-mini` |
| Anthropic | `https://api.anthropic.com/v1/messages` (different shape) | `ANTHROPIC_API_KEY` | `claude-haiku-4-5` |
| Google | `https://generativelanguage.googleapis.com/v1beta/openai/chat/completions` | `GEMINI_API_KEY` | `gemini-2.5-flash` |
| OpenRouter / Groq | their `/v1/chat/completions` | their key | any fast model |

Pick a cheap, fast model — the task is simple matching.

**No AI key at all?** Fall back to plain client-side filtering: lowercase-match the query against title, author, genres, and blurb. The UI is identical; only the ranking is dumber.

Because the catalogue is generated from `books` at request time, search reflects your imported library automatically. If you later move books into a database, point the endpoint at that data source instead.

---

## 6. Database (optional) — the recommendations table

Skip this section entirely if you don't want the visitor "Recommend a book" shelf. The **only** database piece is that shelf; your personal library is the static file from Step 4.

Any Postgres works (Supabase, Lovable Cloud, Neon, Replit). Run this SQL — the client reads and inserts with the project's anon/public key, so the grants and row-level policies below are what keep it safe. On a non-Postgres backend, mirror the same shape: public read, public insert with length limits, no update/delete.



```sql
CREATE TABLE public.recommendations (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at timestamptz NOT NULL DEFAULT now(),
  recommender text NOT NULL,
  note text,
  title text NOT NULL,
  author text NOT NULL,
  cover text NOT NULL DEFAULT '',
  year int NOT NULL DEFAULT 0,
  publisher text NOT NULL DEFAULT '',
  binding text NOT NULL DEFAULT 'paperback',
  finish text NOT NULL DEFAULT 'matte',
  spine text NOT NULL DEFAULT '#584f46',
  band text,
  ink text NOT NULL DEFAULT '#faf7f0',
  face text NOT NULL DEFAULT 'serif',
  caps boolean NOT NULL DEFAULT false,
  width int NOT NULL DEFAULT 30,
  height int NOT NULL DEFAULT 220,
  lean real NOT NULL DEFAULT 0,
  depth real NOT NULL DEFAULT 0,
  wear real NOT NULL DEFAULT 0
);

GRANT SELECT, INSERT ON public.recommendations TO anon;
GRANT SELECT, INSERT ON public.recommendations TO authenticated;
GRANT ALL ON public.recommendations TO service_role;

ALTER TABLE public.recommendations ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Recommendations are publicly readable"
  ON public.recommendations FOR SELECT TO anon, authenticated USING (true);

CREATE POLICY "Anyone can recommend a book"
  ON public.recommendations FOR INSERT TO anon, authenticated
  WITH CHECK (
    length(trim(recommender)) BETWEEN 1 AND 60
    AND length(title) BETWEEN 1 AND 300
    AND length(author) <= 200
    AND (note IS NULL OR length(note) <= 500)
  );
```

Notes:

- **GRANTs are mandatory** — without them the app can't read or write the table, even with policies.
- The two policies: anyone (anon or signed-in) can **read** all recommendations, and anyone can **insert** one with basic length limits (recommender 1–60 chars, title 1–300, author ≤200, note ≤500). There's no `UPDATE`/`DELETE` grant, so recommendations can't be edited or deleted through the public API.
- The columns mirror the physical fields on `Book`, so a recommended book can be shelved and rendered exactly like a finished book.

---

## 7. Component map & build order

Build in roughly this order. Each subsection names the file, its job, and the non-obvious details to get right.

### `src/data/books.ts`
The `Book` type (Section 3) and the `books` array (Step 4). Nothing else.

### `src/components/bookFaces.ts`
Shared constants for the cover plane. Defines `COVER_W = 178` (front-cover depth in un-scaled spine space) and a `faceFont` map (`serif → font-display`, `sans → font-sans`, `mono → font-mono`).

### `src/components/TypedTitle.tsx`
Types **"Welcome to my library"** one character every **95ms** in the italic display font, with a blinking caret that pulses once finished. `FULL = "Welcome to my library"`; renders an `<h1 aria-label={FULL}>` with the visible typed slice inside `aria-hidden`.

### `src/components/BookSpine.tsx`
The 3D spine. The key trick for stability:

- An **outer `<button>` stays stationary** as the hover/click hit target (it never transforms). This stops the book sliding out from under the pointer.
- An **inner `<span>` is the only thing that moves**, with `transformStyle: preserve-3d` and transform `rotateY(var(--ry)) rotateZ(lean) translateZ(pull+depth) translateY(lift)`.

Exact hover values: **pull = 96**, **lift = -26**, **lean = 0** while hovered (otherwise `book.lean`). On hover the button gets `zIndex: 40`.

Layers on the spine face (in order): cover-art wraparound (left edge), base color settle, head/foot rules in the `band` accent, vertical title, vertical author (only if `width >= 44`), publisher mark at the foot (only if `width >= 30`), material **texture** over the ink, cylindrical **sheen** (`finishSheen` per finish), edge **wear** (sun/wear gradient at `opacity: wear`), inset highlight. Putting texture & sheen *over* the lettering makes the type look printed into the material.

A hinged **front cover** face sits at `left-full`, `width: COVER_W`, `transformOrigin: left center`, `rotateY(90deg)`, showing the real cover image. A **page block** tops the spine (`rotateX(78deg)`); hardcovers get a `headband` strip.

**Hover metadata card:** portaled to `document.body` via `createPortal` so it floats above the filter bar. It's `fixed`, `z-[100]`, `w-[248px]`, positioned at the spine's top-center (`left: center, top: top-14`), translated up. Text sizes: title **20px**, author **15px**, metadata/genres **14px**. A 90ms leave-grace timer prevents flicker as the transform moves.

### `src/components/Shelf.tsx`
The scroll rail. Receives `books` and optional `justAdded` (a book id to animate in).

- **Seamless loop:** if total shelf width > **2600px**, render **3 copies** and keep the viewport in the middle copy (start at `scrollWidth/3`); wrap scroll position at segment boundaries. If the row is short (≤2600px), render **1 copy** — this prevents duplicate books during filtering.
- **Curved perspective:** on scroll/resize, compute each spine's `rotateY` from its distance to the viewport center. `--ry` up to **±34°**, eased with `pow(|t|, 1.35)` so the middle stays flat. `perspective: 1400px`, `perspectiveOrigin: 50% 65%`.
- **Horizontal wheel:** intercept vertical wheel deltas and apply to `scrollLeft` (`passive: false`).
- **Drag:** pointer down/move/leave update `scrollLeft`.
- **Arrow keys** (Left/Right by 320px) when no book is open.
- **Short-row centering:** a `ResizeObserver` sets `overflowing`; when not overflowing, center the row (`justify-center`) and hide the edge fade gradients.
- **`justAdded`:** find the freshly shelved book, `scrollIntoView({ inline: "center" })`, and apply the `animate-shelve-in` class (Section 8) for ~1.1s.
- **Pull-out origin:** `openAt(i)` captures the spine's `getBoundingClientRect()` into a `rect` (`{left, top, width, height}`) and passes it to `BookDetail`.
- Spacing: `gap-[2px]`, `items-end`, top padding `pt-16`, bottom `pb-6`.
- Below the rail: a thin center-line gradient + a soft ground shadow (`from-foreground/8`).

### `src/components/BookDetail.tsx`
The pulled-out book. Receives the captured `rect`.

- On mount, starts at the shelf pose `translate3d(0,0,0) scale(1) rotateY(-26deg)`, then a `requestAnimationFrame` flips `out=true` so the transition runs to the final pose `translate3d(dx, dy, 0) scale(scale) rotateY(-90deg)` (front cover facing you).
- Transition: **`transform 900ms cubic-bezier(0.16, 1, 0.3, 1)`**.
- Target position is responsive: `narrow = vp.w < 720`; cover height `min(vp.h*0.6, 480)` (mobile `vp.h*0.42`). `scale = coverH / rect.height`; `coverW = COVER_W * scale`.
- Backdrop blurs/dims (`bg-background/70 backdrop-blur-xl`), fading in over 700ms.
- Details panel fades/rises in with a **260ms delay**; shows recommender or "Finished {date}", title, author, blurb, star rating (or "Unrated"), and Previous/Next/Shelve-it controls.
- Keys: **Escape** = retract (then close after 620ms), **← / →** = prev/next.
- Fallbacks: missing cover → a typeset title/author card; rating 0 → "Unrated".

### `src/components/RecommendBookDialog.tsx`
The visitor recommendation modal.

- Live Open Library search, debounced **280ms**, min 2 chars, abortable.
- Result list shows cover thumbnail + title + author/year + a "Pick" affordance.
- After picking: cover, author/year, **Your name** (1–60) and **Why should I read it?** (≤500) fields.
- On submit: `buildBook(picked)` → `onRecommend(...)` → insert into the `recommendations` table. Errors show inline.
- Placeholder for the search field: **"Search by title or author…"**.

### `src/hooks/useRecommendations.ts`
Loads recommendations from the database client (Supabase client here; use whatever your platform provides): `select("*")`, newest first (`order("created_at", { ascending: false })`), `limit(200)`. `recommend(input)` inserts a row, prepends the returned book to local state, and sets `justAdded` for **1400ms** (driving the slide-in animation). Maps DB rows → `Book` via `toBook` (note: recommendation's `blurb` becomes the visitor's note, `finished` becomes "Recommended by {name}"). If you skipped Section 6, delete this hook and the second shelf.

### `src/components/LibraryFilter.tsx`
The search + genre controls.

- Debounced AI search: **600ms**, min 2 chars, race-guarded by a `reqId` ref. Calls the `smartSearch` server function; renders "Reading the shelves…", then "{n} found" or an error.
- Genre pills are built **dynamically** from the genres present in `books`, sorted by count. They're a single **horizontal no-wrap line** (scrollable, hidden scrollbar). Placeholder: **"What are you looking for?"**.
- Filtering: if an AI result set exists, intersect with the selected genre and **order by the AI's ranking**. Passes the visible list (or `null` = show all) up via `onChange`.

### `src/lib/openLibrary.ts`
Three exports used by the recommend dialog (and reused by the import logic conceptually):

- `searchBooks(q, signal)` → calls `https://openlibrary.org/search.json?q=…&limit=12&fields=key,title,author_name,first_publish_year,cover_i,number_of_pages_median,publisher`; keeps docs with a title + cover; cover URL `https://covers.openlibrary.org/b/id/<cover_i>-L.jpg`.
- `readCoverPalette(src)` → loads the cover with `crossOrigin="anonymous"`, draws to an 80px-wide canvas, samples the **left 6%** (`edgeW = round(w*0.06)`) averaged into `spine`, finds the most saturated mid-luminance pixel into `band`, and sets `ink` dark if the spine is light (`lum > 0.55`) else near-white.
- `buildBook(result)` → derives physical props **deterministically** from a hash of the Open Library key: binding/finish from page count, height per binding, width `clamp(pages*0.055 + jitter, 16, 58)`, lean/depth/wear/face/caps, and the cover palette. Falls back to an HSL palette if cover sampling fails.

### `src/lib/librarySearch.functions.ts`
Covered in Section 5.

### `src/routes/index.tsx` (the home page)
Composes everything:

- `grain` background with a radial warm gradient overlay.
- Header: tiny uppercase "A personal archive" label, `<TypedTitle />`, a "{n} volumes" counter, a **"Recommend a book"** button, and `<LibraryFilter />`.
- Main `<Shelf books={shown} />` (or "No books match" empty state).
- When recommendations exist, a second "Recommended to me" shelf: `<Shelf books={recommendations} justAdded={justAdded} />`. Tight spacing between the two shelves (main `pb-2`, recommended header `pt-2`).
- `<RecommendBookDialog />` toggled by the button.
- SEO `head()`: title **"Your virtual library"** (change to your own), description, `og:title/description`, `og:type=website`, `twitter:card=summary_large_image`.

### `src/routes/__root.tsx`
Loads the Google Fonts `<link>`, the stylesheet, favicon; sets base `<html lang="en">`, viewport, and a default title/description (override per route in `index.tsx`). Keeps `<Outlet />` so child routes render.

### `src/styles.css`
Tailwind v4 (`@import "tailwindcss"` + `@source "../src"`). Key pieces:

- `@theme inline` maps the font tokens (`--font-display/sans/mono`) and color tokens (`--color-background`, `--color-primary`, …, `--color-glow`) to utilities.
- `:root` warm neutral palette in **oklch** (e.g. `--background: oklch(0.895 0.004 95)`, `--foreground: oklch(0.28 0.008 80)`, `--primary: oklch(0.5 0.085 52)`). No dark-blue dominance.
- `@utility grain` — an SVG fractal-noise overlay via `::after` at `opacity:0.35`, `mix-blend-mode: overlay`.
- `@utility no-scrollbar` — hides scrollbars.
- Keyframes/animations: `drift` (background haze), `rise`, and **`shelve-in`** — the gap opens to `--spine-w` while the book glides in from the right (`translateX/translateZ/rotateY/rotate`) and settles; exposed as `@utility animate-shelve-in { animation: shelve-in 1100ms cubic-bezier(0.22,1,0.32,1) both; }`.
- Spines use `transformStyle: preserve-3d` (set inline in components).

---

## 8. Customize & publish

- **Your name / title:** edit the typed string in `TypedTitle.tsx` and the SEO `head()` in `index.tsx`.
- **Hover feel:** in `BookSpine.tsx`, change `pull` (96) and `lift` (-26).
- **Pull-out speed:** in `BookDetail.tsx`, the `900ms` transition (and the 620ms retract delay).
- **Genre set:** edit the `genres` arrays in `books.ts`; pills regenerate automatically. Keep them consistent.
- **Color palette:** adjust the oklch tokens in `:root` of `styles.css`; never hardcode colors in components.
- **Publish:** deploy however your platform does it (Lovable/Replit/Bolt: one-click publish; elsewhere: Vercel, Netlify, or Cloudflare — the app is a standard React build). Verify the `og:`/`twitter` metadata reflects your title before publishing.

---

## 9. Quick reference — the import prompt

```text
I'm attaching my Goodreads export (goodreads_library_export.csv). Regenerate
src/data/books.ts with my real library. Read only "read"/"currently-reading"
rows; fetch real covers + short blurbs from Open Library; sample each cover's
left edge into `spine` and a saturated accent into `band`; derive binding/
finish/height/width from page count; assign genres from ONLY {Fiction,
Nonfiction, Sci-Fi, Mystery & Thriller, Fantasy, Romance} (Literary Fiction
-> Fiction); vary lean/depth/wear/face/caps deterministically per book; fall
back gracefully on missing cover/rating/publisher. Keep the existing Book
type; export `books` newest-first and overwrite src/data/books.ts entirely.
```

**Before you send this prompt, make sure the AI actually has `goodreads_library_export.csv`** — attach it with the paperclip button in a chat-based tool, or drop it in the project root and name the path for a terminal-based agent. The prompt alone won't work; the AI needs the file to read your books. Your shelf is live the moment `src/data/books.ts` regenerates, and AI search picks it up automatically.
