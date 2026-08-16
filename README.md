# Aurum

A savings and goal-tracking app that turns *"I want this"* into progress you can
watch. Name something you're saving for, put money aside when you have it, and
the app works out the milestones, the pace you'd need, and when to make a fuss
about how far you've come.

Everything lives in your browser. No account, no server, no sign-up.

```bash
npm install
npm run dev
```

---

## What it does

- **Goals** — unlimited (soft-capped at 60), each with its own target, colour,
  priority, notes, transaction history and milestone ladder.
- **Add and spend** — one sheet, one required field. Quick-amount chips scale to
  the goal, and a live preview shows the new balance and any milestone the
  contribution would unlock *before* you commit.
- **Milestones** — generated from the goal's shape, not hard-coded. A ₹2,000
  goal celebrates ₹500; a ₹1,00,000 goal celebrates ₹25,000. Reaching one
  triggers a celebration with confetti, a count-up and an optional sound cue.
- **Target dates** — the app computes what you'd need per week and per month,
  compares it against the pace you've actually been saving at, and projects a
  completion date. A date that has passed is a normal state, reported plainly.
- **History** — every entry is editable and deletable, with filters. All
  balances recalculate from the log.
- **Insights** — savings curve, monthly money in/out, and where your balance
  currently sits, each with a table view.
