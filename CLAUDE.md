# CLAUDE.md — planyour.day Project Instruction Manual

This file is the operating manual for Claude Code sessions working on **planyour.day**. Read it before making changes. It defines the architecture, the rules that keep the project cheap to run, the repository layout, the backend/frontend conventions, the core data models, and the roadmap for future sessions.

> **App summary:** planyour.day is an AI-powered spatial-temporal social planner that helps users discover and organize activities based on location, interests, and time. Core interaction loop: pick a date → browse a map or a swipeable card feed of events/POIs → save what you like → export or revisit your plan.

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

### 1.3 Infra & CI/CD

- **Terraform** is the only way GCP resources are created or modified. Nobody click-ops in the GCP console; state must reflect reality.
- **GitHub Actions** builds/deploys the frontend to Firebase Hosting and the backend container to Cloud Run on merge to main. Keep workflows minimal — build, test, deploy — resist adding infra that isn't in Terraform.
- **Makefile** is the single entry point for local dev so contributors (and Claude Code sessions) never need to remember multi-step tooling incantations.

### 1.4 Rules of thumb for future changes

1. **Would this component bill while idle?** If yes, justify it explicitly or find a scale-to-zero alternative.
2. **Does this need a server at all?** Prefer static/client-side solutions (like the swipe re-ranking in §4) over adding backend endpoints.
3. **Does this require a paid API key?** Avoid unless there's no free/open alternative (OSM over Mapbox, etc.).
4. **Does this fit in Firestore?** Prefer modeling new data in Firestore over introducing a second datastore.

---

## 2. Repository Directory Tree & File Purpose

```
planyour.day/
├── .github/
│   └── workflows/            # CI/CD: lint/test/build on PR; deploy frontend → Firebase
│                              # Hosting and backend → Cloud Run on merge to main.
│                              # Also houses Terraform plan/apply workflows for infra/.
├── docs/                     # Human-facing documentation, not code:
│   ├── architecture.md       #   - system diagram, data flow, scale-to-zero rationale
│   ├── api-spec.md           #   - REST endpoint contracts (request/response shapes)
│   └── monetization.md       #   - tier definitions, entitlement rules, pricing notes
├── infra/                    # Terraform for all GCP resources.
│   ├── main.tf                #   - Cloud Run service, Firestore database, IAM bindings,
│   │                          #     Artifact Registry (or GCR) for backend images
│   ├── variables.tf           #   - project_id, region, environment, image tag, etc.
│   └── outputs.tf              #   - Cloud Run URL, Firestore name, service account emails
├── services/
│   ├── frontend/              # Vite + React (TypeScript) SPA
│   │   ├── src/
│   │   │   ├── components/    #   - shared UI (Shadcn-based) + feature components
│   │   │   ├── views/          #   - MapView, ScheduleView (see §4)
│   │   │   ├── hooks/          #   - e.g. useSwipeWeights, useDateScrubber
│   │   │   ├── lib/            #   - api client, ranking engine, .ics export helper
│   │   │   ├── types/          #   - TypeScript interfaces (see §5)
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
│       ├── internal/            #   - see §3 for full package breakdown
│       ├── go.mod
│       ├── go.sum
│       └── Dockerfile           #   - multi-stage build → distroless/scratch final image
│                                  #     for minimal Cloud Run cold-start size
├── .env.example                 # All env vars consumed by frontend (VITE_*) and backend
│                                # (PORT, GCP_PROJECT_ID, FIRESTORE_*, etc.), no real secrets
├── .gitignore                   # Go build artifacts, node_modules/dist, .terraform/*.tfstate,
│                                # .env, service-account keys
├── Makefile                     # See §6 and inline targets: dev, build, test, deploy, tf-*
├── CLAUDE.md                    # This file
└── README.md                    # Project overview, quickstart, contributor onboarding
```

**Rule:** application code lives only under `services/`. Nothing under `infra/` or `.github/` should contain business logic. Nothing under `docs/` is executable.

