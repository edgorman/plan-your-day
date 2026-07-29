# User Flow — planyour.day

This describes the intended end-to-end flow for a user of planyour.day. It's the narrative counterpart to the mechanics defined in `CLAUDE.md` (Section 4: Client-Side State & Swipe List Strategy, Section 5.3: private events) — read those for the technical contract behind each step.

## 1. Onboarding (first run only)

1. Sign in (Firebase Auth).
2. Set a primary location, with optional secondary locations for travel/exploration (e.g. an upcoming trip).
3. Pick interest categories from the Category list (`CLAUDE.md` Section 5.2) — seeds what shows up first, not a hard filter.

## 2. Pick a date

- **Free tier:** today only (1-day browsing limit).
- **Paid tier:** any date, including multi-day/90-day/1-year planning.

## 3. Browse the candidate pool

One request fetches the full pool of events + evergreen POIs for the selected date and location range. From there the user can browse via either view, freely switching between them without re-fetching:

- **Map View:** interactive Leaflet map with a 1-day time scrubber; pins for transient Events and evergreen POIs, badged distinctly.
- **Schedule View:** a TikTok-style swipeable card deck over the same pool, shown one card at a time in the order the backend returned it.

## 4. Swipe (Schedule View)

- **Right (save):** the card is added to the **current list** — the plan being built for this date.
- **Left (skip):** the card is added to a single **generic catch-all list**, shared across all skipped cards, in case the user wants to revisit it later.

No weighting or re-ranking happens — the deck order never changes based on swipes.

## 5. Add a private event (optional, any time)

Outside the swipe deck entirely: the user fills in a title, a time, and an address (autocompleted via the geocoding proxy). The result is inserted directly into the current list, badged as private — it has no source URL and is never shown to other users.

## 6. Autosave

The current list and the catch-all list sync to the backend automatically on a fixed 3-second interval whenever there's something unsynced. There is no explicit "save" step for the user to remember.

## 7. Revisit or export

- Reopen any previously-planned date to review or keep editing its current list.
- **Paid tier only:** export the current list as an `.ics` file for Google/Apple Calendar.
