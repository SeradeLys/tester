# Wolfie Butler (NutriPlan) — code export

Stony Brook dining-hall meal planner: Nutrislice menus, nutrition estimates, meal planner with a full-day plan, favourites, recipes with Medal clip links, and the Wolfie Butler chat.

## Files
- `index.html` — standalone page. Open it in any browser or host it on any static host (GitHub Pages, Netlify, Vercel).
- `wolfie-butler-artifact.html` — the exact source published as the claude.ai artifact (no doctype/head; claude.ai adds those).
- `data/west--2026-10-23.json`, `data/east--2026-10-23.json` — Oct 23, 2026 menus parsed from the Nutrislice API, in the app's menu format.
- `data/recipes.json` — the six starter recipes.

## Running outside claude.ai
Everything runs in one file, with no build step. Outside claude.ai, the claude.ai-only features fall back automatically:
- **Cloud database** (shared menus/recipes, private profile/plan/favourites) → saved in the browser's localStorage instead. Use **Import menu** to load Nutrislice JSON.
- **Wolfie Butler chat** runs on the viewer's Claude account through the claude.ai runtime, so it's unavailable outside claude.ai. That's the slot to connect your own backend (Gemini, Photon) — see `AgentService.send` in the script.
- **Photo/icon uploads** need the claude.ai assets store. Set image paths directly in `CONFIG` instead.

## Where to change things
At the top of the `<script>`, in `CONFIG`:
- `hallImages` — West/East dining hall image paths.
- `chatIcon` — chatbot icon path.
- `activity`, `carbPct`, `fatPerKg`, `fiberRef`, `safety`, `dayShares`, `calTolerance` — nutrition and planning settings.
- `healthResources` — health links.

Code layout inside the script: `NutritionService` (BMR/TDEE/targets, meal totals, goal safety) · `MenuProvider` (Nutrislice parser) · `MealPlanService` · `DayPlanService` (±50 kcal day plan) · `RecommendationService` · `MedalProvider` · `Store` (persistence) · views and event handlers.

## Nutrislice API
`https://stonybrook.api.nutrislice.com/menu/api/weeks/school/{west-side-dining|east-side-dining}/menu-type/{todays-dine-in-specials-wsd|todays-dine-in-specials-esd}/YYYY/MM/DD/?format=json`
A browser page can't fetch this directly (cross-origin), so either paste the JSON into **Import menu** or fetch it from a small server.

Nutrition values are estimates, not medical advice.

## Wolfie Butler AI agent (backend in `agent/`)
The chat box talks to a real backend: the Gemini + Photon agent in `agent/`. Its web endpoint is `POST /chat`
(`{ text, sessionId, page }` → `{ reply }`). The site's `CONFIG.agentUrl` (default `http://localhost:8787/chat`) points to it.
The same agent also answers iMessage through Photon. Secrets live only in `agent/.env` (never in the website).

### Run it locally (two terminals)
```bash
# 1) Agent  (needs Bun: https://bun.sh)
cd agent
cp .env.example .env     # then fill in the values (see below)
bun install
bun run start            # → http://localhost:8787/chat

# 2) Website  (must be served over http; file:// won't work)
cd ..                    # project root
python3 -m http.server 5173     # or: npx serve -l 5173 .
# open http://localhost:5173
```
Required in `agent/.env`: `GEMINI_API_KEY`, `PHOTON_API_KEY`, `PHOTON_PROJECT_ID`, `PHOTON_PLATFORM`, `PHOTON_AGENT_HANDLE`, `PHOTON_TEST_SENDER`.
(The agent refuses to start without the Photon values even if you only use the website chat.)

### Deploying the site (e.g. Vercel)
The agent must run somewhere reachable over https (a tunnel such as `cloudflared`/`ngrok`, or a server). Then:
1. set `agentUrl` in `CONFIG` in `index.html` to that address + `/chat`;
2. set `WEB_CHAT_ORIGINS` in `agent/.env` to your site's address (default allows `https://sbuhacks26.vercel.app`; localhost is always allowed).
`.vercelignore` keeps `agent/` out of the static deploy.

### Tests
`cd agent && bun test` (29 nutrition unit tests) · `bun check` (type check).

### Known limits
- One shared profile/food log (`agent/nutrition-data.json`) for iMessage and every web visitor: fine for a demo/single user, not multi-user.
- The agent logs meals in its own log; it can't press the site's "add to meal plan" button (that tool only exists in the claude.ai build).