---

## 3. Go Backend Architecture Specs

### 3.1 Router

Use **Chi** (`go-chi/chi`) as the HTTP router. Rationale: it's a thin, idiomatic layer over `net/http` (no framework magic), has first-class middleware composition, and keeps the binary lean — consistent with the cold-start goals in §1. Do not introduce Fiber (it wraps fasthttp, which breaks `net/http` compatibility and complicates Cloud Run's request handling) or a heavier framework like Gin unless Chi proves genuinely insufficient.

### 3.2 Package layout (`services/backend/`)

```
services/backend/
├── cmd/api/main.go        # Composition root only: load config, build dependencies,
│                           # register routes, start http.Server with graceful shutdown.
│                           # No business logic here.
└── internal/
    ├── config/             # Env var loading + validation (PORT, GCP_PROJECT_ID, etc.)
    ├── httpapi/             # Chi router setup, route registration, middleware
    │   ├── router.go
    │   ├── middleware.go    #   - request logging, recover, CORS, auth (Firebase Auth
    │   │                    #     ID token verification)
    │   └── handlers/        #   - one file per resource: events.go, pois.go, users.go,
    │                        #     preferences.go — thin: parse request → call service →
    │                        #     write response. No Firestore calls in handlers.
    ├── domain/              # Core types shared across layers (Go structs — see §5).
    │                        # No external deps in this package (no Firestore tags leak
    │                        # into business logic beyond struct tags).
    ├── service/              # Business logic: one package per bounded concern
    │   ├── events/           #   - transient event queries, filtering by date/location
    │   ├── pois/              #   - evergreen POI queries
    │   ├── users/              #   - user profile + preferences
    │   └── planner/            #   - tier enforcement (1-day vs multi-day), .ics export
    ├── store/                # Firestore access layer — the ONLY package that imports
    │   │                     # cloud.google.com/go/firestore. Repository-style interfaces
    │   │                     # per collection (EventRepository, POIRepository, etc.) so
    │   │                     # service/ depends on interfaces, not Firestore directly.
    │   ├── firestore.go       #   - client init, shared helpers
    │   ├── events_repo.go
    │   ├── pois_repo.go
    │   └── users_repo.go
    └── platform/              # Cross-cutting infra concerns: logging setup, Firebase Auth
                               # verification client, geo utilities (distance/bounding box).
```

**Dependency direction:** `handlers` → `service` → `store` → Firestore. `domain` has no outward dependencies and is imported by all layers. Never let `httpapi/handlers` call `store` directly, and never let `store` contain business rules (e.g., tier limits belong in `service/planner`, not in a Firestore query file).

### 3.3 Firestore SDK integration

- Use the official `cloud.google.com/go/firestore` client, instantiated once in `store/firestore.go` and injected via the composition root in `cmd/api/main.go` (constructor injection, no globals/singletons).
- Collections (initial): `events`, `pois`, `users`, `preferences` — exact schema in §5. Use Firestore document IDs that are meaningful where possible (e.g., user doc ID = Firebase Auth UID) to avoid an extra lookup index.
- Reads that back the Map/Schedule views should be scoped by geohash + date range where feasible — plan for a `geohash` field on `events`/`pois` documents to support bounding-box queries without a paid geo-index service.
- Cloud Run's service account (defined in `infra/main.tf`) gets least-privilege IAM (`roles/datastore.user`) — never broader Firestore/Owner roles.
- Local development talks to the Firestore emulator (wired through the `Makefile`, see §6) — never point local dev at production Firestore.

### 3.4 API conventions

- JSON in/out, `application/json` only.
- Auth: Firebase Auth ID tokens verified via middleware in `httpapi/middleware.go`; unauthenticated routes are the exception, not the default.
- Errors: a single consistent error envelope (`{"error": {"code": "...", "message": "..."}}`), mapped from typed domain errors in `service/`, not raw Firestore errors leaking to clients.
- Versioning: prefix routes with `/api/v1` from day one to avoid a breaking migration later.

