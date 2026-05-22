# WeatherNext — Company Test-Flight Deployment Guide

This kit turns the generic WeatherNext PWA into a **branded, per-company
copy**: own icons, own app name, own fixed farm list, own GitHub Pages
folder. Your API key stays embedded for the test-flight; companies switch
to their own key later.

---

## What's in this kit

```
wnext-build/
├── base-index.html        ← your uploaded app (untouched master)
├── base-sw.js             ← your uploaded service worker (master)
├── base-manifest.json     ← your uploaded manifest (master)
├── *.png                  ← your original base icons (fallback)
├── build_company.py       ← the generator
├── make_icons.py          ← optional icon generator
├── companies/
│   ├── _TEMPLATE.json     ← copy this per company
│   └── greenvalley-farms.json  ← worked example
├── icons/
│   └── example-greenvalley/    ← example icon set
└── dist/
    └── greenvalley-farms/      ← a finished, deployable build
```

The `base-*` files are **never modified**. Every build is copy-then-patch,
so you can re-run a build any time and regenerate cleanly.

---

## The 60-second workflow for a new company

### 1. Write the company config
Copy `companies/_TEMPLATE.json` → `companies/<slug>.json` and fill it in.
The only things you must decide:

| Field            | What it controls                                          |
|------------------|-----------------------------------------------------------|
| `slug`           | GitHub Pages folder name (lowercase, hyphens). **See §API key warning.** |
| `company_name`   | Shown in app header / title / share text. Chinese OK.     |
| `app_short_name` | Label under the home-screen icon.                         |
| `lock_locations` | `true` = hide the "+ Add" button, staff see ONLY company farms. `false` = staff can still add their own. |
| `icons_dir`      | Folder with the 4 brand PNGs.                             |
| `locations`      | The company's farms — `id`, `en`/`zh` name, `lat`, `lng`. |

### 2. Prepare the icons
Either:
- **Company supplies brand icons** → drop 4 PNGs into the `icons_dir`
  folder, named exactly: `favicon-32.png`, `icon-192.png`,
  `icon-512.png`, `apple-touch-icon.png`.
- **No brand icons yet** → generate a distinctive set:
  ```
  python3 make_icons.py icons/<company> --sky 86efac --field 15803d --sun f59e0b
  ```
  Pick different colours per company so they're easy to tell apart on a
  phone home screen. (If you skip this and `icons_dir` files are missing,
  the build falls back to the base WeatherNext icon — fine for a first
  test, but every company looks identical.)

### 3. Build
```
python3 build_company.py companies/<slug>.json
```
Output: `dist/<slug>/` — a complete folder ready to deploy.

### 4. Deploy to GitHub Pages
Push the contents of `dist/<slug>/` to a repo (or a subfolder of an
existing Pages repo) so it serves at:
```
<your-username>.github.io/<slug>/
```

### 5. ⚠️ Authorise the API key — DO NOT SKIP
See the next section.

---

## ⚠️ API key referrer restriction — the one thing that will break

Your `DEFAULT_API_KEY` is **HTTP-referrer restricted** in Google Cloud to
exactly two paths today:

```
stanleywoosweeleong.github.io/weathernextpersonal/*
stanleywoosweeleong.github.io/weatherforleesooihon/*
```

A new company at `…github.io/greenvalley-farms/*` is **not on that list**,
so Google will **reject every AI request** with a referrer error. The
weather forecast itself (Open-Meteo) still works — but the AI briefing,
pest analysis, work-plan, etc. will fail.

**Two ways to fix it — pick one:**

**Option A — Add each company path to the key (recommended for test-flight)**
1. Google Cloud Console → *APIs & Services* → *Credentials*
2. Open the API key → *Application restrictions* → *Website restrictions*
3. Add `<your-username>.github.io/<slug>/*` for each new company
4. Save. Takes a few minutes to propagate.

**Option B — Host all companies under one already-approved path**
e.g. serve every company as a subfolder of `weathernextpersonal`:
`…/weathernextpersonal/greenvalley-farms/`. Then the existing wildcard
already covers them and you change nothing in Cloud Console. The trade-off
is a slightly less clean URL.

**When a company moves to their own key:** they just open the in-app API
Key modal and paste theirs. Their key (stored in `localStorage`) overrides
the embedded default automatically — no rebuild needed. Remind them their
own key will need *its own* referrer restriction set to *their* domain.

---

## How the build differs from the generic app

| Aspect            | Generic build              | Company build                          |
|-------------------|----------------------------|-----------------------------------------|
| App name          | User types it in onboarding| Fixed at build time, no onboarding      |
| Onboarding modal  | Shows on first launch      | Disabled                                |
| Locations         | Empty, user adds farms     | Company farms **seeded on first launch** |
| Deleting farms    | Fully deletable            | **Fully deletable** (soft + permanent)  |
| Editing farms     | Full name + GPS edit       | **Full name + GPS edit**                |
| Adding new farms  | Yes                        | **Yes** — "+ Add" stays available       |
| Receiving shares  | Yes (deep-link)            | **Yes** — works natively                |
| "+ Add" button    | Always visible             | Hidden only if `lock_locations: true`   |
| Icons / manifest  | Generic WeatherNext        | Company-branded                         |
| API key           | Embedded default           | Same embedded default (test-flight)     |

### How the location seed works

The company's farms are **not** baked into the app code. Instead, a one-time
seed writes them into the browser's `localStorage` on the **very first
launch**, as genuine custom locations (ids prefixed `c_`). From that point
the user has total control:

- **Delete** a farm (swipe → recycle bin, or permanent purge) and it stays
  deleted — the seed never resurrects it.
- **Edit** name or GPS coordinates freely.
- **Add** brand-new farms via the "+ Add" button.
- **Receive** locations shared by other users running the personalizable
  build — they merge in alongside the seeded farms.

The seed is guarded by a `SEED_VERSION` flag. To push a *new* farm set to
existing users in a future build, bump `seed_version` in the company config
(it defaults to `"1"`). Leave it alone and the seed never runs twice.

This means the company build is the **full generic app** in every respect —
it just arrives pre-populated with the company's farms so they see their
forecast immediately on first open, with zero setup.

---

## Re-running / updating a company

When you ship a new version of the base app:
1. Replace `base-index.html` / `base-sw.js` / `base-manifest.json` with the
   new versions.
2. Re-run `python3 build_company.py companies/<slug>.json` for each company.
3. Re-deploy each `dist/<slug>/`.

The service worker `CACHE_VERSION` is auto-bumped on every build (it
includes a timestamp), so users' devices pick up the update on next launch.

---

## Quick reference — build all companies at once

```bash
for cfg in companies/*.json; do
  [ "$(basename "$cfg")" = "_TEMPLATE.json" ] && continue
  python3 build_company.py "$cfg"
done
```
