# CLAUDE.md — planyour.day Project Instruction Manual

This file is the operating manual for Claude Code sessions working on **planyour.day**. Read it before making changes. It defines the architecture, the rules that keep the project cheap to run, the repository layout, the backend/frontend conventions, the core data models, and the roadmap for future sessions.

> **App summary:** planyour.day is an AI-powered spatial-temporal social planner that helps users discover and organize activities based on location, interests, and time. Core interaction loop: pick a date → browse a map or a swipeable card feed of events/POIs (Points of Interest — evergreen places like restaurants, venues, and parks, as distinct from one-off transient events) → save what you like → export or revisit your plan. Users can also manually add their own **private events** (e.g. a relative's wedding) into the same day's plan — see Section 5.3.

---

## 1. Tech Stack Rationale & Cost-Optimization Rules

The single non-negotiable constraint on this project: **it must cost ~$0/month at zero traffic and scale gracefully without re-architecture.** Every technology choice below exists to serve that constraint. Do not introduce a component that bills at idle (no always-on VMs, no provisioned-capacity databases, no paid map tile providers) without discussing it first.

### 1.1 Frontend — Vite + React on Firebase Hosting

- **Vite + React (TypeScript)** produces a static SPA bundle. There is no SSR server to keep warm and no per-request compute cost.
- **Firebase Hosting** serves the built assets from Google's global CDN. Static hosting on Firebase's free tier covers typical early-stage traffic entirely — **$0 static CDN delivery**. There is no server process, so there is nothing to scale or cold-start on the frontend.
- **Tailwind CSS** for styling — utility-first, no runtime CSS-in-JS cost, tiny production bundle via purge/JIT.
- **Shadcn UI** for component primitives — copied into the repo (not an npm dependency that bloats bundle size or requires a component-library server), built on Radix + Tailwind.
- **Lucide Icons** — tree-shakeable icon set, no icon font, no external requests.
- **React-Swipeable** — powers the TikTok-style card deck gesture handling client-side only.
- **React-Leaflet + OpenStreetMap tiles** — deliberately **not Mapbox/Google Maps**. OSM tiles have no API key and no per-load billing, which matters because map views are the app's primary UI surface and would be the single largest variable cost under a paid tile provider. If tile load volume ever requires a dedicated tile CDN, evaluate a pay-as-you-go OSM tile proxy before touching Mapbox.

### 1.2 Backend — Go on Cloud Run + Firestore

- **Go (1.22+)** for the REST API: small static binaries, low memory footprint, fast startup — all of which directly minimize Cloud Run cold-start latency and per-request CPU billing.
- **GCP Cloud Run**, configured to **scale to zero**. There is no traffic → no running container → no bill. Go's near-instant binary startup keeps cold starts around **~50ms**, so scale-to-zero doesn't translate into a bad user experience the way it would with a JVM or heavyweight Node server.
- **Firestore (Native Mode)** as the only datastore. Firestore's free tier + pay-per-operation pricing means an idle database costs nothing, unlike a provisioned Cloud SQL instance that bills whether or not it's queried. Native Mode (not Datastore Mode) is required for real-time listeners and the richer query/index model, which the map + schedule views will eventually want.
- No Redis, no message queue, no separate cache layer at this stage. If a caching need emerges (e.g., expensive geocoding lookups), prefer a Firestore-backed cache document/collection with a TTL field over standing up a new managed service — it avoids adding another always-billed component.
- **Nominatim (OpenStreetMap) geocoding** for the private-event address lookup (Section 5.3) — same no-paid-key philosophy as the map tiles. Calls go through a backend proxy endpoint, never directly from the client, to respect Nominatim's usage policy (required `User-Agent`, rate limits); repeated lookups of the same address are a good candidate for the Firestore-backed TTL cache above.

### 1.3 Infra & CI/CD

- **Terraform** is the only way GCP resources are created or modified. Nobody click-ops in the GCP console; state must reflect reality.
- **GitHub Actions** builds/deploys the frontend to Firebase Hosting and the backend container to Cloud Run. Keep workflows minimal — build, test, deploy — resist adding infra that isn't in Terraform.
- **Environments:** merges to the default branch (`main`) deploy to **production**. Merges to `develop` deploy to the **developer environment** — a separate Firebase Hosting site / Cloud Run service / Firestore database from production, provisioned by the same Terraform config with a different `environment` variable. Unless a step in this file names an environment explicitly, assume it targets the developer environment.
- **Makefile** is the single entry point for local dev so contributors (and Claude Code sessions) never need to remember multi-step tooling incantations.

### 1.4 Rules of thumb for future changes

1. **Would this component bill while idle?** If yes, justify it explicitly or find a scale-to-zero alternative.
2. **Does this need a server at all?** Prefer static/client-side solutions (like the swipe-to-list sorting in Section 4) over adding backend endpoints.
3. **Does this require a paid API key?** Avoid unless there's no free/open alternative (OSM over Mapbox, etc.).
4. **Does this fit in Firestore?** Prefer modeling new data in Firestore over introducing a second datastore.
5. **Where does this logic belong?** Business logic must never leak into `service/handlers` or `repository` code — both stay thin. Prefer a method on the relevant `entities` type over a freestanding `usecase`/util function; reach for a plain `usecase` function only when the logic doesn't belong to a single entity (e.g. it spans several, or needs a repository).

---

## 2. Repository Directory Tree & File Purpose

```
planyour.day/
├── .github/
│   └── workflows/            # CI/CD: lint/test/build on PR; deploy frontend → Firebase
│                              # Hosting and backend → Cloud Run. Merges to the default
│                              # branch (main) deploy to production; merges to `develop`
│                              # deploy to the developer environment. Also houses
│                              # Terraform plan/apply workflows for infra/.
├── docs/                     # Human-facing documentation, not code:
│   ├── architecture.md       #   - system diagram, data flow, scale-to-zero rationale
│   ├── api-spec.md           #   - REST endpoint contracts (request/response shapes)
│   └── user-flow.md          #   - the intended end-to-end user flow (Section 4/5.3 detail
│                              #     lives here in narrative form: onboarding → browse →
│                              #     swipe → add a private event → sync → revisit/export)
├── infra/                    # Terraform for all GCP resources, one file per resource
│   │                          # area rather than one monolithic file:
│   ├── cloud_run.tf            #   - Cloud Run service (scale to zero)
│   ├── firestore.tf            #   - Firestore database (Native Mode)
│   ├── artifact_registry.tf    #   - Artifact Registry repo for backend images
│   ├── iam.tf                  #   - service accounts + least-privilege bindings
│   ├── variables.tf           #   - project_id, region, environment, image tag, etc.
│   └── outputs.tf              #   - Cloud Run URL, Firestore name, service account emails
│                                #     (consumed by the deploy workflows in .github/)
├── services/
│   ├── frontend/              # Vite + React (TypeScript) SPA
│   │   ├── src/
│   │   │   ├── components/    #   - shared UI (Shadcn-based) + feature components
│   │   │   ├── views/          #   - MapView, ScheduleView (see Section 4)
│   │   │   ├── hooks/          #   - e.g. useSwipeDeck, useDateScrubber
│   │   │   ├── lib/            #   - api client, .ics export helper
│   │   │   ├── types/          #   - TypeScript interfaces (see Section 5)
│   │   │   └── main.tsx
│   │   ├── public/
│   │   ├── index.html
│   │   ├── vite.config.ts
│   │   ├── tailwind.config.ts
│   │   └── package.json
│   └── backend/                # Go REST API
│       ├── cmd/
│       │   └── api/
│       │       └── main.go     #   - entrypoint: wiring, router, Firestore client, server start
│       ├── internal/            #   - see Section 3 for full package breakdown
│       ├── go.mod
│       ├── go.sum
│       └── Dockerfile           #   - multi-stage build → distroless/scratch final image
│                                  #     for minimal Cloud Run cold-start size
├── .env.example                 # All env vars consumed by frontend (VITE_*) and backend
│                                # (PORT, GCP_PROJECT_ID, FIRESTORE_*, etc.), no real secrets
├── .gitignore                   # Go build artifacts, node_modules/dist, .terraform/*.tfstate,
│                                # .env, service-account keys
├── docker-compose.yml            # Local dev only: runs the Firestore emulator container
│                                # (no official Google image exists — use `gcr.io/
│                                # google.com/cloudsdktool/cloud-sdk` running `gcloud
│                                # emulators firestore start`). Frontend and backend run
│                                # natively via the Makefile for fast reload, not in
│                                # containers — this only replaces "install & run the
│                                # Firestore emulator by hand".
├── Makefile                     # See Section 6 and inline targets: dev, build, test, deploy, tf-*
├── CLAUDE.md                    # This file
└── README.md                    # Project overview, quickstart, contributor onboarding
```

**Rule:** application code lives only under `services/`. Nothing under `infra/` or `.github/` should contain business logic. Nothing under `docs/` is executable.

---

## 3. Go Backend Architecture Specs

### 3.1 Router

Use the **standard library** (`net/http`) as the HTTP router — no third-party router dependency. Go 1.22+'s `http.ServeMux` supports method-aware patterns and path wildcards (e.g. `mux.HandleFunc("GET /api/v1/events/{id}", ...)`), which covers everything this API needs. Rationale: zero extra dependency, smallest possible binary, nothing to learn beyond the standard library — consistent with the cold-start goals in Section 1. Do not introduce Chi, Gin, or Fiber (Fiber wraps fasthttp, which breaks `net/http` compatibility and complicates Cloud Run's request handling) unless `http.ServeMux` proves genuinely insufficient for a specific route — bring that decision back to this file if it happens.

### 3.2 Package layout (`services/backend/`)

```
services/backend/
├── cmd/api/main.go        # Composition root: load config, build dependencies, set up
│                           # structured logging, register routes, start http.Server
│                           # with graceful shutdown. No business logic here.
└── internal/
    ├── config/             # Env var loading + validation (PORT, GCP_PROJECT_ID, etc.)
    ├── service/              # HTTP layer: net/http `http.ServeMux` setup, route
    │   ├── router.go         #   registration, middleware.
    │   ├── middleware.go    #   - request logging, recover, CORS, auth (Firebase Auth
    │   │                    #     ID token verification lives here, not in a separate
    │   │                    #     platform package)
    │   └── handlers/        #   - one file per resource: events.go, pois.go, users.go,
    │                        #     preferences.go — thin: parse request → call usecase →
    │                        #     write response. No Firestore calls in handlers.
    ├── entities/            # Core types shared across layers (Go structs — see Section 5).
    │                        # No external deps in this package (no Firestore tags leak
    │                        # into business logic beyond struct tags).
    ├── usecase/              # Business logic: one package per bounded concern, operating
    │   ├── events/           #   on entity structs. Also owns private-event CRUD (Section
    │   │                      #     5.3) — validation, ownership checks, resolving a submitted
    │   │                      #     address via GeocodeRepository, and ingestion: calling each
    │   │                      #     EventSource and writing normalized results through
    │   │                      #     EventRepository into the shared `events` collection.
    │   ├── pois/              #   - evergreen POI queries
    │   ├── users/              #   - user profile + preferences
    │   └── planner/            #   - tier enforcement (1-day vs multi-day), .ics export.
    │                            #     Geo utilities (distance/bounding box) live in the
    │                            #     usecase package that needs them, not a shared
    │                            #     platform package.
    └── repository/           # Storage & external API access — the only layer with
        │                     # outbound integrations. Each file at this level is an
        │                     # interface (e.g. `user.go` declares `UserRepository`);
        │                     # `usecase/` depends on these interfaces, never on a
        │                     # concrete implementation directly.
        ├── user.go            #   - UserRepository interface
        ├── event.go            #   - EventRepository interface: our own storage, read/write
        │                       #     the shared `events` collection and private events
        ├── eventsource.go       #   - EventSource interface: FetchEvents(ctx, ...) ([]entities.Event,
        │                       #     error) — one implementation per external event listing site,
        │                       #     all normalizing into the same Event type (see `http/` below)
        ├── poi.go              #   - POIRepository interface
        ├── geocoder.go          #   - GeocodeRepository interface (address string → Location)
        ├── firestore/           # Concrete implementation of the storage interfaces above —
        │   ├── firestore.go     #   the ONLY subpackage that imports
        │   ├── user.go          #   cloud.google.com/go/firestore. If a second storage
        │   ├── event.go         #   backend is ever needed, it gets its own sibling
        │   └── poi.go           #   subpackage here (e.g. repository/memory/) implementing
        │                        #   the same interfaces from repository/.
        └── http/                 # Concrete implementations of every outbound HTTP integration —
            ├── nominatim.go       #   named by transport, not by vendor, since it holds several
            │                      #   unrelated providers: GeocodeRepository against Nominatim
            │                      #   (Section 1.2)...
            └── <source>.go         #   ...plus one EventSource implementation per external
                                    #   event site/API (e.g. eventbrite.go, meetup.go — named
                                    #   for whichever providers actually get integrated).
```

**Dependency direction:** `service` (handlers) → `usecase` → `repository` → Firestore. `entities` has no outward dependencies and is imported by all layers. Never let `service/handlers` call `repository` directly, and never let `repository` contain business rules (e.g., tier limits belong in `usecase/planner`, not in a Firestore query file).

### 3.3 Firestore SDK integration

- Use the official `cloud.google.com/go/firestore` client, instantiated once in `repository/firestore/firestore.go` and injected via the composition root in `cmd/api/main.go` (constructor injection, no globals/singletons).
- Collections (initial): `events`, `pois`, `users`, `preferences` — exact schema in Section 5. Use Firestore document IDs that are meaningful where possible (e.g., user doc ID = Firebase Auth UID) to avoid an extra lookup index.
- Private events (Section 5.3) live in a **user-scoped subcollection** — `users/{uid}/privateEvents/{eventId}` — not the shared `events` collection, since they're personal to that user and must never appear in another user's candidate pool. `EventRepository` takes a `uid` for private-event operations and routes to the right collection internally.
- Reads that back the Map/Schedule views should be scoped by geohash + date range where feasible — plan for a `geohash` field on `events`/`pois` documents to support bounding-box queries without a paid geo-index service.
- Cloud Run's service account (defined in `infra/iam.tf`) gets least-privilege IAM (`roles/datastore.user`) — never broader Firestore/Owner roles.
- Local development talks to the Firestore emulator, run via `docker-compose.yml` and started as part of `make dev` (see Section 6) — never point local dev at production Firestore.

### 3.4 API conventions

- JSON in/out, `application/json` only.
- Auth: Firebase Auth ID tokens verified via middleware in `service/middleware.go`; unauthenticated routes are the exception, not the default.
- Errors: a single consistent error envelope (`{"error": {"code": "...", "message": "..."}}`), mapped from typed errors defined in `usecase/`, not raw Firestore errors leaking to clients.
- Versioning: prefix routes with `/api/v1` from day one to avoid a breaking migration later.
- Address lookup: `GET /api/v1/geocode?address=...` proxies to Nominatim (`repository/http/nominatim.go`, see Section 3.2/1.2) and returns a `Location`, used by the "add a private event" flow to turn free-text input into coordinates before the event is saved.

---

## 4. Client-Side State & Swipe List Strategy

The Schedule View's core promise is **zero API spam**: swiping through the card deck must never fire a network request per swipe. Sorting is deliberately simple — two lists, no weighting or re-ranking.

### 4.1 Data flow

1. On date selection, the frontend makes **one** request (`GET /api/v1/plan?date=...&lat=...&lng=...`) that returns the full candidate pool of events + POIs for that day within range.
2. The pool is held in client state (React context or a small store, e.g. Zustand — evaluate at implementation time, but keep it dependency-light) and shown to the user one card at a time, in the order the backend returned it.
3. Each swipe (`useSwipeDeck` hook, backed by `react-swipeable`) does one thing, purely client-side:
   - **Right (save):** add the card to the **current list** — the list the user is actively building for the selected date. Create it on the first save if it doesn't exist yet.
   - **Left (skip):** add the card to a single **generic catch-all list** shared across all skipped cards, then advance to the next card.
4. Saved selections (both lists) are synced to the backend on a fixed 3-second interval whenever there are unsynced changes — never per-swipe, but frequent enough that a crashed tab or lost connection loses at most ~3 seconds of swipes. No explicit "sync" action for the user to remember to hit.
5. Private events (Section 5.3) don't go through the swipe deck at all — the user adds them directly via a form (title, time, address → geocoded to a `Location`), and they're inserted straight into the **current list** alongside anything swiped right.

### 4.2 Why this shape

- Keeps Cloud Run request volume proportional to *sessions*, not *swipes* — directly protects the scale-to-zero cost model in Section 1.
- Keeps the feed feeling instant (no network round-trip in the interaction loop), which matters for a swipe-gesture UI.
- No weighting or ranking logic to build, test, or reason about — the deck order is exactly what the backend returned; a swipe only decides which of the two lists a card lands in.

---

## 5. Data Models

Keep Go structs (backend/Firestore) and TypeScript interfaces (frontend/API contract) in sync by hand for now — both are listed below as the source of truth. If/when this drifts, consider generating one from the other, but don't build that tooling preemptively.

### 5.1 Location

```go
// entities/location.go
type Location struct {
    Lat   float64 `json:"lat" firestore:"lat"`
    Lng   float64 `json:"lng" firestore:"lng"`
    Label string  `json:"label,omitempty" firestore:"label,omitempty"` // e.g. "Home", "Weekend trip"
}
```

```ts
// types/location.ts
export interface Location {
  lat: number;
  lng: number;
  label?: string; // "Home", "Weekend trip"
}
```

### 5.2 Category (shared enum)

The 25 most popular activities a user is likely to search for, ordered roughly by expected search frequency. Every value is a single word — no multi-word compounds (`gaming`, not `gaming_esports`) — and there is deliberately no generic "other sports" catch-all; specific sports are broken out into their own categories instead, so search filters stay meaningful.

```go
type Category string

const (
    CategoryFood       Category = "food"
    CategoryMusic      Category = "music"        // has Genre sub-tag
    CategoryCinema     Category = "cinema"
    CategoryShopping   Category = "shopping"
    CategoryNightlife  Category = "nightlife"
    CategoryFitness    Category = "fitness"
    CategoryFamily     Category = "family"
    CategoryOutdoors   Category = "outdoors"
    CategoryComedy     Category = "comedy"
    CategoryTheatre    Category = "theatre"
    CategoryGaming     Category = "gaming"
    CategoryArts       Category = "arts"
    CategoryMarkets    Category = "markets"
    CategoryFootball   Category = "football"
    CategoryBasketball Category = "basketball"
    CategoryTennis     Category = "tennis"
    CategoryRunning    Category = "running"
    CategoryCycling    Category = "cycling"
    CategorySwimming   Category = "swimming"
    CategoryGolf       Category = "golf"
    CategoryAnime      Category = "anime"
    CategoryWellness   Category = "wellness"
    CategoryNetworking Category = "networking"
    CategoryKids       Category = "kids"
    CategoryEsports    Category = "esports"
)
```

```ts
export type Category =
  | "food"
  | "music"
  | "cinema"
  | "shopping"
  | "nightlife"
  | "fitness"
  | "family"
  | "outdoors"
  | "comedy"
  | "theatre"
  | "gaming"
  | "arts"
  | "markets"
  | "football"
  | "basketball"
  | "tennis"
  | "running"
  | "cycling"
  | "swimming"
  | "golf"
  | "anime"
  | "wellness"
  | "networking"
  | "kids"
  | "esports";
```

### 5.3 Event (Transient)

An `Event` is either sourced from a public feed (the shared candidate pool everyone swipes through) or added privately by a user — e.g. a relative's wedding — via a manual "add your own event" flow. Private events have no `SourceURL`, are never shown to other users, and get their `Location` from the address-lookup (geocoding) flow described in Section 1.2 / Section 3.2 rather than from an ingested feed. See Section 3.3 for where they're stored.

```go
// entities/event.go
type Event struct {
    ID          string     `json:"id" firestore:"-"` // Firestore doc ID
    Title       string     `json:"title" firestore:"title"`
    Description string     `json:"description,omitempty" firestore:"description,omitempty"`
    Category    Category   `json:"category" firestore:"category"`
    Genre       string     `json:"genre,omitempty" firestore:"genre,omitempty"` // for music
    Location    Location   `json:"location" firestore:"location"`
    Geohash     string     `json:"-" firestore:"geohash"` // query support, not exposed to client
    StartTime   time.Time  `json:"startTime" firestore:"startTime"`
    EndTime     time.Time  `json:"endTime" firestore:"endTime"`
    IsTransient bool       `json:"isTransient" firestore:"isTransient"` // true, always, for Event
    IsPrivate   bool       `json:"isPrivate" firestore:"isPrivate"` // true for user-added personal events; excluded from the shared candidate pool
    SourceURL   string     `json:"sourceUrl,omitempty" firestore:"sourceUrl,omitempty"` // always empty when IsPrivate
    ImageURL    string     `json:"imageUrl,omitempty" firestore:"imageUrl,omitempty"`
}
```

```ts
export interface Event {
  id: string;
  title: string;
  description?: string;
  category: Category;
  genre?: string;
  location: Location;
  startTime: string; // ISO 8601
  endTime: string;   // ISO 8601
  isTransient: true;
  isPrivate: boolean; // true for user-added personal events; excluded from the shared candidate pool
  sourceUrl?: string; // always absent when isPrivate
  imageUrl?: string;
}
```

### 5.4 EvergreenPOI

```go
// entities/poi.go
type EvergreenPOI struct {
    ID          string   `json:"id" firestore:"-"`
    Name        string   `json:"name" firestore:"name"`
    Description string   `json:"description,omitempty" firestore:"description,omitempty"`
    Category    Category `json:"category" firestore:"category"`
    Location    Location `json:"location" firestore:"location"`
    Geohash     string   `json:"-" firestore:"geohash"`
    OpeningHours map[string]string `json:"openingHours,omitempty" firestore:"openingHours,omitempty"` // "mon": "09:00-18:00"
    IsTransient bool     `json:"isTransient" firestore:"isTransient"` // false, always, for POI
    ImageURL    string   `json:"imageUrl,omitempty" firestore:"imageUrl,omitempty"`
}
```

```ts
export interface EvergreenPOI {
  id: string;
  name: string;
  description?: string;
  category: Category;
  location: Location;
  openingHours?: Record<string, string>; // { mon: "09:00-18:00" }
  isTransient: false;
  imageUrl?: string;
}

export type PlanItem = Event | EvergreenPOI; // discriminated union on isTransient
```

### 5.5 UserPreferences

```go
// entities/user_preferences.go
type UserPreferences struct {
    UserID           string     `json:"userId" firestore:"-"` // == Firebase Auth UID, doc ID
    PrimaryLocation  Location   `json:"primaryLocation" firestore:"primaryLocation"`
    SecondaryLocations []Location `json:"secondaryLocations,omitempty" firestore:"secondaryLocations,omitempty"`
    Interests        []Category `json:"interests" firestore:"interests"` // from onboarding
    Tier             UserTier   `json:"tier" firestore:"tier"`
    CreatedAt        time.Time  `json:"createdAt" firestore:"createdAt"`
    UpdatedAt        time.Time  `json:"updatedAt" firestore:"updatedAt"`
}

type UserTier string

const (
    TierFree UserTier = "free" // 1-day browsing limit
    TierPaid UserTier = "paid" // multi-day / 90-day / 1-year + .ics export
)
```

```ts
export type UserTier = "free" | "paid";

export interface UserPreferences {
  userId: string;
  primaryLocation: Location;
  secondaryLocations?: Location[];
  interests: Category[];
  tier: UserTier;
  createdAt: string; // ISO 8601
  updatedAt: string; // ISO 8601
}
```

### 5.6 SwipeAction (client-side only — not persisted to Firestore by default, see Section 4.1)

```ts
// types/swipe.ts — no Go struct: this never crosses the network in the default flow
export type SwipeDirection = "left" | "right"; // left = skip → catch-all list, right = save → current list

export interface SwipeAction {
  itemId: string;
  itemType: "event" | "poi";
  direction: SwipeDirection;
  timestamp: string; // ISO 8601, local
}
```

If/when batched sync of swipe history is implemented (see Section 4.1), define a corresponding minimal Go struct in `entities/` at that time — don't pre-build it now.

---

## 6. Step-by-Step Implementation Roadmap

Each step below is scoped to be a reasonable unit of work for a single future Claude Code session. Don't jump ahead — later steps assume earlier ones exist and follow these conventions.

1. **Repo scaffolding**
   - Create `.gitignore`, `.env.example`, `README.md`.
   - Scaffold `services/backend/` (`go.mod`, `cmd/api/main.go` with a bare `net/http` `http.ServeMux` + `/healthz`, empty `internal/` packages per Section 3.2, `Dockerfile`).
   - Scaffold `services/frontend/` via Vite React-TS template; wire up Tailwind, Shadcn init, Lucide.
   - Write `docker-compose.yml` to run the Firestore emulator locally (see Section 2).
   - Write the `Makefile` with targets: `dev` (starts the `docker-compose` Firestore emulator, then runs frontend + backend concurrently), `build`, `test`, `lint`, `deploy-frontend`, `deploy-backend`, `tf-plan`, `tf-apply`.

2. **Infra baseline**
   - `infra/cloud_run.tf`, `infra/firestore.tf`, `infra/artifact_registry.tf`, `infra/iam.tf`: Cloud Run service (min instances 0), Firestore database (Native Mode), Artifact Registry repo, service accounts + least-privilege IAM — one resource area per file rather than a single monolithic file.
   - `infra/variables.tf` / `outputs.tf`, parameterized per environment (production vs. developer — see Section 1.3).
   - GitHub Actions workflows: `ci.yml` (lint/test on PR for both services), `deploy-frontend.yml` / `deploy-backend.yml` (deploy to the developer environment on merge to `develop`, to production on merge to `main`).

3. **Backend core**
   - Implement `entities/` structs from Section 5.
   - Implement `repository/` interfaces and their `repository/firestore/` implementations (events, POIs, users) against the local emulator.
   - Implement `repository/http/nominatim.go` (`GeocodeRepository`) and the `/api/v1/geocode` proxy endpoint. Add the first `EventSource` implementation under `repository/http/` once a real external event feed is chosen.
   - Implement `usecase/` layer with basic CRUD + date/geo filtering, including private-event CRUD scoped to `users/{uid}/privateEvents`.
   - Implement `service/handlers` + Firebase Auth middleware; wire `/api/v1/...` routes.
   - Seed script/fixtures for local dev sample events & POIs.

4. **Frontend core**
   - Onboarding flow: location input(s) + interest category picker (Section 2 of business rules).
   - API client (`lib/api.ts`) and TypeScript types from Section 5.
   - `MapView`: React-Leaflet + OSM tiles, 1-day time scrubber, pins for events/POIs with transient/evergreen badges.
   - `ScheduleView`: swipeable card deck, `useSwipeDeck` hook (right → current list, left → catch-all list, per Section 4), transient/evergreen badges.
   - "Add a private event" form: title, time, address (autocomplete against `/api/v1/geocode`) → inserted directly into the current list, badged distinctly from transient/evergreen items.

5. **Tiering & monetization**
   - `usecase/planner` tier enforcement: free tier hard-limited to 1-day queries.
   - Paid tier: multi-day/90-day/1-year planning endpoints.
   - `.ics` export (likely client-side generation from saved `PlanItem[]`, no backend needed — evaluate against cost rules in Section 1).
   - Stripe (or chosen provider) integration for tier upgrades — design as its own future session, not bundled into this step.

6. **Polish & hardening**
   - Empty/error/loading states across both views.
   - Rate limiting / abuse protection on public endpoints.
   - Observability: structured logging (Cloud Logging via stdout JSON), basic uptime check in Terraform.
   - `docs/architecture.md` and `docs/api-spec.md` written up to reflect what was actually built.

**When starting any session against this roadmap:** confirm which step you're on, re-read the relevant section of this file, and do not skip ahead to a later step's concerns (e.g., don't add Stripe billing logic while still on step 3).
