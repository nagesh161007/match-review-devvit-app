# Match Review — Judge demo guide

**Match Review Autopilot** runs post-match player ratings on Reddit: rate the lineup, give love, pick Player of the Match, and choose match mood. Votes roll up into **Reddit Player of the Month** and **Reddit Player of the Season**.

You can try it in **two ways**:

---

## Option A — Try our demo subreddit (fastest)

**Subreddit:** [r/match_review_dev](https://www.reddit.com/r/match_review_dev/)

The app is already installed. Jump to **Try as a fan** or **Full demo as a mod** below.

---

## Option B — Install in your own community

Any moderator can install Match Review on a subreddit they mod (test subreddit or your own community).

### Install the app

1. Open a subreddit where you are a **moderator**.
2. Go to **Mod tools → Installed apps** (or **Community apps** / **Add app**, depending on Reddit UI).
3. Search for **Match Review** (or open the app link from the submission / Devvit listing).
4. Click **Install** / **Add to community** and approve permissions (Redis, posts, media, SportMonks HTTP).

If the app is not yet public in the directory, use the **playtest / review link** provided by the team, or ask them to add your subreddit as a playtest community.

### After install — one-time setup

1. **Connect your team**  
   **Mod tools → Community settings → Match Review** → set **Team name** (e.g. `Manchester City`, `Arsenal`) → Save.

2. **Sync photos** (recommended)  
   **Mod tools → Match Review → Sync squad photos** → wait ~1 minute.

3. **SportMonks API key**  
   Set by the **app developer** (global for all installs). Judges using the team’s deployed build do **not** need their own key. If fixture sync fails, contact the submitter.

### Run the demo on your subreddit

Same mod menu as the demo subreddit:

| Action | What it does |
| --- | --- |
| **Schedule 3 match posts** | 3 recent real matches, posts open now / +5 min / +10 min |
| **Schedule award posts** | POTM + POTSeason (after closed matches with real votes) |
| **Sync squad photos** | Player headshots + team logo |
| **Reset app data** | Clear posts and votes for a fresh run |

---

## Try as a fan (~2 minutes)

1. Open a **match rating post** in the feed.
2. Tap players to rate (0–5 stars).
3. Optional: love credits + **Player of the Match** (crown).
4. Tap **Review** → **Happy / Neutral / Sad** → **Confirm** (one vote per account per match).
5. After **15 minutes**, the post shows the **results dashboard** (ratings, goals/assists, Community Mood, MOTM).
6. Tap the **scoreboard** or award **header** for full screen.

---

## Full demo as a mod (~20 minutes)

### 1. Connect team + photos

<img width="1078" height="688" alt="Screenshot 2026-05-27 at 7 40 46 PM" src="https://github.com/user-attachments/assets/0612f1d0-3611-4b83-853e-eb85553006d9" />

See **After install — one-time setup** above (skip if using r/match_review_dev and team is already connected).

### 2. Schedule match posts

<img width="1264" height="442" alt="Screenshot 2026-05-27 at 7 41 17 PM" src="https://github.com/user-attachments/assets/56552924-6a1d-45cf-99f9-9eafc8c19823" />

**Mod tools → Match Review → Schedule 3 match posts**

- Post 1 — **now**
- Post 2 — **+5 min**
- Post 3 — **+10 min**

Open each post and **vote** (real votes only — no mock data).

### 3. Wait for close

Each match: **15 min** voting, then auto-close → dashboard in the same post.

### 4. Schedule awards

After **≥1 closed match with a vote**:

**Mod tools → Match Review → Schedule award posts**

Publishes immediately:

- **Reddit Player of the Month**
- **Reddit Player of the Season**

**Reset:** **Reset app data** to start over.

---

## Timing

| Event | When |
| --- | --- |
| Match post 1 | Immediately |
| Match post 2 | +5 min |
| Match post 3 | +10 min |
| Voting window | 15 min per match |
| Award posts | When you click **Schedule award posts** |

**Quick path:** Vote on a live post → wait for close → **Schedule award posts**.

---

## What to look for

- Inline voting in the feed
- Live lineup + formation from SportMonks
- **Community Mood** on closed-match dashboard
- Award posts: highest rated, most loved, loyal fan (streak + profile link)
- Works on **any** installed community — same app, your team name

---

## Troubleshooting

| Issue | Fix |
| --- | --- |
| Can’t find app to install | Use submission link / Devvit listing; or use r/match_review_dev |
| Awards won’t publish | Match **closed** + **≥1 real vote** |
| No fixtures | Check **Team name** spelling; confirm developer set SportMonks API key |
| No player photos | **Sync squad photos** |
| Wrong team | **Reset app data** → set new **Team name** |