---

## 4. Client-Side State & Swipe Weighting Strategy

The Schedule View's core promise is **zero API spam**: swiping through the card deck must never fire a network request per swipe. All re-ranking happens client-side against the day's already-fetched candidate pool.

### 4.1 Data flow

1. On date selection, the frontend makes **one** request (`GET /api/v1/plan?date=...&lat=...&lng=...`) that returns the full candidate pool of events + POIs for that day within range.
2. The pool is held in client state (React context or a small store, e.g. Zustand — evaluate at implementation time, but keep it dependency-light) alongside a **local weights object** keyed by category/tag (e.g. `{ music: 0, football: 0, foodAndBaking: 0, ... }` seeded from onboarding interests).
3. Each swipe (`useSwipeWeights` hook, backed by `react-swipeable`) does two things purely client-side:
   - **Right (save):** add the card to the user's saved-for-the-day list; bump the weight of every category/tag on that card upward.
   - **Left (skip):** discard the card; bump the weight of every category/tag on that card downward.
4. After each swipe, the **remaining** unseen cards in the pool are re-sorted by a locally-computed score (weighted sum over each card's tags, optionally decayed by recency of the swipe that produced the weight change). No card is re-fetched or re-validated against the backend for this.
5. Saved selections are only synced to the backend in a batched call (e.g., on view exit, on explicit "sync" action, or debounced) — never per-swipe.

### 4.2 Why this shape

- Keeps Cloud Run request volume proportional to *sessions*, not *swipes* — directly protects the scale-to-zero cost model in §1.
- Keeps the feed feeling instant (no network round-trip in the interaction loop), which matters for a swipe-gesture UI.
- The ranking function itself should live in `services/frontend/src/lib/ranking.ts` as a pure function `(pool, weights) => rankedPool` so it's unit-testable without any DOM/gesture dependency.

### 4.3 Persistence of weights

- Weights persist in `localStorage` (or `IndexedDB` if the pool grows large) keyed by user, so returning to the app mid-day keeps the personalization without a backend round-trip.
- Weights are **not** the same thing as `UserPreferences` (see §5) — `UserPreferences` are explicit onboarding choices synced to Firestore; swipe weights are an ephemeral, local, fast-moving signal layered on top. Only consider promoting aggregated swipe signal to the backend as a deliberate, explicit future feature (e.g., a periodic batched "preference learning" sync) — not by default.

---

## 5. Data Models

Keep Go structs (backend/Firestore) and TypeScript interfaces (frontend/API contract) in sync by hand for now — both are listed below as the source of truth. If/when this drifts, consider generating one from the other, but don't build that tooling preemptively.

### 5.1 Location

```go
// domain/location.go
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

```go
type Category string

const (
    CategoryMusic       Category = "music"        // has Genre sub-tag
    CategoryGamingEsports Category = "gaming_esports"
    CategoryAnime       Category = "anime"
    CategoryBakingFood  Category = "baking_food"
    CategoryCinema      Category = "cinema"
    CategoryFootball    Category = "football"
    CategoryTennis      Category = "tennis"
    CategoryOtherSports Category = "other_sports"
    CategoryArtsOutdoors Category = "arts_outdoors"
)
```

```ts
export type Category =
  | "music"
  | "gaming_esports"
  | "anime"
  | "baking_food"
  | "cinema"
  | "football"
  | "tennis"
  | "other_sports"
  | "arts_outdoors";
```

### 5.3 Event (Transient)

```go
// domain/event.go
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
    SourceURL   string     `json:"sourceUrl,omitempty" firestore:"sourceUrl,omitempty"`
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
  sourceUrl?: string;
  imageUrl?: string;
}
```

### 5.4 EvergreenPOI

```go
// domain/poi.go
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
// domain/user_preferences.go
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

### 5.6 SwipeAction (client-side only — not persisted to Firestore by default, see §4.3)

