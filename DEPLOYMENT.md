# RefBuddy — Deployment Guide (v3.2)

Target stack: **GitHub → Render → Cloudflare DNS → refbuddy.ai**

---

## Repo layout

```
refbuddy/
├── app.py                          ← the whole application (single file)
├── requirements.txt                ← Python dependencies
├── .gitignore                      ← keeps secrets & rulebook PDFs out of git
├── DEPLOYMENT.md                   ← this file
└── .streamlit/
    ├── secrets.toml.template       ← committed; a blank example
    └── secrets.toml                ← LOCAL ONLY, gitignored, never pushed
```

**Never commit:** `secrets.toml`, `.env`, any NFHS/MSHSL PDF, any game film.

---

## Step 1 — Run locally

```bash
cd ~/RefBuddy\ Football

# Anaconda's base env interferes with the 3.14 venv — deactivate first
conda deactivate

source venv314/bin/activate
pip install -r requirements.txt

streamlit run app.py
```

Opens at http://localhost:8501

### Local secrets

```bash
cp .streamlit/secrets.toml.template .streamlit/secrets.toml
```

Then edit `.streamlit/secrets.toml`:

```toml
ANTHROPIC_API_KEY = "sk-ant-api03-your-real-key"
APP_PASSWORD      = "your-crew-password"
```

If you **omit** `APP_PASSWORD`, the login gate is skipped entirely — convenient
locally, but it means anyone who reaches the URL can spend your API credit.
Always set it in production.

---

## Step 2 — Push to GitHub

```bash
git init
git add app.py requirements.txt .gitignore DEPLOYMENT.md .streamlit/secrets.toml.template
git commit -m "RefBuddy v3.2 — password gate, usage cap, legal disclaimer"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/refbuddy.git
git push -u origin main
```

Before pushing, confirm secrets aren't staged:

```bash
git status --short | grep -i secret     # should return nothing
```

---

## Step 3 — Deploy on Render

1. **render.com → New → Web Service** → connect the GitHub repo
2. Settings:

| Field | Value |
|-------|-------|
| Environment | Python 3 |
| Build command | `pip install -r requirements.txt` |
| Start command | `streamlit run app.py --server.port $PORT --server.address 0.0.0.0 --server.headless true` |
| Instance type | Starter ($7/mo) — see note below |
| Health check path | `/_stcore/health` |

3. **Environment → Environment Variables**, add both:

```
ANTHROPIC_API_KEY = sk-ant-api03-your-real-key
APP_PASSWORD      = your-crew-password
```

4. Deploy. You get `https://refbuddy-xxxx.onrender.com`. Confirm the password
   screen appears before doing anything else.

### How the app finds these values

`st.secrets` does **not** read arbitrary environment variables — it only reads a
`secrets.toml` file on disk. Since there is no such file on Render (it's
gitignored), the app uses a `get_secret()` helper that checks, in order:

1. `os.environ` — how Render, Docker, and most hosts supply config
2. `secrets.toml` — local development and Streamlit Community Cloud
3. a legacy nested `[anthropic]` table

Every `st.secrets` access is wrapped in a broad `except Exception`, because when
no `secrets.toml` exists anywhere, simply touching `st.secrets` raises
`StreamlitSecretNotFoundError` — which is not a `KeyError` and will crash the
app on boot if caught too narrowly.

