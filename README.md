# ONE DECK
# TCG Collection Tracker: Specification (Draft v0.17)

Status: draft. Sections marked **[OPEN]** still need a decision. The app is for personal use first, with no public release planned for now; items marked as launch requirements only apply before any public release. Based on a full review of the existing Unity prototype (One Piece TCG) and its real collection data (about 5,800 copies).

## 1. Vision

A multi-game trading card collection tracker built around **individual physical copies**, not quantities. Every card and every product is its own record with full provenance (where it came from, what it cost, what it was worth that day) and a physical location (which binder, box, or deck it lives in).

Headline features:
- **Fast logging.** Precons add their full contents in one step, random packs are logged on a single screen guided by their expected rarity mix, and huge second-hand lots can be logged across many sessions.
- **Net position.** A clear answer to "how much have I spent on this game, what is my collection worth, and am I up or down".
- **Per-copy value tracking.** Cost basis and market value at acquisition for every copy, compared with current value.
- **Trades** with a live value balance between both traders.
- **Deck management.** Decklists with versions, physical decks, and where the missing cards are.
- **Pull stats.** How hits are really distributed across packs, boxes, and cases.

First supported game: **Riftbound**. The architecture must support adding more games (One Piece TCG next, including migrating the prototype's data) without schema rewrites.

## 2. Goals and non-goals

### MVP goals
- **Catalog Studio**: an internal companion app for downloading source data and images, confirming variant and Cardmarket matches by hand, and publishing catalog releases to the main app.
- **Daily prices**: Cardmarket price guide downloaded automatically every day.
- **Catalog**: Riftbound cards browsable, searchable, and filterable, with images from imported image packs available offline.
- **Logging**: products of any kind (packs, boxes, cases, precons, bundles, lots), singles, gifts, and prizes, with acquisition data, including partial boxes and cases.
- **Opening**: instant add for fixed contents, profile-guided cracking for random packs, fast bulk entry for lots.
- **Sessions**: every long flow autosaves and can be paused and resumed.
- **Organization**: containers with bulk move, a master set container, and trade containers.
- **Value**: per-copy cost basis and market value at acquisition, and a headline net position.
- **Trades**: trade builder with a live value balance and trade history.
- **Surplus**: a view of copies beyond what the user wants to keep, with protected copies.
- **Lending and borrowing**: track cards lent to or borrowed from other players.
- **Decks**: versioned decklists, physical decks, pull lists, want lists.
- **Events**: lightweight logging of entry fees, results, and rewards.
- **Stats**: collection, completion, playsets, and personal pull stats.
- **Platform**: web first for fast development and testing, then iOS and Android once the web app is stable. Works offline and syncs across devices. Layouts are designed for phone-sized screens and touch from the start, since the app will mainly be used on phones (cracking at a table, trading at a store).

### Not in MVP (planned later)
- Card scanning.
- Additional games beyond Riftbound (One Piece is the second).
- Import of prototype One Piece data.
- Price history charts and sell alerts.
- Community pull data.
- Selling: sell candidates, marketplace export, and listed copies (6.10.2 and 6.10.3).
- A full deckbuilder (card discovery, synergy suggestions, meta stats).
- Social features, public collections, marketplace listing.
- Native iOS and Android apps (milestone 14, after the web app is stable). The MVP avoids web-only choices so this step is a port, not a rewrite.

## 3. Platform and stack

| Layer | Choice | Notes |
|---|---|---|
| App | Expo (React Native) + TypeScript | Web first (Expo web), then iOS and Android from the same codebase |
| Web hosting | Local development server only | Hosting decided later, if testing from other devices is needed |
| Navigation | expo-router | File-based routing |
| Backend | Supabase (hosted, free tier during development) | Postgres, auth, scheduled functions, storage for raw price files only |
| Local database | SQLite provided by PowerSync: WebAssembly SQLite in a web worker on web, native SQLite on mobile | Offline-first; no separate Expo SQLite setup |
| Images | Image packs produced by Studio, imported into the app and stored locally (browser storage on web, app storage on mobile) | No image hosting and no live source downloads, see 8.6 |
| Sync | PowerSync (Cloud free plan to start; free self-hosted Open Edition as fallback) | Syncs Supabase Postgres to local SQLite; see 3.1 |
| State | PowerSync reactive queries for all data; Zustand for UI and in-progress screen state | No separate server cache library in MVP |
| Catalog Studio | Local web app (TypeScript, React) run on the developer's computer, sharing game module packages | Run manually per set release, see 8.2 |
| Daily price job | One scheduled server function (Supabase Edge Function with pg_cron) | Downloads the price guide daily, see 8.4 |
| Repository | Monorepo: `apps/app` (Expo: web, then mobile), `apps/studio`, `packages/core`, `packages/game-*`, `supabase/` | Studio and app share schemas and game logic |
| Testing | Jest + React Native Testing Library | Unit tests for domain logic are mandatory |

### 3.1 Data layer

**Why PowerSync**
- It syncs the Supabase Postgres database to a local SQLite database on each client; the app reads and writes locally, which is exactly the offline-first, autosaving session model in 6.1.
- It has an official Supabase integration and SDKs for React Native and Web, with an official demo using one React Native codebase for mobile and web.
- Building custom sync (change tracking, conflict handling, retries) was the alternative and would be a large, bug-prone project on its own.

**How it is used**
- **Writes**: always to local SQLite, then uploaded by PowerSync to Supabase through a backend connector, with Supabase row level security ensuring users only touch their own rows.
- **Reads**: reactive local queries power every screen, so lists update instantly as cards are logged.
- **What syncs to clients**: the user's own collection data (all user tables in 5.3), the published catalog, and only the **latest** price per printing and product. Full price history stays on the server and is queried online only when a screen needs it (post-MVP charts).
- **Conflicts**: last write wins per row by default. Per-copy records and child-to-parent links keep real conflicts rare (two devices rarely edit the same copy at once).
- **UI state** (open tabs, selections, draft form inputs) lives in Zustand and is never synced. Anything a session must survive a restart with is written to the database, not Zustand.

**Known risks and mitigations**
- **React Native Web support is in beta** (available since React Native SDK 1.12.1 and Web SDK 1.8.0). Milestone 1 starts with a spike: PowerSync on Expo web with one table, offline writes, reload persistence, and sync to Supabase. If it fails, fall back to PowerSync's plain Web SDK behind the same data adapter.
- **Free plan inactivity**: PowerSync free instances with no deploys or client connections for 7 days are deprovisioned and can be restarted by redeploying, which re-syncs data to clients. Supabase free projects can also pause after inactivity. Acceptable for development; if pauses become annoying, move to the self-hosted Open Edition or a paid plan.
- **Supabase WAL growth**: PowerSync has noted an issue where idle Supabase instances using logical replication see excessive disk growth. Check the integration guide's current guidance when setting up.
- **Mobile builds**: PowerSync needs native modules, so the mobile apps (milestone 14) use Expo development builds, not Expo Go.

## 4. Core concepts

**Catalog**
- **Game**: a TCG (Riftbound, One Piece).
- **Set**: an internal grouping of printings, mapped to external set codes. Includes non-standard groups such as tournament packs or pre-release promos.
- **Card**: the abstract card (name, rules text, stats). Game data stored once.
- **Printing**: one specific physical version of a card (set, collector ID, rarity, variant, language, image). Physical copies always point to a printing.
- **Product type**: a kind of product (booster pack, booster box, case, precon, double pack, bundle) and what it contains.
- **Pack profile**: what a random pack can contain (its pool) and its expected composition (card count and rarity mix). Guides entry, never blocks it.

**Collection**
- **Acquisition**: the event of getting something (purchase, trade, prize, gift). Holds date, counterparty, and what was paid.
- **Product**: one physical product (this specific box), a user-defined lot, or a partial box or case used only for grouping.
- **Card copy**: one physical card.
- **Container**: a physical place cards or products live (binder, box, deck box).
- **Contact**: a store, organizer, or person the user buys from, trades with, or plays at.
- **Loan**: cards lent to or borrowed from a contact, with dates.
- **Surplus**: owned copies beyond what the user wants to keep, derived, never stored.
- **Session**: any in-progress long flow (cracking, lot entry, trade draft, deck pull), always autosaved.

**Activity**
- **Trade**: an exchange with a contact, with items and cash on both sides.
- **Event**: a tournament or other play event, with entry fee, results, and rewards.
- **Decklist**: a versioned template of what a deck should contain.
- **Deck**: a physical deck in a deck box, built from a decklist version.

## 5. Data model

General rules:
- All IDs are opaque UUIDs, never built from counts or source data.
- All timestamps are ISO 8601.
- Nulls are real nulls (no "N/A", "-", or empty strings). Unknown values are null, never zero.
- Money is stored as integer minor units plus a currency code.
- Provenance links are stored only child to parent. A pack's cards, a box's packs, and a container's contents are always derived by query, never stored as lists.
- Counts (cards in a container, copies of a printing, packs in a box) are always derived, never stored.
- All text is Unicode-safe (real data includes Greek names).

### 5.1 Catalog (shared, read-only for users)

**games**
- id, slug, name

**sets**
- id, game_id, code, name, release_date
- kind (main, starter, extra, premium, promo_group, other), used for grouping in UI
- external_refs (JSONB: official series IDs, Cardmarket expansion IDs per language)

**cards**
- id, game_id, name, rules_text
- attributes (JSONB, validated per game by a schema in the game module: e.g. One Piece type, attribute, power, counter, color, traits)

**printings**
- id, card_id, set_id
- collector_id, display_code (what users see)
- rarity (always on the printing, never inherited from the card)
- variant (per game: base, parallel, reprint, alt_art, overnumbered, artist_signature, etc.)
- finishes (the finishes this printing exists in: nonfoil, foil, or both). Finish is not a separate printing: Cardmarket lists a card and its foil as one product with separate foil prices, and the card, number, and art are identical.
- default_finish (the finish of the standard version of this printing). E.g. a Riftbound Common from a booster is nonfoil by default with an optional foil; a Riftbound Rare from a booster is foil by default. A precon printing of the same card can have a different default.

**Finish rules** (apply everywhere: browse, filters, stats, surplus, decks, pricing)
- A copy in its printing's default finish is the **standard version** of that card, whatever the finish is. A booster Rare is standard even though it is foil.
- A copy in a non-default finish is a **finish variant** (e.g. a foil Common).
- Filters offer: **All** (default), **Standard only**, and **Finish variants only**. A plain "foil" or "non-foil" filter is available as a secondary option for physical sorting, but no default view hides standard cards because of their finish.
- Finish badges are shown only on finish variants, not on standard cards that happen to be foil.
- Completion counts a card as owned in any finish; "finish variants owned" is tracked separately.
- Playsets and deck completeness count copies in any finish.
- Values use the price matching the copy's actual finish.
- language (on the printing, because marketplaces treat each language as a separate product with its own price)
- image (JSONB: image_key, checksum, source (e.g. official, cardmarket) for reference only), set by Studio
- source (official, supplementary, manual), for printings not present in official data
- external_refs (JSONB: official image filename keys, Cardmarket idProduct, TCGplayer IDs)
- attributes (JSONB for game-specific print details)

**product_types**
- id, game_id, set_id (nullable), name, image (same shape as printings)
- kind (pack, box, case, precon, double_pack, bundle, other)
- external_refs (JSONB: Cardmarket sealed product IDs)

**product_type_contents**
- id, product_type_id, content_type, quantity
- content_type is one of:
  - `product`: a nested product. References child_product_type_id.
  - `fixed_printing`: a guaranteed specific card. References printing_id.
  - `random_pack`: random cards following a pack profile. References pack_profile_id.
- One product type can mix all three.

Examples from real products:
- One Piece booster pack: 1 × random_pack (12 cards; 13 for OP01 to OP03; 10 for PRB01).
- One Piece booster box: 24 × booster pack (20 for PRB01).
- One Piece case: 12 × booster box (10 for PRB01).
- One Piece double pack: 2 × booster pack + 1 × DON!! pack (a small random pack with its own pool).
- Tournament, judge, winner, and pre-release packs: 1 × random_pack with 1 to 3 cards.
- Precon or promo collection: N × fixed_printing.
- Starter deck with a bonus pack: N × fixed_printing + 1 × booster pack.

**pack_profiles**
- id, product_type_id, version, card_count, notes

**pack_profile_pool**
- id, pack_profile_id
- a set (all its printings), a filtered subset (by rarity or variant), or explicit printings (for promo and DON!! packs)
- A profile can combine several pool entries (e.g. main set plus a promo insert).

**pack_profile_composition**
- id, pack_profile_id, group_label (e.g. "Commons", "Rare or better"), allowed rarities, variants, and finishes, expected_count
- Groups describe the normal pack, e.g. 7 C, 3 UC, 1 R, 1 R-or-better hit.
- Composition is guidance only. Any card from the pool can be entered regardless of groups.

A profile with a card count and no composition is **flat mode**: pick `card_count` cards from the pool. This is how the prototype works today, and it lets new products be supported before their composition is verified.

**pull_rate_references** (optional, per set)
- id, set_id, source (official, community, other), source_note
- per rarity or variant: stated odds (e.g. 1 in 24 packs) or expected count per box or case

### 5.2 Pricing

**price_points**
- id, printing_id or product_type_id (null until matched), marketplace_product_id, source (cardmarket, tcgplayer), date, currency
- values: low, avg, trend, avg1, avg7, avg30 (matching the Cardmarket price guide), with foil values (Cardmarket's foil columns); a copy's value uses the values matching its finish
- Imported daily by the price job (8.4).

**Currency.** MVP uses euros only. Every amount still stores a currency code, so multi-currency can be added later without data migration.

**Price basis.** Wherever a value is shown, it uses a price basis: trend, low, or 30-day average. The user sets a global default (trend). Individual screens (trade builder, want list) offer a quick toggle that does not change the default.

**Value snapshots.** Items store `value_snapshot` at acquisition: the full set of values (low, trend, avg30) from the most recent price point on that date. Any basis can then be applied later without losing history. Null if no price was available.

**Price matching rules**
- Each printing stores its marketplace product ID explicitly in `external_refs`.
- IDs are established once per set in Catalog Studio (8.3) and published with the catalog.
- Cardmarket's catalogue files carry no collector numbers, and every version of a card shares the same product name (e.g. six One Piece products all named "Roronoa Zoro (OP01-001)"; Riftbound alt arts, overnumbered, and artist signature printings share names with their base cards). Only cards with a single printing match automatically; all variants need mapping.
- Never match by sorting prices. The prototype assigned prices to variants in order of trend price, which breaks whenever an alt art is cheaper than the base or a new parallel is added.

### 5.3 User collection

**contacts**
- id, user_id, name, kind (local_store, online_store, organizer, person), notes
- Used as sellers on acquisitions, hosts of events, and partners in trades.

**acquisitions**
- id, user_id, date, type (purchase, trade, prize, gift, other)
- contact_id (nullable), event_id (nullable), trade_id (nullable), notes
- total_price, currency (zero for gifts and prizes)
- The total is allocated across the acquisition's top-level items (5.4). Items can also carry an explicit price, which takes precedence.

**products**
- id, user_id, acquisition_id (null for partial parents)
- kind (owned, lot, partial)
  - `owned`: a real product with a product_type_id.
  - `lot`: a user-defined group of cards bought together; name required, product_type_id null.
  - `partial`: a grouping record for a box or case the user does not fully own (6.2.3). No price, no value, never counted as owned.
- product_type_id (null for lots)
- name (required for lots, auto-generated for partials, optional override otherwise)
- parent_product_id (nullable: pack inside box inside case)
- position_in_parent (nullable: e.g. pack 7 of 24)
- status (sealed, opened, sold, traded, gifted; null for partials; partials use open or closed instead)
- price_paid, value_snapshot, currency
- container_id (nullable, for sealed products in storage)
- opened_at (nullable)
- session_status (nullable: in_progress, complete), for boxes, cases, and lots being logged over time
- expected_count (nullable, lots only)
- open_method (nullable: profile, flat, instant, manual)
- contents_complete (packs only: every card was logged)
- unknown_card_count (packs only, default 0: cards known to exist but not identified)
- off_profile (packs only: composition differed from the profile), with deviation details

**card_copies**
- id, user_id, printing_id
- ownership (owned, borrowed). Borrowed copies are physical cards belonging to someone else (6.11).
- source_product_id (nullable: pack, precon, or lot it came from)
- acquisition_id (nullable: set when not from a product, e.g. singles, gifts, trades, prize cards)
- container_id, container_position (nullable: page and pocket in a binder)
- finish (nonfoil, foil), limited to the printing's finishes; defaults to the printing's default_finish when logging
- condition (Cardmarket scale: MT, NM, EX, GD, LP, PL, PO)
- grading (nullable JSONB: company, grade, cert_number optional). Stored only; no dedicated grading screens in MVP.
- price_paid (cost basis), value_snapshot, currency
- status (owned, sold, traded, gifted, returned), disposed_at, disposal_price (nullable). `returned` applies only to borrowed copies given back.
- protected (boolean: never suggest as surplus or sell candidate, e.g. sentimental or a signed card)
- listed_price, listed_at (nullable, post-MVP: set when a copy is listed for sale, see 6.10.3)
- loan_id (nullable: the open loan this copy is part of)

**containers**
- id, user_id, name, type (binder, storage_box, deck_box, bulk, other)
- type_label (nullable free text, e.g. "OP Box" or a repurposed product box, shown instead of the generic type)
- parent_container_id (nullable: nesting)
- layout (nullable: pages and pockets per page for binders)
- is_master (boolean: master set container, see 6.9; one per game)
- is_trade (boolean: trading container; several allowed)
- is_default_trade (boolean: exactly one per user)
- Every new user gets two containers automatically: **Bulk** (the default destination for new cards everywhere) and **Trade Binder** (is_trade and is_default_trade). Both can be renamed.

**events**
- id, user_id, game_id, name, date, contact_id (nullable: host store or organizer)
- status (registered, completed)
- kind (store_tournament, pre_release, regional, championship, casual, other)
- format (nullable, per game: e.g. constructed, sealed, draft)
- entry_fee, currency
- deck_id (nullable), decklist_version_id (nullable: filled from the deck's target version, editable), deck_label (nullable free text fallback)
- placement, player_count, wins, losses, draws (all nullable)
- notes

**event_rounds** (optional detail)
- id, event_id, round_number, opponent_label (nullable), opponent_deck (nullable), result (win, loss, draw), notes

**event_rewards**
- id, event_id, reward_type (participation, prize), kind (collection_item, store_credit, other)
- For collection items (cards, products, lots): acquisition_id of the acquisition created for them (type prize, linked to the event).
- For store credit and other: amount, currency, description.

**decklists**
- id, user_id, game_id, name, notes, format (nullable)
- current_version_id

**decklist_versions**
- id, decklist_id, version_number, created_at, note (nullable, e.g. "cut 2 Nami for 2 Robin after regionals")
- source (manual, imported), import_text (nullable)
- Versions are immutable once saved. Editing happens in a draft; saving creates a new version.

**decklist_entries**
- id, decklist_version_id, card_id, quantity
- section (per game: e.g. One Piece leader, main; Riftbound legend, champion, main, runes, battlefields, sideboard)
- preferred_printing_id (nullable)
- Entries reference cards, not printings, since any printing of a card is legal.

**decks**
- id, user_id, decklist_id, container_id (the deck box), name (defaults to the decklist name)
- target_version_id (follows the current version by default; can be pinned to an older one)
- Contents are the card copies in the container; completeness is derived against the target version.

**deck_placeholders**
- id, deck_id, card_id, quantity, notes
- Proxies only. Borrowed cards are real borrowed copies (6.11), not placeholders.

**loans**
- id, user_id, direction (lent, borrowed), contact_id (nullable), partner_label (nullable)
- started_at, due_at (nullable), returned_at (nullable)
- status (open, partially_returned, returned), notes
- For lent loans, items are the user's own copies (card_copies.loan_id). For borrowed loans, items are borrowed copies created when the loan is logged.

**trades**
- id, user_id, date, contact_id (nullable), partner_label (nullable free text if no contact)
- status (draft, completed, cancelled)
- price_basis (the basis shown at completion)
- cash_given, cash_received, currency
- notes

**trade_items**
- id, trade_id, direction (out, in)
- out: card_copy_id or product_id from the user's collection
- in: printing_id (with finish and condition) or product_type_id; on completion a new copy or product is created
- value_snapshot (set at completion)

### 5.4 Cost basis rules

**Allocation**
- Price flows down from parent to children: acquisition to its items, case to boxes, box to packs, pack to cards, lot to cards, precon to cards.
- Default split is equal by count. User setting: weighted by value snapshot.
- Mixed products split by content (a double pack divides across its three packs).
- Allocation recalculates when contents change (cards added to a lot, prizes added to an event).
- Partial parents have no price and allocate nothing; each child keeps its own price.

**Trades**
- Cost given up = cost basis of outgoing items + cash given − cash received.
- Allocated across incoming items. If negative, incoming items get zero cost and the remainder counts as money returned.

**Events**
- The entry fee is allocated across all collection items received from the event, participation and prize alike. Participation packs pass their share down to their cards when cracked.
- Store credit takes no share. If an event gives only store credit, the fee stays as an unallocated event expense.

## 6. Key flows

### 6.1 Sessions and autosave
Applies to pack, box, and case cracking, lot entry, trade drafts, decklist drafts, and deck pull mode.
- Every action saves immediately to the local database. There is no unsaved state to lose.
- Any session can be left at any point and resumed later, on the same or another device after sync.
- Unfinished sessions appear in a **Continue** list on the home screen with progress (e.g. "Strike Shoebox: 1,204 cards logged", "OP10 box: 9 of 24 packs", "Trade with Nikos: draft").
- No timeouts, no requirement to finish in one sitting.
- Session defaults (destination container, condition, language) are set once at the start and editable per card.

### 6.2 Logging acquisitions

#### 6.2.1 Products
1. Add an acquisition: date, contact, total price, type.
2. Add one or more products. Nested children are created per `product_type_contents`, with price allocated down (5.4).
3. Sealed products can be stored in containers and keep their own value.

#### 6.2.2 Singles and gifts
Search the catalog, pick a printing, set condition, attach to an acquisition (purchase or gift). Prizes are logged through events (6.6), trades through the trade builder (6.5).

#### 6.2.3 Partial boxes and cases
Sometimes items are known to come from the same box or case without owning all of it: packs bought over the counter from a store's open box, a box split with friends, packs sold before the rest were opened, or boxes from the same case.

A **partial parent** (products.kind = partial):
- Groups children only. No price, no acquisition, no value, never counted as owned or in net position.
- Shows known children out of the total, e.g. "6 of 24 packs".
- Accepts children over time, across multiple acquisitions.
- Nests at every level: packs in a partial box, boxes (full or partial) in a partial case, and chains of both.
- Can be closed ("no more from this box") so it stops being suggested.

A fully owned box or case whose children were later sold stays a normal product; the sold children just have status sold.

**UX rules: grouping must never slow down logging.**
- Grouping is always optional. Items with no parent are fully valid.
- Logging several packs (or boxes) of the same product in one acquisition shows one toggle: "From the same box" (or "case"). Off by default.
- Logging packs or boxes of a set with an open partial parent from the same contact shows a one-tap suggestion, e.g. "Add to Playhouse Patras OP10 box (6 of 24)?". Dismissing takes one tap.
- Placing a partial box into a case is only suggested if a matching open partial case exists.
- Names are automatic (contact, set, date).
- Grouping after the fact: select packs or boxes in the products view, "Group as same box" or "Group as same case".

### 6.3 Opening products
Opening happens at logging time or later from the sealed collection. A product's contents dispatch by type, and one product can mix all three:
- **Nested products** become individually sealed children, opened now or kept sealed.
- **Fixed printings** use instant open (6.3.1).
- **Random packs** use cracking (6.3.2).

#### 6.3.1 Instant open (precons, promo collections)
1. Pick the product and a destination container.
2. Full contents list shown, all pre-checked.
3. Optional: untick missing cards, set condition on damaged ones, add an unexpected extra.
4. Confirm creates all copies with provenance, cost basis, and value snapshots.

#### 6.3.2 Crack a random pack (headline feature)
1. Pick a sealed pack, box, or case, and a destination container (default Bulk).
2. One screen: a grid of the pack's pool, sorted by rarity then collector number, with rarity tabs, search, and filters.
3. Tap a card to add it; tap again for a second copy. Any order.
4. A **pack tray** shows entered cards and progress against the expected composition, e.g. "C 7/7 · UC 3/3 · R 1/1 · Hit 0/1". When a group fills, the grid moves to the next unfilled tab; the user can go anywhere at any time.
5. Tapping a card in the tray removes it. Undo is always available.
6. Cards outside the pool (misprints, wrong-set inserts) are added through a "not in this pack" catalog search.
7. Confirm is enabled when the card count matches the profile.
   - If the composition differs from expected, a one-line notice appears (e.g. "2 hits, 0 R: unusual pack, confirm?"), dismissed with one tap. The pack is saved complete and flagged **off-profile** with the deviation.
   - If cards are missing (lost or not identified), the user can confirm with an unknown card count. The pack is marked not contents_complete, with a prompt to fix it later.
8. The next pack in a box opens immediately with an empty tray. Packs from a box are opened in order by default, recording position_in_parent; the user can turn this off if they opened out of order.
9. A running session summary for boxes and cases: packs done, hits so far, value vs cost.
10. Flat mode (no composition defined): same screen, tray shows only the card count.

#### 6.3.3 Fill a lot
Lots are the dominant acquisition type in real use: in the prototype data, about 86% of all copies came from bulk lots, some over 1,800 cards. Lot entry must be as fast as pack cracking.
1. Create a named lot with acquisition data (price can be zero for donations or gifts).
2. Pick a set to enter from; switch sets at any time, or search the whole catalog. The grid is sorted by collector number, with quantity steppers and tap to increment.
3. Filters (rarity, color, variant) narrow the grid while sorting a pile.
4. A running total shows cards logged, against an optional expected count.
5. Each entry batch can target a different container, since big lots get split across storage.
6. Lot logging is a session (6.1); the lot is marked complete when the user is done, and cost allocation recalculates as cards are added.
7. Scanning (post-MVP) should prioritize this flow.

### 6.4 Browse and organize

**Browse**
- **Stack mode**: grouped by printing with quantity. **Individual mode**: every copy with its provenance and value. Data is always per copy; stacking is only a view.
- Filter by game attributes, set, rarity, variant, finish (All, Standard, Finish variants; see finish rules in 5.1), container, source product, lot, deck, or acquisition.

**Organize**
- Move copies between containers individually or in bulk (select, select all, move).
- Binder view shows pages and pockets.
- Creating a container asks for a name and type; flags (master, trade) can be set on creation or later.

**Dispose**
- Mark copies or products as sold or gifted, with date and price, individually or in bulk. Trades go through 6.5.
- Disposed items stay in history for profit tracking.

### 6.5 Trades

#### Trade builder
1. Start a new trade, optionally choosing a contact. It opens on the **default trade container**. A picker switches to any other container (trade containers first, then all others, then whole collection).
2. **Your side**: tap copies or sealed products. Stack mode with quantity steppers for multiples. Adding a copy that is in a deck shows a warning ("This card is in your Red Zoro deck"). Borrowed copies and copies lent out cannot be added.
3. **Their side**: add cards from the catalog with condition, and sealed products.
4. **Cash**: optional amount on either side.
5. **Live balance bar**, always visible:
   - Value of each side including cash, and who is ahead by how much (e.g. "You're giving €4.20 more").
   - A price basis toggle: **Trend** or **Low**. Switching recalculates instantly.
   - Unpriced items flagged in the list and noted in the balance ("2 cards unpriced").
6. Item rows show individual values on the current basis.
7. Designed to be shown across a table: large numbers, clear sides, readable in landscape.
8. **Complete**: choose a destination for incoming items (default Bulk). Outgoing items are marked traded; incoming items are created through a trade acquisition with cost basis per 5.4.

#### Trade history
- Past trades with date, partner, value given vs received, gain or loss at the time.
- Per contact history.
- Provenance on copies: "traded away to X on date", "received in trade with X".

### 6.6 Events
Lightweight logging of play, so costs, rewards, and results live alongside the collection.
1. Create an event (status registered or completed): name, date, host, kind, format, entry fee.
2. Link the deck played (a tracked deck, with its version recorded) or a free text label.
3. Log results: placement and record, or round by round with optional opponent notes.
4. Log rewards, each tagged participation or prize:
   - Collection items (cards, products, lots) are created through a prize acquisition linked to the event, and receive a share of the entry fee (5.4). Products are opened later like any product.
   - Store credit and other rewards are recorded with amount and description.
5. Rewards added later trigger fee reallocation.

### 6.7 Decks
Deck management, not deckbuilding.

#### Decklists and versions
1. Create a decklist manually (add cards from the catalog with quantities, per section) or import from text in the game's common formats (e.g. "4xOP01-016"), with unmatched lines flagged.
2. Export to text.
3. Validation against the game module's rules (deck size, copy limits, required sections). Invalid lists can be saved but are clearly marked.
4. Optional preferred printing per entry.
5. Editing happens in a draft; saving creates a new version with an optional note.
6. **History** with change summaries ("+2 Robin, −2 Nami") and a **diff** between any two versions.
7. **Restore** creates a new version from an old one; history is never overwritten.
8. **Duplicate** branches a version into a separate decklist.

#### Physical decks
1. Build a deck from a decklist and choose or create a deck box container.
2. **Status** against the target version: complete or cards missing; per entry have, missing, and extras not in the list; proxies and borrowed copies counted as present but shown separately ("complete: 2 borrowed, 1 proxy"); preferred printings shown as "complete, 2 cards not in preferred version".
3. **Pull list** for missing cards, grouped by container, e.g. "Roronoa Zoro ×2: OP-08 Box (1), Trade Binder (1)". Prefers copies in storage over trade containers and other decks, and flags when the only copy is committed elsewhere.
4. **Pull mode** (a session): tick cards off as they are pulled; each tick moves the copy into the deck box.
5. **Want list**: cards not owned at all, with current cost to complete (price basis toggle) and text export.
6. **Version updates**: when the target version changes, an update checklist shows cards to pull in (with locations) and cards to take out (with a destination, default Bulk). Ticking moves the copies.
7. Decks pinned to an older version don't update and show "on v3, latest is v5".
8. **Deck value**: current value of copies in the deck box.
9. Removing cards from a deck moves them to a chosen container (default Bulk).

A decklist can have several physical decks or none. A physical copy is only ever in one container, so a card shared across decks shows as committed in the pull list, and the user decides whether to move it, borrow one, or use a proxy.

### 6.8 Stats

#### 6.8.1 Net position (headline stat)
First on the stats screen: "You've spent X on this game. Your collection is worth Y. You are up or down Z."

**Total spent**
- Purchases: products, singles, and lots, sealed or opened.
- Cash paid on top of trades.
- Event entry fees.
- Purchases paid with store credit are recorded at their full price (the credit itself counts as returned below).

**Total returned**
- Sales of copies and products.
- Cash received on top of trades, and negative trade cost remainders (5.4).
- Store credit received from events.

**Current value**
- Owned card copies and owned sealed products at current value on the default price basis. Copies currently lent out are included (still the user's).
- Borrowed copies, partial parents, disposed items, and placeholders are excluded.

**Net position = current value + total returned − total spent**, shown as a large number, green or red, with the percentage.

Rules:
- Gifts and zero-price items count as zero spent, and their value counts.
- Items with no known price are excluded from value, and the screen always shows coverage (e.g. "Value based on 5,420 of 5,835 cards").
- Items with a null value snapshot are excluded from gain-since-acquisition figures, never treated as zero.
- Breakdown by category: spent (packs, boxes, singles, lots, trades, events), value (cards, sealed), returned (sales, trades, store credit).
- Filters: game, and time range (all time, this year, custom). A range covers money spent and returned in it and the current value of items acquired in it.
- A chart of spent vs value over time is post-MVP (requires price history).

#### 6.8.2 Collection stats
- **Money detail**: average cost and value per card, biggest gains and losses by copy, best and worst products and lots by return.
- **Activity**: packs opened, total copies, unique printings.
- Borrowed copies are excluded from all collection stats and completion.
- **Completion per set**: unique cards owned out of total, per rarity, and variants owned. Rows color-coded complete, partial, none.
- **Playsets per set**: complete playsets and copies up to the limit, using the game module's deck rules (e.g. One Piece: 4 copies, leaders 1).

#### 6.8.3 Trade stats
- Number of trades, value given vs received, net gain or loss from trading.
- How trades aged: current value of cards received vs current value of cards given away.

#### 6.8.4 Event stats
- Entry fees vs value of rewards (current and at acquisition), plus store credit.
- Events played, win rate, average placement: per game, per store, per physical deck, per decklist, and per decklist version (e.g. "v3: 8-2, v4: 5-5").
- Cards won, with value.

#### 6.8.5 Pull stats

**Data quality tiers**
- **Packs**: any pack opened with profile or flat cracking and contents_complete. Lots, singles, and instant-open products are never included. Off-profile packs count fully; they are exactly the packs worth studying.
- **Complete boxes and cases**: every child is logged and complete. Completeness rolls up, so a case is complete only when all its boxes are. A partial parent that reaches all of its children counts as complete.
- **Partial groups**: known subsets of a box or case. They prove what was in that box (lower bounds) but cannot feed averages or distributions.
- Every stat shows its sample size (e.g. "based on 86 packs, 3 boxes"); small samples are labeled, not hidden.

**Hits**: each game module defines hit tiers (e.g. One Piece: SR, SEC, parallels, SP, manga). Stats default to these; users can pick any rarity or variant.

**Rates and distributions** (packs, complete boxes and cases)
- **Pull rates per set**: per pack and per box, e.g. "SEC: 1 in 26 packs, 0.92 per box".
- **Against reference odds**, where known.
- **Luck summary** per set, e.g. "1.3× the expected SECs in OP10" (requires reference odds or enough data).
- **Distribution** of hits per box and per case (histogram).
- **Position in box**, when pack order was recorded.
- **Subsets**: hits within partial groups (e.g. "these 6 packs from one box had 1 parallel").
- **Droughts**: packs since the last hit of each tier, and longest dry streaks.
- **Value per product**: average pulled value per pack and box vs average cost.
- **By source**: rates and value per contact, with a clear small-sample note.

**Records: what's possible** (all tiers)
- **Pack records** (exact): most hits in one pack, most of each tier, minimum guarantees (e.g. "every pack has at least 1 R or better").
- **Box and case records**: most of each tier confirmed in one box or case, e.g. "up to 3 alt arts in one box". Records from partial groups are lower bounds and say so ("at least 3, from 8 of 24 packs"). Minimums only come from complete groups.
- **Co-occurrence**: combinations confirmed together in one pack, box, or case (e.g. a SEC and an SP, two parallels of the same card). Partials count.
- **Special packs**: named patterns from the game module (e.g. god packs), detected automatically and listed with contents, date, and source, e.g. "1 god pack in 412 packs".
- **Off-profile packs**: grouped by deviation type (e.g. "hit replacing the R: 14 packs", "card from outside the pool: 1 pack"), to show what's possible beyond the profile and refine profiles over time.

**Community data** (post-MVP, opt-in): anonymized cracking data (set, product type, pack contents, position in box, date; no personal or contact information), aggregated alongside personal stats, with safeguards against bad data.

### 6.9 Master set gaps
A container per game can be marked as the master set container. When browsing other containers, copies of printings with no copy in the master container are highlighted as "first copy", so the user knows what to pull and file. (The prototype does this for Bulk against the main binder.)

### 6.10 Surplus and selling

#### 6.10.1 Surplus (MVP)
Surplus is derived from the collection, never stored:
- **Keep target** per card: the game module's playset limit by default (e.g. One Piece: 4, leaders 1; Riftbound: 3 for main deck cards, 1 for Legends and Battlefields, runes excluded). User settings: a different global keep count, per-card overrides, and "keep one of every printing" for variant collectors.
- A copy counts as surplus when the user owns more copies of that card than the keep target, after excluding copies that are in a deck box, in the master set container, lent out, borrowed, or protected.
- Which copies are surplus: the lowest condition and the least preferred printings first, so the best copies are kept.

**Surplus view**
- Grouped by set, with filters by rarity, container, and value.
- Each row: card, copies owned vs keep target, surplus count, current value, where the surplus copies are.
- Total surplus value at the top (e.g. "€312 in surplus across 1,480 cards").

**Actions**, on single or bulk selections:
- Move to a trade container (default Trade Binder).
- Mark as sold.
- Mark as protected (never suggested as surplus, e.g. sentimental or signed cards).

#### 6.10.2 Sell candidates (post-MVP)
A view ranking copies worth selling or trading, each with the reason shown:
- **High value surplus**: surplus copies above a value threshold.
- **Biggest gainers**: largest gain since acquisition, absolute or percentage.
- **Not playing it**: valuable copies in no deck and no master container.
- **Falling**: largest value drop over the last 30 days (requires price history).

#### 6.10.3 Listing and marketplace export (post-MVP)
Findings that shape this feature (September 2026):
- Cardmarket is not accepting new API applications, and existing API credentials may not be shared with third-party software that changes stock, so the app cannot list or sync stock directly.
- Cardmarket's website has no native CSV stock upload. Sellers use its bulk listing form, browser extensions that fill that form from a CSV, or paid third-party tools that import CSVs with Cardmarket product IDs.

Planned approach:
1. Select copies from anywhere; identical copies (printing, finish, condition, language) group into one row with a quantity.
2. Pricing from a price basis with adjustments (percentage, minimum price, rounding), editable per row.
3. Condition and language mapped to the marketplace's scale.
4. Output options:
   - **CSV with Cardmarket product IDs** for third-party listing tools.
   - **Listing sheet** ordered like Cardmarket's bulk listing form (by expansion, then name and version), to speed up manual entry.
   - Other marketplaces with real upload or API routes (e.g. CardTrader, TCGplayer), evaluated when this feature is picked up.
5. Copies without a marketplace product ID are listed separately with a prompt to resolve them.
6. Exported copies are marked **listed**, with a badge, warnings in trades and pull lists, bulk "mark as sold" and "unlist", and exclusion from surplus.

### 6.11 Lending and borrowing

#### Lending
1. Select copies (from anywhere, bulk allowed) and choose "Lend", a contact, and an optional due date.
2. Copies stay in the collection and in value, keep their container, and show a "lent to Nikos" badge.
3. Lent copies cannot be traded or counted toward deck completeness (a deck whose card is lent out shows it as missing, "lent to Nikos").
4. **Return**: mark the whole loan or individual copies as returned; copies clear their loan and can be moved back to a chosen container.

#### Borrowing
1. Choose "Borrow", a contact, and an optional due date, then add cards from the catalog (like the other side of a trade), usually straight into a deck box.
2. Borrowed copies are created with ownership borrowed. They occupy a real container position and count toward deck completeness, but are excluded from value, net position, completion, surplus, trades, and export.
3. **Return**: mark returned; the copies get status returned and leave the containers.

#### Loans view
- Open loans in both directions, grouped by contact, with counts and value (lent value shown so the user knows what's at stake).
- Due or overdue loans appear in the home screen **Continue** area as reminders.
- Loan history per contact.

## 7. Game modules

Each game has a module in code containing:
- Attribute schemas for cards and printings.
- Rarity and variant definitions.
- Filter definitions (e.g. One Piece colors and rarities).
- Hit tiers and special pack patterns for pull stats.
- Deck rules (deck size, copy limits, required sections) for validation, playset stats, and default keep targets for surplus.
- Marketplace mappings: Cardmarket game ID, condition scales, and language codes.
- Decklist text import and export formats.
- Printing identification (7.1).
- Source adapters used by Catalog Studio: how to fetch and normalize this game's catalog, images, and marketplace catalogue (8.2).
- Seed data for sets, product types, pack profiles, and reference pull rates. These are stored as data, never as hardcoded lists in app code.
- Display components for card details.

Core tables never gain game-specific columns. Adding a game means a new module and its catalog import, with no migrations to core tables.

### 7.1 Printing identification by game

**Riftbound catalog source: the official Riot API**
- Riot's developer portal offers `riftbound-content-v1`, a single endpoint returning all Riftbound content: game, content version, last updated timestamp, and sets, each with its cards.
- Per card: ID, name, set, rarity, type, faction (domain), description (rules text), flavor text, keywords, tags, stats (cost, energy, might, power), collector number, and art (artist, full image URL, thumbnail URL).
- Studio calls it with a Riot API key. A development key is granted automatically on sign-in to the developer portal and expires every 24 hours, which suits manual Studio runs (reset the key before a run).
- The endpoint's `version` and `lastUpdated` fields let Studio detect whether anything changed since the last release.
- Only English was available during the API's beta; other languages to be checked when needed.
- **[VERIFY IN MILESTONE 2]** How variants (alt art `a`, artist signature `*`, overnumbered) appear in the API: `collectorNumber` is an integer, so variant suffixes are expected in the card ID. Confirm against real data before finalizing the Riftbound variant rules below.

**Riot's Riftbound policy** (developer portal), relevant rules:
- Card galleries and deck managers are explicitly encouraged; tools must not simulate or replicate gameplay.
- Apps may only use Riftbound assets (including cards) provided by the Riot API, no external or unofficial materials. For Riftbound this means **card data and images come only from the Riot API**, never from Cardmarket images or community sites. Cardmarket is used only for prices and product IDs.
- Community databases (e.g. Riftcodex, which exposes Cardmarket and TCGplayer set IDs) may be used in Studio only as matching hints, never as a source of card data or images.
- Launch requirements (not needed for personal use): register the product on the developer portal and get it approved, have a free tier, no gambling, don't imply Riot endorsement, include Riot's "Legal Jibber Jabber" statement in the app, and review the policy's limits on "metagame-defining data" (e.g. deck play rates) before any community stats feature.

**Terminology: two different "signatures"** (never mix them up in code, data, or UI)
- **Artist signature** (`artist_signature` variant): a Showcase printing physically signed by the artist, marked with `*`. A collector treatment of a printing.
- **Signature card** (`is_signature_card` card attribute): a gameplay card type (Signature Spell, Signature Unit, Signature Gear) tied to a champion tag, e.g. Tibbers for Annie. A property of the card, subject to deck rules.
- A Signature card can itself have an artist signature printing; the two are independent.

**Riftbound identifiers** printed on the card (format `number/setSize`):
- Base cards use the set collector number (e.g. `007/298`).
- Overnumbered cards use numbers above the set size, without an asterisk (e.g. `238/219`).
- Artist signature printings add `*` (e.g. `299*/298`).
- Alt arts add `a` (e.g. `007a/298`).
- Other formats exist (e.g. `SP3/006`, `R01b`); Studio flags unrecognized patterns for review.
- Variant is derived from the collector ID on import.

**Riftbound rarities and finishes**
- Rarities: Common, Uncommon, Rare, Epic, Showcase, Promo, and Ultimate (introduced in Unleashed). Riftbound has no "Legendary" tier.
- Showcase is one rarity covering three treatments: alt art (`a`), overnumbered, and artist signature (`*`).
- Commons and Uncommons exist non-foil, and most also have a foil version.
- Rare and above from boosters are foil only. Precon decks (e.g. Proving Grounds, champion decks) are printed non-foil, so their Rares and Epics are the only non-foil ones.
- Runes come in the token slot; foil runes and rare alt-art runes also exist.

**Riftbound booster profile** (same structure for Origins, Spiritforged, and Unleashed; Vendetta to be confirmed from product listings in Studio)
- 14 cards per pack, officially: 7 Commons, 3 Uncommons, 2 foil Rares or better, 1 foil of any rarity, 1 token slot.
- Composition groups:
  - Commons: 7, Common, non-foil.
  - Uncommons: 3, Uncommon, non-foil.
  - Foil slot: 1, any rarity, foil (usually a Common or Uncommon; can upgrade to Rare, Epic, or a premium treatment).
  - Rare or better: 2, Rare, Epic, Showcase, or Ultimate, foil.
  - Token slot: 1, token or rune (non-foil, foil, or alt-art rune).
- Booster box: 24 packs. Case: 6 boxes according to retailer listings. **[VERIFY IN STUDIO]** case size per set.
- Off-profile examples worth detecting: two Epics in one pack, a premium treatment in the foil slot, an alt-art rune.

**Riftbound reference pull rates** (stored as pull_rate_references, with sources)
- Epics: more than 6 per box on average; alt arts: more than 2 per box (Riot, official product description, Origins and Spiritforged).
- Overnumbered: about 1 per 1 to 3 boxes; artist signatures: about 1 in 10 overnumbered cards (Origins, shared at Riot's pre-launch summit, reported by press).
- Unleashed and later sets: official odds for Ultimate and premium treatments not yet published; left empty until available.

**Riftbound hit tiers** (defaults for pull stats): Epic, Showcase alt art, Showcase overnumbered, Showcase artist signature, Ultimate, alt-art rune. Foil Commons and Uncommons are tracked but not counted as hits.

**Riftbound deck rules** (Core Rules, rule 103; tournament rules for constructed)
- **Legend**: exactly 1 Champion Legend. Its two domains form the deck's Domain Identity.
- **Chosen Champion**: 1 Champion Unit whose champion tag matches the Legend (e.g. a Jinx Legend allows Jinx, Rebel or Jinx, Demolitionist). It starts in the Champion Zone but is one of the Main Deck cards and counts toward that card's copy limit. Signature units are not Champion Units and can never be the Chosen Champion.
- **Main Deck**: at least 40 cards (units, gear, spells), exactly 40 for sanctioned constructed events.
- **Copy limit**: up to 3 copies per card name, where the name includes the subtitle. "Darius, Trifarian" and "Darius, Executioner" are different names, so a Darius deck can run 3 of each alongside its Chosen Champion copy.
- **Other champion units**: no special restriction. Any champion unit, including other champions, can be in the Main Deck if it fits the Domain Identity and copy limit.
- **Signature cards** (the gameplay card type, not artist signatures): at most 3 in total across all names, and all must match the Legend's champion tag.
- **Rune Deck**: exactly 12 runes, within the Domain Identity.
- **Battlefields**: exactly 3, each with a unique name.
- **Sideboard** (tournaments): up to 10 cards, swapped one for one; runes, Legend, and battlefields are locked once registered.
- **Domain Identity**: every Main Deck and Rune card must fit the Legend's domains; a multi-domain card needs all its domains in the identity.
- **Formats** for validation: Constructed (exactly 40), Casual (40 or more). Limited formats (Sealed, Draft) have different rules (e.g. Draft Main Deck of at least 20, Sealed Signature cards not required to match the champion); decklists in those formats get relaxed validation only.
- Rules and errata change over time, so copy limits, deck sizes, and any banned or restricted list live as data in the game module seed, updated through Studio releases, not in code.
- **[VERIFY IN MILESTONE 2]** Which Riot API fields carry domains, card type, champion tag, champion unit flag, and Signature card flag needed for validation (expected in `faction`, `type`, and `tags`).

**Riftbound special pack patterns**: pack with 2 or more Epic-or-better cards; pack with a premium treatment in the foil slot; pack with 2 or more premium treatments.

**One Piece TCG** has no printed distinction between base cards and variants; they share the card number.
- Official image filenames distinguish versions: `_p1`, `_p2` for parallels and `_r1` for reprints.
- Some versions are missing from official data and come from a supplementary source (prototype: `_v` versions).
- The official "Promo" series must be split into internal sets by distribution. There are dozens (tournament, winner, judge, and pre-release packs, regional participation and finalist packs, Treasure Cups, Pirates Party, event packs, anniversary sets, premium collections, convention exclusives), and new ones appear constantly, so promo sets must be addable as data without app releases.
- Imports match existing printings using multiple signals (card number, set, image, external IDs) and flag conflicts for review instead of silently reassigning.
- The app shows a readable variant label (e.g. "Parallel 1", or the product it came from).

## 8. Data ingestion

Shared data is split by how often it changes and how much judgment it needs:
- **Catalog and image references** (cards, printings, sets, product types, pack profiles, marketplace mappings, image source URLs) change mostly at set releases and need human confirmation for variants. They are prepared in **Catalog Studio**, an internal companion app, and published to the main app as versioned **catalog releases**.
- **Prices** change daily and need no judgment once mappings exist. A single scheduled **price job** in the main app's backend downloads them automatically.

User devices never fetch source data directly. They sync the published catalog and prices from the main app's backend.

### 8.1 Cadence
- **Catalog Studio runs**: at every set release, when promos or new products appear, and for corrections. Expected every few weeks, not daily.
- **Price job**: daily, fully automatic.
- **Prompts to run Studio**: the price job checks Cardmarket's product catalogue each day and notifies the team when there are Cardmarket products with no mapping (e.g. "Riftbound: 214 new unmapped products, 1 new expansion"), so a Studio run happens when it is actually needed.

### 8.2 Catalog Studio

An internal tool for the team, never shipped to users. It runs locally, keeps its own working files, and shares `packages/core` and the game modules with the main app so schemas and identification logic are identical.

**Workflow per run**
1. **Download**: pull source data for a game: its catalog source (Riftbound: Riot API; One Piece: official site plus supplementary data), Cardmarket product catalogue files (singles and non-singles), and images from the sources the game allows. Raw downloads are kept with dates, so a run can be re-parsed later.
2. **Normalize**: game module adapters turn raw data into catalog records.
3. **Diff**: compare against the last published catalog release: new, changed, and removed records.
4. **Confirm**: work through confirmation queues (below). Clear cases are pre-accepted; anything uncertain needs a human decision.
5. **Validate**: automated checks before publishing (every printing has an image, no duplicate identities, owned printings not removed, pack profiles reference existing printings, deck rules and composition groups are valid).
6. **Publish**: produce a catalog release and import it into the main app (8.5).

**Confirmation queues**
- **Identity**: new printings that could match more than one existing record, changed collector IDs, possible renumbering, and removals.
- **Marketplace mapping**: per-set screen pairing printings with Cardmarket products (8.3).
- **Images**: where a game allows several sources, choose per printing or per set, preview side by side, and flag missing or low quality images. Riftbound has a single allowed source (Riot API art), so this queue only flags problems.
- **Sets and promos**: map Cardmarket expansions and official series to internal sets; create new promo groups.

**Manual editing**
- Add or correct printings not in any source (manual and supplementary printings).
- Create and edit sets, product types and their contents, pack profiles, special pack patterns, and reference pull rates.

**Progress view** per game and set, e.g. "Origins: 340 printings, 312 mapped to Cardmarket, 28 need review, 2 missing images".

### 8.3 Marketplace mapping (in Studio)

**Cardmarket catalogue**
- Cardmarket publishes its product catalogue (updated on new releases) and price guide (updated daily) as public JSON downloads for all games, replacing its old API endpoints (announced June 2024).
- Files are served from a public CDN, fetchable without login or API key. The website itself blocks automated clients and is never scraped.
  - `https://downloads.s3.cardmarket.com/productCatalog/productList/products_singles_{gameId}.json`
  - `https://downloads.s3.cardmarket.com/productCatalog/productList/products_nonsingles_{gameId}.json`
  - `https://downloads.s3.cardmarket.com/productCatalog/priceGuide/price_guide_{gameId}.json`
- Game IDs: One Piece 18 (matches the prototype files), Riftbound 22. Stored in each game module; URLs are configuration.
- The files contain no collector numbers and no public name for expansion IDs, and every version of a card shares the same product name.

**Mapping steps**
1. **Expansions**: Cardmarket expansion IDs are suggested for internal sets by matching product names against the catalog; the user confirms. Promo and extras expansions are mapped manually.
2. **Single printings**: products whose name matches exactly one printing in the set are pre-accepted.
3. **Name collisions** (base plus alt art, overnumbered, artist signature, parallels): shown side by side, all printings of the card with images next to all Cardmarket products with that name, with hints (idMetacard, dateAdded order, current prices). The user confirms each pairing. Price ranking is only ever a hint.
4. **Sealed products**: non-singles mapped to product types the same way.
5. Confirmed mappings are stored in the printing's and product type's `external_refs` and published with the release.

### 8.4 Price job (main app backend)
- One scheduled function, daily, early morning CET after Cardmarket's price guide refresh.
- Downloads the price guide for each supported game and stores the raw file in storage.
- Appends `price_points` for that date, keyed by Cardmarket product ID; existing dates are never overwritten.
- Prices for products not yet mapped are stored against the product ID and become visible once a catalog release maps them, so no history is lost.
- Also downloads the product catalogue files and compares them with published mappings, producing the unmapped products notification (8.1). It never changes the catalog itself.
- Failures retry, and repeated failures notify the team. Clients show the price date and a stale data notice if the latest prices are more than 2 days old.
- **Launch requirement**: before any public release, obtain written confirmation from Cardmarket that the app may use and display price guide data. Another Riftbound developer has reported support confirming the files can be used freely, but this must be confirmed for this app.

### 8.5 Catalog releases and import

**Release bundle** produced by Studio:
- A manifest: game, release version, previous version, created at, notes, checksums.
- Changed records only (create, update, retire) for cards, printings, sets, product types, contents, pack profiles, reference pull rates, and special pack patterns, including marketplace mappings in `external_refs`.
- Image keys and checksums for new or changed images. Image files ship separately as image packs (8.6).

**Import into the main app**
- Studio uploads the bundle with team credentials; the backend applies it in one transaction. A bundle file can also be imported manually as a fallback.
- Imports are idempotent (the same release applied twice changes nothing) and rejected if the previous version doesn't match the current one.
- Printings referenced by any user copy are never deleted; releases can only retire them.
- Each import bumps the catalog version; clients download only records changed since their version. Printings whose image is not yet installed show a placeholder until the matching image pack is imported.

**catalog_releases** (main app)
- id, game_id, version, previous_version, published_at, notes, checksum, record counts

**price_job_runs** (main app)
- id, game_id, run_date, status (success, failed), prices_stored, unmapped_products, error_log

### 8.6 Images
There is no image hosting and devices never download from live sources. Studio downloads images once, and they reach devices as **image packs**.

**In Studio**
- Downloads candidate images from every configured source into its local working files.
- The user picks the source per printing or per set.
- Studio converts chosen images to compressed WebP in two sizes (thumbnail for grids, full for detail views) and assigns each an `image_key` and checksum.

**Image packs**
- One pack per game and set (e.g. `riftbound-origins-v3.tcgpack`), a zip containing a manifest and the image files.
- Manifest: game, set, catalog release version it was built for, and a list of image_key, checksum, and file paths.
- Studio also produces **delta packs** containing only images added or changed since the previous pack for that set.
- Studio writes packs to a local output folder. How they reach devices is up to the user (below).

**Delivery** (two routes, both supported)
1. **Import (default)**: pick a pack inside the app (Settings > Images > Import pack). On web, a file picker or drag and drop; on mobile (later), also the phone's files, a personal cloud drive, or the share sheet. Works for every new set without rebuilding. Each browser or device imports its own packs.
2. **Bundled at build time (optional)**: packs placed in the app's assets are installed on first launch. On web this means the images are served with the app's files, so only use it for local development or a private deployment.

**In the app**
- Import verifies checksums, copies images into local storage, and updates a local image manifest.
- On web, images live in browser storage (Origin Private File System or IndexedDB). The app requests persistent storage so the browser doesn't clear it, and shows a warning if persistence is denied or quota is low. Re-importing the same pack changes nothing; a delta pack replaces only its images.
- The app shows which sets have images installed, which are missing or outdated compared with the current catalog release (e.g. "Origins: 12 images missing, import the latest pack"), and storage used per set, with an option to remove images.
- Missing images show a placeholder with the card's name, set, and collector ID, so the app stays fully usable without packs.

**Launch requirement**: confirm each image source's terms before any public release, since images would then be redistributed to other users.

User-submitted catalog corrections (e.g. "this card is missing") are post-MVP; they would be collected in the main app and reviewed in Studio.

## 9. Card scanning (post-MVP, design now)
- Scanning is an input method into existing flows, not a separate feature.
- Designed for the mobile apps, after milestone 14.
- In cracking, recognition only matches the pack's pool, weighting rarity groups not yet filled.
- Lot entry is the highest-value target for scanning.
- Riftbound collector IDs (including `*` and `a` suffixes) can be read to confirm the exact variant.
- Outside sessions, match against the full catalog with a confirm step.
- **[OPEN]** Approach: on-device image embedding match, OCR of collector ID, or hybrid.

## 10. Migration from the prototype (post-MVP)

Import of existing One Piece data once the One Piece module exists:
- `card_collection.json` → card_copies. Current `card_id` keys are card number plus optional version suffix (`OP07-006`, `OP08-030_p1`, `_r1`, `_v1`), mapped to printings via `newdb.json`. Older data used a series ID suffix (`OP08-030_p1_569108`), which the importer also accepts.
- `parent_id` prefixes identify the source: `Pack-`, `Lot-`, `Single-`, `DoublePack-`.
- `sealed_collection.json` and `opened_collection.json` → products (kind owned) with status sealed or opened, parent links from `parent_UID`.
- `lots.json` → products with kind lot.
- `containers.json` → containers, with `container_type` imported as `type_label`.
- Seller names (`acquired_from`) → contacts, deduplicated by name.
- The catalog comes from Catalog Studio, not the prototype files. `newdb.json`, `AdditionalCards.json`, and the prototype's Cardmarket files (the same public downloads) are used to validate the pipeline's One Piece output and to seed supplementary printings missing from official data.
- Cost basis from `price_acquired`. `value_on_date` was never populated (logged before a pricing system existed), so value snapshots import as null and can be backfilled if historical prices are found.
- Imported packs have no box links, so they feed pack-level pull stats only.
- `tournament_id` is empty everywhere; prize lots (e.g. "KG Decclesia Xmas Winnings") can be linked to events manually after import.
- Test and placeholder values (e.g. dates like "asd", sellers like "222") are flagged for review, not imported silently.

## 11. Milestones

1. **Foundation**: PowerSync spike on Expo web (3.1), then Expo project running on web with phone-sized layouts, platform adapters (see Conventions), Supabase schema, auth, local SQLite, game module structure, default containers.
2. **Catalog Studio and catalog**: Studio download, normalize, diff, confirmation queues, image selection, validation, and publish for Riftbound; catalog release import in the backend; catalog sync to clients; image packs from Studio with import and optional bundling; browse, search, filters.
3. **Collection basics**: acquisitions, contacts, singles, copies, containers, browse modes, bulk move, sessions framework.
4. **Products**: all product kinds, nesting, partial parents, price allocation, instant open, lots.
5. **Pack cracking**: flat mode first, then pack profiles with composition guidance, box and case cracking.
6. **Sync**: offline-first sync across devices.
7. **Pricing**: Cardmarket mapping screen in Studio, daily price job with unmapped product notifications, stale price notice, value snapshots, price basis, net position.
8. **Trades**: trade builder with live balance, completion, history.
9. **Decks**: decklists with versions, import and export, physical decks, pull lists, want lists.
10. **Lending and borrowing**: loans, borrowed copies, deck integration, reminders.
11. **Surplus**: keep targets, surplus view, protected copies.
12. **Events**: event logging, rewards, fee allocation.
13. **Stats**: collection, trade, event, and pull stats; master set gaps.
14. **Mobile apps**: iOS and Android builds, native implementations of platform adapters (storage, files, image packs, share sheet import), touch and performance testing on real devices, offline behavior on mobile.
15. **One Piece module** and prototype data import.
16. **Selling**: sell candidates, listing, marketplace export.
17. **Scanning** (mobile).

Each milestone ends in a working, testable app state.

## 12. Conventions (summarized for day-to-day use in CLAUDE.md)

- TypeScript strict mode, no `any`.
- Domain logic (allocation, composition checks, stats, provenance, import matching) lives in pure functions with unit tests, separate from UI.
- Schema changes only via Supabase migrations.
- Catalog Studio and the price job never write to user collection tables. Catalog changes reach the main app only through catalog releases.
- Game-specific code only inside its game module. No hardcoded set, product, or rarity lists in app code.
- Money handled as integers, never floats.
- IDs always generated, never derived from counts or external data.
- Unknown values are null, never zero.
- Platform-specific code (local database, file picking, image storage, share and import) lives behind adapter interfaces in `packages/core`, with a web implementation first and native implementations added in milestone 14. Screens never call platform APIs directly.
- No web-only libraries or DOM access in app screens; use React Native components so screens carry over to mobile unchanged.
- Layouts are tested at phone widths in the browser, not just desktop widths.
- Done means (milestones 1 to 13): runs in current Chrome and Safari, tests pass, no lint errors. From milestone 14: also builds and runs on iOS and Android.

## 13. Open questions

- **[LAUNCH]** Email Cardmarket support for written confirmation that the app may use and display price guide and product catalogue data.
- **[LAUNCH]** Riot: product registration and approval, Legal Jibber Jabber statement, policy review.
- **[LAUNCH]** Terms of use for each other catalog and image source (e.g. One Piece), including attribution requirements.

### Decided
- Data layer: PowerSync with Supabase, local SQLite via PowerSync on web and mobile, Zustand for UI state (3.1). Validated by a spike at the start of milestone 1.
- Riftbound pack profile, rarities, finishes, reference pull rates, hit tiers, and special pack patterns (7.1). Case size to verify in Studio.
- Finish (foil or non-foil) is stored on the copy, not as a separate printing. Each printing has a default finish, and standard cards are never hidden by finish filters.
- Riftbound deck rules and keep targets (7.1), stored as updatable data.
- Riftbound catalog and images: official Riot API (`riftbound-content-v1`), with a development key for Studio runs.
- Currency: euros only for MVP, currency codes stored everywhere.
- Condition scale: Cardmarket's. TCGplayer mapping added with selling.
- Catalog Studio: local web app.
- Web hosting: local development server for now.
- Price sources: Cardmarket only for MVP; TCGplayer postponed.
- Grading: minimal stored shape, no screens.

### Postponed (not blocking MVP)
- Scanning approach (mobile phase).
- Selling: export route for Cardmarket and other marketplaces.
- Community pull data: storage, anonymization, moderation, timing.