```ts
// types/swipe.ts — no Go struct: this never crosses the network in the default flow
export type SwipeDirection = "left" | "right"; // left = skip, right = save

export interface SwipeAction {
  itemId: string;
  itemType: "event" | "poi";
  direction: SwipeDirection;
  categories: Category[]; // tags carried by the swiped item, used to update weights
  timestamp: string; // ISO 8601, local
}

export interface SwipeWeights {
  [category: string]: number; // running score per Category, seeded from UserPreferences.interests
}
```

If/when batched sync of swipe history is implemented (see §4.3), define a corresponding minimal Go struct in `domain/` at that time — don't pre-build it now.

---

## 6. Step-by-Step Implementation Roadmap

Each step below is scoped to be a reasonable unit of work for a single future Claude Code session. Don't jump ahead — later steps assume earlier ones exist and follow these conventions.

1. **Repo scaffolding**
   - Create `.gitignore`, `.env.example`, `README.md`.
   - Scaffold `services/backend/` (`go.mod`, `cmd/api/main.go` with a bare Chi router + `/healthz`, empty `internal/` packages per §3.2, `Dockerfile`).
   - Scaffold `services/frontend/` via Vite React-TS template; wire up Tailwind, Shadcn init, Lucide.
   - Write the `Makefile` with targets: `dev` (run frontend + backend + Firestore emulator concurrently), `build`, `test`, `lint`, `deploy-frontend`, `deploy-backend`, `tf-plan`, `tf-apply`.

2. **Infra baseline**
   - `infra/main.tf`: Cloud Run service (min instances 0), Firestore database (Native Mode), Artifact Registry repo, service accounts + least-privilege IAM.
   - `infra/variables.tf` / `outputs.tf`.
   - GitHub Actions workflows: `ci.yml` (lint/test on PR for both services), `deploy-frontend.yml` (build + `firebase deploy --only hosting`), `deploy-backend.yml` (build container, push, `gcloud run deploy`).

3. **Backend core**
   - Implement `domain/` structs from §5.
   - Implement `store/` Firestore repositories (events, POIs, users) against the local emulator.
   - Implement `service/` layer with basic CRUD + date/geo filtering.
   - Implement `httpapi/handlers` + Firebase Auth middleware; wire `/api/v1/...` routes.
   - Seed script/fixtures for local dev sample events & POIs.

4. **Frontend core**
   - Onboarding flow: location input(s) + interest category picker (§2 of business rules).
   - API client (`lib/api.ts`) and TypeScript types from §5.
   - `MapView`: React-Leaflet + OSM tiles, 1-day time scrubber, pins for events/POIs with transient/evergreen badges.
   - `ScheduleView`: swipeable card deck, `useSwipeWeights` hook, `lib/ranking.ts` pure ranking function (§4), transient/evergreen badges.

5. **Tiering & monetization**
   - `service/planner` tier enforcement: free tier hard-limited to 1-day queries.
   - Paid tier: multi-day/90-day/1-year planning endpoints.
   - `.ics` export (likely client-side generation from saved `PlanItem[]`, no backend needed — evaluate against cost rules in §1).
   - Stripe (or chosen provider) integration for tier upgrades — design as its own future session, not bundled into this step.

6. **Polish & hardening**
   - Empty/error/loading states across both views.
   - Rate limiting / abuse protection on public endpoints.
   - Observability: structured logging (Cloud Logging via stdout JSON), basic uptime check in Terraform.
   - `docs/architecture.md`, `docs/api-spec.md`, `docs/monetization.md` written up to reflect what was actually built.

7. **Beyond MVP (do not start without explicit direction)**
   - Batched swipe-history sync → server-side preference learning.
   - Social features (shared plans, following).
   - Push notifications for saved/starting-soon events.

**When starting any session against this roadmap:** confirm which step you're on, re-read the relevant section of this file, and do not skip ahead to a later step's concerns (e.g., don't add Stripe billing logic while still on step 3).