**Fail-closed safety:** if `APP_PASSWORD` is missing *and* the app detects it's
running on Render (via Render's own `RENDER=true` variable), it refuses to start
and shows an error instead of serving an unprotected app. Locally, a missing
password just skips the gate.

### Free vs Starter

Free spins down after ~15 minutes idle, so the first visitor waits ~50s for a
cold start. On a Friday night with a crew checking a rule between series, that's
the difference between "useful" and "abandoned." Starter stays warm.

---

## Step 4 — Point refbuddy.ai at it

### 4a. Add the domain to Cloudflare

1. Cloudflare → Add a site → `refbuddy.ai` → Free plan
2. Cloudflare scans existing DNS, then gives you **two nameservers**
   (e.g. `ada.ns.cloudflare.com`, `rob.ns.cloudflare.com`)

### 4b. Repoint nameservers at Namecheap

Namecheap → Domain List → Manage `refbuddy.ai` → **Nameservers** →
change from "Namecheap BasicDNS" to **Custom DNS** → paste both Cloudflare
nameservers → save (green checkmark).

Propagation is usually 10–60 minutes. Cloudflare emails you when it's active.

### 4c. Add the custom domain in Render

Render → your service → **Settings → Custom Domains** → add:
- `refbuddy.ai`
- `www.refbuddy.ai`

Render shows you a target hostname to point at.

### 4d. Create the DNS records in Cloudflare

Cloudflare → DNS → Records:

| Type | Name | Target | Proxy |
|------|------|--------|-------|
| CNAME | `www` | `refbuddy-xxxx.onrender.com` | Proxied (orange) |
| CNAME | `@` | `refbuddy-xxxx.onrender.com` | Proxied (orange) |

Cloudflare flattens the root CNAME automatically — no A record needed.

### 4e. Cloudflare settings that matter

These are the ones that break Streamlit if you get them wrong:

- **SSL/TLS → Overview → Full (strict).** "Flexible" causes an infinite redirect
  loop. This is the single most common failure.
- **SSL/TLS → Edge Certificates → Always Use HTTPS: On**
- **Network → WebSockets: On** (default on, but verify — Streamlit is
  WebSocket-based and simply won't load without it)
- **Speed → Optimization → Rocket Loader: Off.** It rewrites JS execution order
  and breaks Streamlit's frontend.
- **Caching → Configuration → Caching Level: Standard.** Don't put aggressive
  page rules over the app; Streamlit is dynamic.

Wait for Render to issue the TLS cert (a few minutes), then load
`https://refbuddy.ai`. You should see the password screen.

---

## Step 5 — Cost controls (do this before sharing the URL)

Three layers, all worth setting:

1. **App-level password** — `APP_PASSWORD` env var, covered above.
2. **Per-session frame cap** — `MAX_FRAMES_PER_SESSION = 400` near the top of
   `app.py`. Every film analysis charges against it; the sidebar shows what's
   left. Lower it to 200 if you want to be conservative.
3. **Anthropic account spend limit** — the real backstop. See below.

### Setting your Anthropic spend limit

1. Go to https://platform.claude.com/settings/billing
2. Find the **Spend limits** section
3. Click **Adjust limit** (or **Set limit** if none is set)
4. Enter a monthly ceiling you're comfortable with — e.g. `$50`
5. Save

Once that's hit, API usage pauses until the next calendar month. It cannot be
exceeded. This is your hard stop against a runaway bill.

### Checking your tier and rate limits

1. https://platform.claude.com/settings/limits — your current tier and
   per-model RPM / ITPM / OTPM
2. https://platform.claude.com/usage — charts of actual usage against those
   limits, plus your cache hit rate

Tiers run Evaluation → Start → Build → Scale → Custom, and advance automatically
as you build usage history. Monthly spend caps by tier: Start $500, Build $1,000,
Scale $200,000.

At the **Start** tier, `claude-sonnet-4-6` gets 1,000 requests/min, 2,000,000
input tokens/min, and 400,000 output tokens/min. For a crew-sized audience you
will not come close to throttling — spend is the thing to watch, not rate limits.

---

## Step 6 — Updating CORE_KNOWLEDGE

`CORE_KNOWLEDGE` is a Python string literal starting around **line 57 of app.py**.
It is baked into the deployed file. It is *not* connected to your Claude Project
knowledge base — uploading a PDF to the Project does not change the deployed app.
Editing that string and pushing is the only way to update what RefBuddy knows.

### Add a game note

Find `## 5. PERSONAL GAME NOTES` and add a line:

```
**Holding on screen (MSHSL Year 5 — Edina 9/12/26)**: Receiver extended arms to
push off DB before ball arrived = OPI. Contact continuing as ball arrived = still OPI.
```

### Add a new season's rule changes

Find `## 0. 2022–2026 NFHS RULES CHANGES & UPDATES` and add a dated block in the
same format (rule number, year, old, new, why it matters), then add a row to the
Quick-Reference table at the bottom of that section.

### Update MSHSL modifications

Find `## 2. MSHSL MINNESOTA-SPECIFIC MODIFICATIONS`. MSHSL typically posts the
new Minnesota Rules Modifications PDF in late August / early September at
https://www.mshsl.org/sports-and-activities/football

### Deploy the update

```bash
git add app.py
git commit -m "Update CORE_KNOWLEDGE — 2026 MSHSL modifications"
git push
```

Render auto-deploys on push. Watch the deploy log; ~2 minutes.

---

## Troubleshooting

**Infinite redirect loop at refbuddy.ai**
Cloudflare SSL/TLS mode is set to Flexible. Change to **Full (strict)**.

**Page loads but spins forever / "Please wait…"**
WebSockets are off in Cloudflare, or Rocket Loader is on. Check
Network → WebSockets (On) and Speed → Rocket Loader (Off).

**"ANTHROPIC_API_KEY not found in secrets"**
The env var isn't set on Render, or has a typo. Render → Environment. Names are
case-sensitive.

**`StreamlitSecretNotFoundError: No secrets found` on boot**
Fixed in v3.4. If you see it, you're running an older app.py — pull the latest.
The cause: code that read `st.secrets[...]` directly and caught only `KeyError`.
With no `secrets.toml` on the host, Streamlit raises `StreamlitSecretNotFoundError`
instead, which slips past that handler and crashes the app.

**Password screen won't accept the password**
Check for a trailing space in the Render env var value — Render preserves
whitespace exactly.

**"Session analysis limit reached"**
Working as designed — 400 frames per session. Refresh for a new session, or raise
`MAX_FRAMES_PER_SESSION` in app.py.

**PDF export fails**
`fpdf2` must be in requirements.txt. The app calls `bytes(pdf.output())` to
normalize the bytearray some fpdf2 versions return.

**Render deploy fails on opencv**
`requirements.txt` must specify `opencv-python-headless`, not `opencv-python`.
The non-headless build needs GUI libraries Render doesn't have.

---

## Cost reference

`claude-sonnet-4-6` is billed per million tokens, input and output separately.
Check https://platform.claude.com/settings/billing for current rates.

Rough shape of RefBuddy's usage:

| Operation | Approximate input tokens |
|-----------|--------------------------|
| System prompt (CORE_KNOWLEDGE) | ~9,000 per call |
| Chat question | ~9,000 + conversation history |
| Quiz question | ~9,000 |
| Film analysis, 30 frames | ~55,000 |
| Crew Eval, 40 frames | ~70,000 |

Film analysis is 6–8× the cost of everything else. That's why the frame cap
exists, and why it's the number to tune.

**Optimization worth doing later:** the CORE_KNOWLEDGE system prompt is identical
on every call. Adding a prompt-caching breakpoint to it would cut repeat input
cost by ~90%, and on most models cached reads don't count toward your
input-tokens-per-minute limit either. See
https://platform.claude.com/docs/en/build-with-claude/prompt-caching

---

## Privacy & IP posture

- No conversations or uploads are persisted server-side; everything lives in
  session memory and is discarded when the session ends.
- Uploads are transmitted to the Anthropic API for processing. Don't upload
  anything containing sensitive personal information about students.
- The system prompt instructs the model never to reproduce verbatim NFHS/MSHSL
  text — it summarizes and cites rule numbers instead.
- Source rulebook PDFs are excluded by `.gitignore` and must never be committed
  or served.
- The Terms of Use expander in the footer states non-affiliation with NFHS and
  MSHSL, disclaims warranty, and tells users to own a current rulebook.