- **Achievements** — completed goals move to their own area and stay there.
- **Product search** — optional; see [Product search](#product-search) below.

---

## Architecture

```
src/
  domain/      Types, milestone generation, derived selectors, pace and analytics maths
  store/       Commands (all mutations), persistence, React store
  services/    Sound, haptics, the product-search provider interface
  lib/         Money, dates, ids
  router/      A small history router
  components/  UI primitives, charts, and composed pieces
  views/       One file per screen
  styles/      Design tokens and base styles
```

Three rules hold the whole thing together.

### 1. Money is integers

Every monetary value is an integer count of the currency's **minor units** —
paise for INR, cents for USD, whole yen for JPY. No float ever touches a
balance.

Parsing works on the digit strings either side of the decimal point rather than
multiplying by 100, so `19.99` becomes exactly `1999` instead of
`1998.9999999999998`. `src/lib/money.ts` is the only place that converts, and it
rejects ambiguous input rather than guessing.

### 2. Nothing derived is ever stored

A goal has no `savedAmount` field. Balances, percentages, remaining amounts,
completion state and totals are all recomputed from the transaction log every
time they're read (`src/domain/selectors.ts`).

This is what makes editing a three-week-old contribution safe: there is no
second copy of the number to fall out of step. The one piece of bookkeeping that
*is* persisted — which milestones have already been celebrated — is reconciled
against the balance after every change, so lowering a balance below a milestone
lets it celebrate again when you re-reach it.

### 3. Every mutation goes through one function

`applyCommand(data, command)` in `src/store/commands.ts` is pure: it takes the
document and a command and returns either a new document or a human-readable
failure. Nothing else in the app mutates state.

That's what guarantees the invariants regardless of which screen fired the
change:

- a balance can never go negative — a spend larger than the balance is refused
  with the real figure in the message, and so is an *edit* or *delete* that
  would have the same effect
- a goal is marked complete only when it's genuinely funded, and un-marked if it
  drops back
- milestones cleared by a change are reported by the command itself, so a
  celebration can't be forgotten at a call site

`useCommands()` is the single bridge from commands to feedback (toast, sound,
haptics, celebration).

---

## Product search

The app can create a goal from a real product and suggest a target from its
current prices. **It never invents a price.** With no provider configured it
says so and routes you to entering a target by hand — which is what this build
does by default.

### Wiring a provider

Set the endpoint at build time:

```bash
VITE_PRODUCT_SEARCH_ENDPOINT=https://your-service.example/search \
VITE_PRODUCT_SEARCH_KEY=optional-bearer-token \
npm run build
```

Your endpoint receives:

```
GET <endpoint>?q=<query>&currency=<ISO-4217>
Authorization: Bearer <key>        # only when a key is configured
```

and answers with:

```json
{
  "results": [
    {
      "id": "ps5-slim",
      "title": "PlayStation 5 Slim",
      "brand": "Sony",
      "description": "…",
      "imageUrl": "https://…",
      "url": "https://…",
      "variants": ["Digital Edition"],
      "offers": [
        { "retailer": "Retailer A", "price": 49999.0, "currency": "INR", "url": "https://…" }
      ]
    }
  ]
}
```

`price` is in **major** units as a JSON number. Only `title` and at least one
well-formed offer are required; everything else is optional, and anything
malformed is discarded rather than guessed at. Non-`http(s)` URLs are rejected.

To try it locally:

```bash
npm run mock:search      # serves the contract above on :5174
VITE_PRODUCT_SEARCH_ENDPOINT=http://localhost:5174/search npm run build
npm run preview
```

### How a target is chosen

Not the first result, and not the cheapest. Listings are noisy — a mispriced
accessory shouldn't set someone's savings target — so the app takes the
**median** across offers, discards anything more than 45% away from it, and
takes the median again over what's left
(`chooseRepresentativePrice` in `src/services/productSearch.ts`).

Whatever it lands on is presented as an *estimate*:

- every source listing is disclosed, including the ones excluded
- the retrieval timestamp is shown, with a note that prices change
- the target field is editable before the goal is created
- the goal keeps a `PriceSnapshot`, never a live dependency on the retailer

### Adding a different provider

Implement `ProductSearchProvider` and hand it to `setProductSearchProvider()`.
The UI consumes the interface, not the transport, so no screen changes.

---

## Design system

Tokens live in `src/styles/tokens.css`. No component defines a raw hex value, a
raw duration, or a raw pixel radius.

- **Type** — Fraunces (a display serif with an optical-size axis) for headings
  and money; Manrope for UI text. Both self-hosted, so there's no runtime
  dependency on a font CDN. Money always uses tabular figures — without them a
  counting-up number jitters as glyph widths change.
- **Colour** — a warm ivory light theme and a deep ink dark theme, one gold
  accent, and six goal accents so each goal feels like its own space. Dark mode
  is defined twice, deliberately: once under `prefers-color-scheme` (guarded so
  an explicit light choice wins) and once under `[data-theme="dark"]`.
- **Charts** get their own palette, separate from the UI accents. The goal
  accents are identity tints, always paired with a goal name; they do not
  survive a colourblind-separation check when plotted side by side. The chart
  colours were picked against that check — the money-in/money-out pair passes
  CVD separation, chroma, lightness and 3:1 contrast in both themes, and
  polarity is carried by position (above/below the baseline) before colour.
- **Motion** — under reduced motion, durations collapse to ~0 rather than
  transitions being deleted, so state changes stay traceable. Confetti doesn't
  render at all. Both the OS setting and the in-app setting are honoured.

### Accessibility

Modals are native `<dialog>` + `showModal()`, so focus trapping, Escape, `inert`
backgrounds and top-layer stacking come from the platform. Filters and theme
pickers are real radiogroups; the sound toggle is a real checkbox. Every field
goes through one `Field` wrapper that wires label, hint and error together.
Nothing is communicated by colour alone — transaction direction carries a
`+`/`−` sign, progress is stated as text beside every bar, and the current nav
item gets a marker as well as a tint.

---

## Testing

```bash
npm run check        # typecheck + lint + unit tests
npm test             # 98 unit tests
npm run e2e          # browser flows (needs `npm run preview` on :4173)
npm run e2e:search   # product search (needs the mock server + a build with the endpoint)
```

Unit tests cover the parts where being wrong is expensive: money parsing and
formatting, milestone generation across goal sizes, pace maths (including
divide-by-zero on the target date and dates in the past), the command
invariants, storage parsing of corrupt payloads, and the search provider's
normalization and failure paths.

The e2e scripts drive a real browser through goal creation, adding and spending,
the overspend guard, editing and deleting entries, completion and celebration,
keyboard access, dark mode, mobile layout and reload persistence. They caught
three real bugs during development, including a closed `<dialog>` that was
covering the page and swallowing every click.

---

## Notes and limits

- Data lives in `localStorage` under `aurum.savings.v1`. Clearing browser data
  removes it — Settings → Export writes a JSON backup.
- Storage being blocked (Safari private mode, storage disabled) is detected at
  startup and stated in the UI rather than discovered on refresh.
- Changing the currency in Settings affects new goals only. Existing amounts are
  never converted, because the app has no exchange-rate source and inventing one
  would be worse than leaving them alone.
- Sound is synthesized with the Web Audio API — no audio files, and no autoplay.
  The context is primed on the first real interaction; if audio is unavailable
  every cue becomes a silent no-op.
