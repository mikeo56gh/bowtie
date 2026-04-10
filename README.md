# BowtiePro — Vercel Deploy (Phase 1)

COMAH bowtie risk analysis tool for **px Group Saltend Chemicals Park**, deployed as a static frontend with a Vercel serverless proxy for the Anthropic API.

This is **Phase 1** of the upgrade plan: the app is ported to Vercel with the Anthropic API key moved server-side (so it's never exposed in the browser), the font swapped to Nunito Sans, and the palette rebranded to the px Saltend brand colours. Phase 2 will port the whole thing into Next.js + the Vercel AI SDK + Supabase for multi-user, versioning, and streaming AI outputs.

## What changed from the original single-file HTML

1. **API key is now server-side.** The two calls from `index.html` that used to hit `https://api.anthropic.com/v1/messages` directly now go to `/api/claude`, a Vercel serverless function that holds the key in an environment variable. Your key is no longer shipped to the browser.
2. **Nunito Sans** replaces IBM Plex Sans (weights 300/400/600/700/800 via Google Fonts). JetBrains Mono replaces IBM Plex Mono for the monospace numerals on the risk badge and barrier scores.
3. **px Saltend brand palette** applied as CSS variables at the top of `index.html`:
   - Navy `#1F2B7C` (primary / dark surfaces)
   - Saltend Orange `#EA541C` (accent / primary buttons / amber severity)
   - Green `#008A3A` (success / healthy barriers)
   - Light green `#7FC28F` (available for secondary highlights)
4. **Logo** is loaded from `/logo.jpg` (or `logo.svg` / `logo.png` if you swap it). The embedded base64 GIF has been removed, saving ~8 KB from the HTML.
5. **Security headers** added via `vercel.json` (nosniff, frame options, referrer policy, permissions policy).

Nothing about the bowtie logic, risk matrix, ALARP calculations, RRF scoring, or COMAH report structure has been touched. Your data model and localStorage persistence are intact.

## Repo structure

```
bowtiepro-vercel/
├── api/
│   └── claude.js          ← Serverless Anthropic proxy
├── public/
│   ├── index.html         ← BowtiePro app (was BowtiePro_v6__9_.html)
│   └── logo.jpg           ← px Saltend logo
├── .env.example           ← Template for local dev env vars
├── .gitignore
├── package.json
├── vercel.json            ← Routing + security headers
└── README.md              ← This file
```

## Deploying to Vercel (first time)

**You'll need:** a GitHub account, a Vercel account (free tier is fine), and your Anthropic API key.

### Option A — via GitHub (recommended)

1. **Create a new GitHub repo.** On github.com click New → give it a name like `bowtiepro-saltend` → leave it private → Create.

2. **Push this folder to it.** From inside the `bowtiepro-vercel` folder:
   ```bash
   git init
   git add .
   git commit -m "Initial BowtiePro Vercel deploy"
   git branch -M main
   git remote add origin https://github.com/YOUR-USERNAME/bowtiepro-saltend.git
   git push -u origin main
   ```

3. **Import it into Vercel.** Go to vercel.com → Add New → Project → Import Git Repository → pick the repo you just pushed. Vercel will auto-detect it as a static + serverless project. Click **Deploy**. The first deploy will fail because there's no API key yet — that's fine.

4. **Add your API key.** In the Vercel dashboard, go to your project → Settings → Environment Variables. Add:
   - **Name:** `ANTHROPIC_API_KEY`
   - **Value:** your `sk-ant-...` key
   - **Environments:** tick Production, Preview, and Development

5. **Redeploy.** Go to the Deployments tab → click the three dots on the latest deployment → Redeploy. This time it will pick up the key.

6. **Visit your site.** You'll get a URL like `bowtiepro-saltend.vercel.app`. Open it, open the AI assistant panel, and confirm it responds. If it does, the proxy is working and your key is safe.

### Option B — via Vercel CLI (no GitHub)

```bash
npm i -g vercel
cd bowtiepro-vercel
vercel              # follow prompts — first run links/creates the project
vercel env add ANTHROPIC_API_KEY    # paste your key, tick all environments
vercel --prod       # deploy to production
```

## Local development

```bash
npm i -g vercel
cp .env.example .env.local    # paste your key into .env.local
vercel dev                    # runs frontend + /api/claude locally on :3000
```

Open http://localhost:3000 — `vercel dev` simulates the serverless function locally so the AI panel works end-to-end without deploying.

## Swapping the logo

The frontend looks for (in order) `/logo.svg`, `/logo.png`, `/logo.jpg` and uses the first one it finds. To replace the logo, just drop a new file into `/public` with one of those names and redeploy. SVG gives the crispest result on high-DPI displays.

## Tuning the palette

All brand colours are CSS variables at the very top of `public/index.html` inside the `:root { ... }` block. The px-prefixed ones are the raw brand colours; the others (`--accent`, `--blue`, `--green`, `--amber`) are the semantic aliases the rest of the CSS uses. To adjust, edit those and redeploy — no rebuild step required.

```css
--px-navy:#1F2B7C;
--px-green:#008A3A;
--px-green-light:#7FC28F;
--px-orange:#EA541C;
```

## Checking the API key isn't leaking

After deploy, open the site → DevTools → Network tab → trigger an AI action. You should see a POST to `/api/claude` (same origin, no key visible). You should **not** see any request to `api.anthropic.com` from the browser. If you do, something's pointing at the old URL — grep `index.html` for `api.anthropic.com` and it should return zero hits.

## Known limitations (addressed in Phase 2)

- **Still localStorage-only.** Bowties are stored in the user's browser. Clear browser data = lose bowties. No sharing between users. No version history.
- **No streaming.** AI responses appear all at once after Claude finishes generating. Phase 2's Vercel AI SDK integration will stream token-by-token.
- **No auth.** Anyone with the URL can use it. Fine for a private Vercel deployment but not for multi-team rollout.
- **No structured output validation.** The AI bowtie builder relies on prompt-engineering Claude to return parseable JSON. Phase 2 will use Zod schemas + `generateObject` from the Vercel AI SDK for guaranteed-valid shapes.

## Phase 2 preview

The roadmap we discussed:

1. Port to Next.js 15 (App Router) — each panel becomes a React component
2. Vercel AI SDK (`@ai-sdk/anthropic`) with `streamText` and `generateObject` + Zod schemas for the bowtie data model
3. Supabase for auth (Azure AD / px Group SSO if available), persistence, and row-level security
4. Version history — every save becomes an immutable snapshot in Supabase, with a diff view
5. SVG / PDF export of the bowtie diagram for inclusion in safety cases
6. Cross-linking with your ALARP Study Tool, incident investigation generator, and Safety Culture survey
7. Real-time collaboration via Supabase Realtime
8. shadcn/ui command palette (⌘K) for power users
9. Generative UI (RSC) — Claude streams live `<ThreatCard>` / `<BarrierCard>` components into the chat

---

Built for Mike @ Centrica Power / px Group Saltend Chemicals Park.
