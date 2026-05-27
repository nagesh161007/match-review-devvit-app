# Match Review Autopilot

A [Devvit](https://developers.reddit.com/) app that automates post-match player ratings for football team subreddits — lineup voting, Reddit love, Man of the Match, and season-long **Player of the Month / Season** awards.

**Production:** hands-off autopilot (sync → open at kickoff + 120 min → close → aggregate → publish awards).  
**Demo:** mod menu can manually stagger posts into the mod queue for playtesting.

---

## Table of contents

- [What it does](#what-it-does)
- [Architecture](#architecture)
- [Media upload](#media-upload)
- [Mod menu](#mod-menu)
- [Scheduler](#scheduler)
- [Setup](#setup)
- [Project structure](#project-structure)

---

## What it does

- **Post-match voting** — Interactive custom posts: 5-star ratings, Reddit love credits, Man of the Match (crown).
- **Results dashboards** — Post-match and full-screen period dashboards (POTM, Most Loved, POTSeason).
- **Intelligent scheduling** — Devvit Scheduler for publish/close jobs and lifecycle ticks; failed steps skip/retry without blocking the pipeline.
- **Incremental aggregation** — Compact Redis aggregates merge into month/season rollups as each match closes.
- **Team setup** — Mod enters **team name** on install; SportMonks search resolves the club ID.
- **Media on Reddit** — Player headshots and team logos uploaded via `media.upload()` so webviews render reliably.

---

## Architecture

### System overview

```mermaid
flowchart TB
  subgraph Reddit["Reddit Platform"]
    Mod["Moderator"]
    Fan["Fan / Voter"]
    Feed["Subreddit feed"]
    ModQueue["Mod queue"]
  end

  subgraph Devvit["Devvit App match-review"]
    subgraph Client["Client React"]
      Splash["splash.html inline feed"]
      Game["game.html full-screen"]
      Lineup["LineUp voting UI"]
      Dashboard["Dashboard PeriodDashboard"]
    end

    subgraph Server["Server Hono Node"]
      API["/api"]
      Menu["/internal/menu"]
      SchedulerEP["/internal/scheduler"]
      SettingsEP["/internal/settings"]
      TriggersEP["/internal/triggers"]
      Store["MatchReviewStore"]
    end

    subgraph Jobs["Devvit Scheduler"]
      CronSync["syncFixtures weekly cron"]
      CronLife["lifecycleTick every minute"]
      JobClose["closeVoting one-off"]
      JobMatch["publishMatchPost one-off"]
      JobPeriod["publishPeriodPost one-off"]
      JobMedia["syncSquadMedia one-off"]
    end

    Redis[("Devvit Redis")]
  end

  subgraph External["External APIs"]
    SM["SportMonks API"]
    SMCDN["SportMonks CDN"]
  end

  Mod --> Feed
  Fan --> Feed
  Feed --> Splash
  Feed --> Game
  Splash --> Lineup
  Game --> Dashboard
  Splash --> API
  Game --> API
  API --> Store
  Mod --> Menu
  Menu --> Store
  CronSync --> SchedulerEP
  CronLife --> SchedulerEP
  JobClose --> SchedulerEP
  JobMatch --> SchedulerEP
  JobPeriod --> SchedulerEP
  JobMedia --> SchedulerEP
  SchedulerEP --> Store
  TriggersEP --> Store
  SettingsEP --> Store
  Store --> Redis
  Store --> SM
  Store --> SMCDN
  Store --> ModQueue
  Store --> Feed
```

### Install and team connection

```mermaid
sequenceDiagram
  autonumber
  participant Mod as Moderator
  participant Reddit as Reddit install UI
  participant Settings as on-team-configured
  participant SM as SportMonks search API
  participant Store as MatchReviewStore
  participant Redis as Devvit Redis

  Mod->>Reddit: Install app and enter team name
  Reddit->>Settings: Validate teamName
  Settings->>SM: GET teams search by query
  alt Team found
    SM-->>Settings: Team id and canonical name
    Settings->>Store: connectTeam
    Store->>Redis: runtime setupState ready
    Settings-->>Reddit: success
  else Not found or ambiguous
    SM-->>Settings: empty or multiple matches
    Settings-->>Reddit: error Team not found
  end
```

### Production autopilot vs demo scheduling

```mermaid
flowchart LR
  subgraph Production["Production autopilot"]
    A1["syncFixtures cron"] --> A2["Fixture index in Redis"]
    A2 --> A3["lifecycleTick cron"]
    A3 --> A4{openVotingAt due?}
    A4 -->|yes| A5["openVoting Reddit match post"]
    A5 --> A6["schedule closeVoting job"]
    A6 --> A7["closeFixture summary"]
    A7 --> A8["rollup month and season aggregates"]
    A8 --> A9["Auto POTM and POTSeason posts"]
  end

  subgraph Demo["Demo playtest mod menu"]
    D1["Schedule dashboard posts"] --> D2["prepareAggregatedDashboardData"]
    D2 --> D3["Queue publishPeriodPost T0 +3m +5m +10m"]
    D4["Create 3 recent match posts"] --> D5["Queue publishMatchPost staggered"]
  end

  A2 -.-> D2
```

### Match lifecycle

```mermaid
stateDiagram-v2
  [*] --> scheduled: syncFixtures stub fixture
  scheduled --> open: openVoting autopilot or publishMatchPost
  open --> closed: closeVoting scheduled or lifecycle fallback
  closed --> published: finalize aggregate and summary
  published --> [*]: rollup into period aggregates

  note right of scheduled
    openVotingAt equals kickoff plus 120 min
  end note

  note right of open
    schedule closeVoting at votingClosedAt
    ballot updates fixture aggregate in Redis
  end note
```

### Aggregation pipeline

```mermaid
flowchart TB
  Ballot["User ballot ratings love MOTM"] --> FA["fixture aggregate"]
  FA --> Close["closeFixture"]
  Close --> FS["fixture summary"]
  Close --> Rollup["rollupClosedFixture"]

  Rollup --> MA["month aggregate"]
  Rollup --> SA["season aggregate"]

  MA --> AwardsM["finalizePeriodAwards POTM and Most Loved"]
  SA --> AwardsS["finalizePeriodAwards Player of Season"]

  AwardsM --> PostM["Player of the Month post"]
  AwardsS --> PostS["Season Awards post"]

  subgraph Rebuild["Self-recovery"]
    R1["rebuildMonthAggregate"] --> MA
    R2["rebuildSeasonAggregate"] --> SA
  end
```

### Redis data model simplified

```mermaid
erDiagram
  RUNTIME ||--o{ FIXTURE : indexes
  FIXTURE ||--|| FIXTURE_AGGREGATE : has
  FIXTURE ||--o| FIXTURE_SUMMARY : produces
  FIXTURE ||--|| POST : links
  FIXTURE }o--|| MONTH_AGGREGATE : rollups
  FIXTURE }o--|| SEASON_AGGREGATE : rollups
  MONTH_AGGREGATE ||--o| MONTHLY_POST : publishes
  SEASON_AGGREGATE ||--o| SEASON_POST : publishes

  RUNTIME {
    string providerTeamId
    string teamName
    string setupState
  }

  FIXTURE {
    string fixtureId
    string status
    number openVotingAt
    number votingClosedAt
  }

  FIXTURE_AGGREGATE {
    int voterCount
    int totalRatings
    json players
  }

  MONTH_AGGREGATE {
    string month
    json players
    string playerOfTheMonth
    string mostLovedPlayer
  }

  SEASON_AGGREGATE {
    string seasonId
    json players
    string playerOfTheSeason
  }
```

### API efficiency

```mermaid
flowchart LR
  subgraph Tier1["Discovery sync cheap"]
    T1["fixtures between participants scores events"]
  end

  subgraph Tier2["Full fetch on demand"]
    T2["fixture by id lineups players coaches"]
  end

  subgraph Tier3["Media cache once"]
    T3["SportMonks CDN URL"]
    T3 --> M1["media.upload to Reddit"]
    M1 --> M2["entity cache in Redis"]
  end

  Tier1 --> Redis2[("Fixture stubs")]
  Tier2 --> Open["openVoting and rollup"]
  M2 --> Client2["Client UI"]
```

### Scheduler job map

```mermaid
flowchart TB
  subgraph Cron["Recurring devvit.json cron"]
    C1["syncFixtures Mon 03:00"]
    C2["lifecycleTick every 1 min"]
  end

  subgraph OneOff["One-off scheduler.runJob"]
    O1["closeVoting per fixture at votingClosedAt"]
    O2["publishMatchPost demo match stagger"]
    O3["publishPeriodPost demo award stagger"]
    O4["syncSquadMedia squad headshots"]
  end

  C1 --> EP1["scheduler sync-fixtures"]
  C2 --> EP2["scheduler lifecycle-tick"]
  O1 --> EP3["scheduler close-voting"]
  O2 --> EP4["scheduler publish-match-post"]
  O3 --> EP5["scheduler publish-period-post"]
  O4 --> EP6["scheduler sync-squad-media"]

  EP1 --> Store["MatchReviewStore"]
  EP2 --> Store
  EP3 --> Store
  EP4 --> Store
  EP5 --> Store
  EP6 --> Store
```

### Client entrypoints

```mermaid
flowchart LR
  Post["Reddit custom post"] --> Entry{Which entry?}

  Entry -->|default splash| Inline["Inline webview match centre and lineup"]
  Entry -->|game| Full["Full-screen period dashboards"]

  Inline --> API["GET api posts postId state"]
  Full --> API

  API --> State{postType}
  State -->|match| MatchState["fixture aggregate summary"]
  State -->|monthly| MonthState["month aggregate"]
  State -->|season| SeasonState["season aggregate"]
```

---

## Media upload

Devvit webviews cannot reliably load external CDN URLs. Images from SportMonks are uploaded to **Reddit via `media.upload()`**, cached in Redis, and only Reddit-hosted URLs are sent to the client.

### Why

| Problem | Solution |
|--------|----------|
| SportMonks CDN blocked or flaky in webview | Upload once to i.redd.it or redditmedia.com |
| Same player or team image across many fixtures | Entity cache in Redis upload once per entity |
| Squad sync is slow | Background scheduler job plus 24h resync skip plus lock |

### Media flow

```mermaid
flowchart TB
  subgraph Sources["Image sources"]
    SM["SportMonks CDN cdn.sportmonks.com"]
  end

  subgraph Server["Server"]
    Resolve["resolveRedditMediaUrl"]
    EnsureTeam["ensureTeamLogo"]
    EnsurePlayer["ensurePlayerHeadshot"]
    SquadSync["syncConnectedTeamSquad"]
    NormFixture["normalizeFixtureMedia"]
  end

  subgraph Devvit["Devvit platform"]
    Upload["media.upload url and type image"]
    Redis[("Devvit Redis")]
  end

  subgraph Client["React client"]
    Check["renderablePlayerImageUrl"]
    UI["LineUp and Dashboard avatars"]
  end

  SM --> EnsureTeam
  SM --> EnsurePlayer
  SM --> SquadSync
  SM --> NormFixture

  EnsureTeam --> Resolve
  EnsurePlayer --> Resolve
  NormFixture --> Resolve

  Resolve -->|cache miss| Upload
  Upload -->|mediaUrl| Redis
  Resolve -->|cache hit| Redis

  EnsureTeam --> Redis
  EnsurePlayer --> Redis

  Redis --> NormFixture
  NormFixture --> UI
  UI --> Check
  Check -->|Reddit URL only| UI
```

### Upload and cache logic

```mermaid
flowchart TD
  A["sourceUrl from SportMonks"] --> B{Already Reddit-hosted?}
  B -->|yes| C["Return as-is"]
  B -->|no| D{Redis cache hit?}
  D -->|hit| E["Return cached mediaUrl"]
  D -->|miss| F["media.upload url type image"]
  F -->|success| G["Store cache entry media cache by hash"]
  G --> H["Return mediaUrl"]
  F -->|fail| I["Return undefined UI shows initials"]
```

### Entity cache team and player

```mermaid
flowchart LR
  subgraph RedisKeys["Redis keys"]
    T["media team subredditId teamId"]
    P["media player subredditId playerId"]
    S["media squad subredditId teamId"]
  end

  subgraph Entries["Stored fields"]
    TE["TeamMediaEntry logoMediaUrl sourceImageUrl"]
    PE["PlayerMediaEntry headshotMediaUrl sourceImageUrl"]
    SE["TeamSquadMeta playerIds syncedAt"]
  end

  T --> TE
  P --> PE
  S --> SE
```

**ensureTeamLogo / ensurePlayerHeadshot logic:**

1. If entity entry already has logo or headshot URL → return it no re-upload.
2. If source URL is already Reddit-hosted → store and return.
3. Else → resolveRedditMediaUrl → store entity entry.

### Squad sync bulk headshots

```mermaid
sequenceDiagram
  autonumber
  participant Job as syncSquadMedia job
  participant Lock as Redis lock
  participant SM as SportMonks squads API
  participant Store as media-store
  participant Upload as media.upload
  participant Redis as Redis

  Job->>Lock: setNxEx squad lock 10 min
  alt lock held
    Lock-->>Job: skip already in progress
  else lock acquired
    Job->>SM: GET squads teams by teamId
    loop each player
      Job->>Store: ensurePlayerHeadshot
      Store->>Redis: check player media entry
      alt not cached
        Store->>Upload: upload SportMonks URL
        Upload-->>Store: Reddit mediaUrl
        Store->>Redis: save PlayerMediaEntry
      end
    end
    Job->>Store: ensureTeamLogo
    Job->>Redis: save TeamSquadMeta
    Job->>Lock: release lock
  end
```

### When uploads happen lazy vs bulk

```mermaid
flowchart TB
  subgraph Bulk["Bulk background"]
    B1["connectTeam"] --> B2["schedule syncSquadMedia"]
    B2 --> B3["All squad headshots and team logo"]
  end

  subgraph Lazy["Lazy on demand"]
    L1["openVoting"] --> L2["normalizeMatchDetailsImages"]
    L2 --> L3["Upload lineup and participant images"]
    L4["syncFixtures"] --> L5["ensureParticipantLogos"]
    L6["fixtureState post load"] --> L7["normalizeFixtureMedia if external URLs"]
  end

  subgraph ClientRule["Client safety net"]
    C1["renderablePlayerImageUrl"]
    C1 --> C2["Only render i.redd.it redditmedia.com"]
    C2 --> C3["Else show jersey initials"]
  end
```

### Media triggers

| Trigger | When |
|--------|------|
| connectTeam | Schedules background syncSquadMedia |
| Mod menu | Sync squad photos force |
| openVoting | normalizeMatchDetailsImages |
| fixture load | normalizeFixtureMedia if external URLs remain |

### Media Redis keys

| Key pattern | Contents |
|-------------|----------|
| media cache by source URL hash | URL hash cache after media.upload |
| media team subredditId teamId | Team logo |
| media player subredditId playerId | Player headshot |
| media squad subredditId teamId | Squad sync metadata |

### Failure handling

| Failure | Behavior |
|--------|----------|
| media.upload fails | Log warning return undefined UI shows initials |
| Invalid source URL | Skip upload |
| Squad sync in progress | Second job skips lock |
| Corrupt URL cache JSON | Re-upload on next resolve |

---

## Mod menu

| Action | Description |
|--------|-------------|
| Schedule dashboard posts | Roll up all matches queue POTM and POTSeason at T+0 +3m +5m +10m |
| Create 3 recent match posts | Queue 3 match posts staggered every 5 minutes |
| Sync squad photos | Force squad headshot and logo upload to Reddit |
| Reset app data | Clear Redis and remove app posts |

---

## Scheduler

| Task | Schedule | Endpoint |
|------|----------|----------|
| syncFixtures | Mon 03:00 UTC | /internal/scheduler/sync-fixtures |
| lifecycleTick | every 1 min | /internal/scheduler/lifecycle-tick |
| closeVoting | one-off at votingClosedAt | /internal/scheduler/close-voting |
| publishMatchPost | one-off demo | /internal/scheduler/publish-match-post |
| publishPeriodPost | one-off demo | /internal/scheduler/publish-period-post |
| syncSquadMedia | one-off | /internal/scheduler/sync-squad-media |

---

## Setup

### Prerequisites

- Node.js 18+
- [Devvit CLI](https://developers.reddit.com/docs/devvit_cli)
- SportMonks API token

### Install and configure

```bash
cd match-review-app
npm install
npm run build

npx devvit settings set sportmonksApiToken YOUR_TOKEN

npx devvit upload
npx devvit playtest
```

On install moderators enter **Team name** e.g. Manchester City. The app validates via SportMonks search and stores the resolved team ID in Redis.

### Dev subreddit

Configured in devvit.json:

```json
"dev": {
  "subreddit": "match_review_dev"
}
```

---

## Project structure

```
match-review-app/
├── devvit.json
├── assets/icon.png
├── src/
│   ├── client/
│   ├── server/
│   └── shared/
└── docs/
```

### Key server files

| File | Role |
|------|------|
| src/server/store/match-review-store.ts | Core store lifecycle aggregation scheduling |
| src/server/services/sportmonks/client.ts | SportMonks API tiered fetch |
| src/server/services/sportmonks/resolve-team.ts | Team name search on install |
| src/server/services/media/normalize-media.ts | media.upload and URL hash cache |
| src/server/services/media/media-store.ts | Team and player entity cache |
| src/server/services/media/sync-squad-media.ts | Bulk squad sync lock and skip |
| src/server/services/media/normalize-fixture-media.ts | Fixture lineup image swap |
| src/client/utils/mediaUrl.ts | Client Reddit URL filter |

---

## Permissions

```json
{
  "redis": true,
  "reddit": true,
  "media": true,
  "http": {
    "enable": true,
    "domains": ["api.sportmonks.com", "cdn.sportmonks.com"]
  }
}
```

---
