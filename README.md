# 🌸 User-Pays AI on itch.io — Pollinations BYOP Integration Guide

> A practical, battle-tested guide to letting players pay for their own AI compute in your itch.io browser game using the [Pollinations](https://pollinations.ai) BYOP (Bring Your Own Pollen) system.
>
> **Built from real trial and error making [Mochi — The Cozy Cat](https://k4l4m.itch.io/mochi-cat), March 2026.**

---

## What is BYOP?

Pollinations lets you issue users a temporary API key scoped to your app. Their [Pollen balance](https://enter.pollinations.ai) gets charged for AI calls — not yours. You pay nothing centrally.

| | |
|---|---|
| **Cost to you** | Zero. Users pay from their own Pollen ($1 ≈ 1 Pollen). |
| **Key lifetime** | 30 days by default, configurable. User can revoke anytime. |
| **Key type** | Temporary `sk_` key — never touches your server. |
| **Free models** | `flux` image generation is always free — no Pollen cost. |
| **Paid models** | `openai` chat ≈ 0.001 Pollen per call. |

---

## The itch.io Problem (and the Fix)

The standard BYOP flow redirects the user to a callback URL with the API key in the URL fragment (`#api_key=sk_...`). This breaks on itch.io for two reasons:

1. **GitHub login refuses to load inside an iframe** — itch.io sandboxes your game in an `<iframe>`, and OAuth providers block this.
2. **itch.io strips the URL fragment** before the page's JavaScript runs, so even pointing the redirect back at an itch.io page doesn't work.

### ✅ The Solution: GitHub Pages as the callback

```
[itch.io game] → opens new tab → [Pollinations auth] → redirects to → [GitHub Pages callback]
                                                                              ↓
                                                              user copies key, pastes into game
```

GitHub Pages loads in a real browser tab (not an iframe), so it reads the `#api_key=` fragment correctly. The user copies the key and pastes it into a text field in your game.

> **Why not BroadcastChannel or postMessage?** itch.io runs games on `html-classic.itch.zone`, a different origin from `k4l4m.itch.io`. BroadcastChannel and cross-origin postMessage are both blocked by the browser's same-origin policy in this context.

---

## Architecture

```
┌─────────────────────────┐     ┌──────────────────────────┐     ┌──────────────────────────────┐
│  🎮 Game                │     │  🔐 Pollinations Auth     │     │  📄 Callback Page            │
│  yourname.itch.io/game  │────▶│  enter.pollinations.ai   │────▶│  yourname.github.io/auth     │
│                         │     │  /authorize              │     │                              │
│  Step 1: Sign in button │     │  User logs in via GitHub │     │  Reads #api_key= from URL    │
│  Step 2: Paste key box  │◀────│  Redirects to callback   │     │  Shows Copy button           │
└─────────────────────────┘     └──────────────────────────┘     └──────────────────────────────┘
        copy-paste
```

---

## Step-by-Step Setup

### Step 1 — Create your itch.io project

1. Create a new project on [itch.io](https://itch.io)
2. Set **Kind** to `HTML`
3. Upload your game as a ZIP containing `index.html` at the root (not inside a folder)
4. Tick **"This file will be played in the browser"**
5. Note your game URL (e.g. `yourname.itch.io/my-game`)

---

### Step 2 — Create the GitHub Pages callback page

1. Create a new **public** GitHub repository (e.g. `mochi-auth`)
2. Upload the [callback page template](#the-callback-page) renamed as `index.html`
3. Go to repo **Settings → Pages → Branch: main → Save**
4. Wait ~2 minutes (check the **Actions** tab for a green tick)
5. Your callback URL is: `https://YOUR-USERNAME.github.io/REPO-NAME`

> ⚠️ The callback **must** be on GitHub Pages (or similar). Do not use an itch.io page as the `redirect_url` — itch.io strips the URL fragment.

---

### Step 3 — Add auth to your game

```js
const AUTH_REDIRECT = 'https://yourname.github.io/your-auth-repo';
const STORAGE_KEY   = 'my_game_pollinations_key';

function connectPollinations() {
  const authUrl = 'https://enter.pollinations.ai/authorize'
    + '?redirect_url=' + encodeURIComponent(AUTH_REDIRECT)
    + '&models=flux,openai'       // restrict to only models your game needs
    + '&expiry=30'                // key lifetime in days
    + '&permissions=profile,balance'; // needed to show balance in-game

  window.open(authUrl, '_blank'); // ← new tab, NOT window.location.href
}

function activateKey(key) {
  key = key.trim();
  if (!key.startsWith('sk_') && !key.startsWith('pk_')) return;
  try { localStorage.setItem(STORAGE_KEY, key); } catch(e) {}
  // hide login screen, start the game
}

function initAuth() {
  // Check for returning user
  try {
    const stored = localStorage.getItem(STORAGE_KEY);
    if (stored) { activateKey(stored); return; }
  } catch(e) {}
  // No key — show login screen
}
```

**Design your login screen as two explicit steps:**

```
┌────────────────────────────────────────┐
│  STEP 1                                │
│  [🐾 Sign in with Pollinations]        │  ← window.open() to auth
│                                        │
│  STEP 2 — Paste your key               │
│  [sk_..............................]   │  ← visible immediately, not after timeout
│  [Let's go →]                          │
└────────────────────────────────────────┘
```

> Show the paste box immediately — don't wait for an automatic delivery that will never come.

---

### Step 4 — Make API calls

```js
// Chat completion (uses Pollen — ~0.001 per call)
const res = await fetch('https://gen.pollinations.ai/v1/chat/completions', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer ' + apiKey
  },
  body: JSON.stringify({
    model: 'openai',
    messages: [{ role: 'user', content: 'Hello!' }],
    max_tokens: 60,
    temperature: 1.0
  })
});
const data = await res.json();
const text = data.choices[0].message.content;

// Image generation (flux is free — no Pollen cost)
const prompt = encodeURIComponent('a fluffy orange cat, cozy living room');
const imgUrl = `https://gen.pollinations.ai/image/${prompt}?model=flux&seed=42&width=512&height=512`;
const imgRes = await fetch(imgUrl, {
  headers: { 'Authorization': 'Bearer ' + apiKey }
});
const blob = await imgRes.blob();
document.getElementById('myImg').src = URL.createObjectURL(blob);

// Check Pollen balance
const balRes = await fetch('https://gen.pollinations.ai/account/balance', {
  headers: { 'Authorization': 'Bearer ' + apiKey }
});
const { balance } = await balRes.json();
// display balance — link top-up to https://enter.pollinations.ai
```

> Handle `402` errors gracefully — this means the user's Pollen balance is empty. Show a friendly message with a link to [enter.pollinations.ai](https://enter.pollinations.ai) to top up.

---

### Step 5 — Upload to itch.io

1. Wrap your `index.html` in a ZIP: `zip game.zip index.html`
2. Delete any existing upload on your itch.io project page
3. Upload the new ZIP
4. Tick **"This file will be played in the browser"**
5. Save and publish

---

## The Callback Page

Save this as `index.html` in your GitHub Pages repo.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Connecting...</title>
<style>
  body { font-family: Georgia, serif; background: #faf6f0; color: #3d2314;
    display: flex; flex-direction: column; align-items: center;
    justify-content: center; min-height: 100vh; gap: 20px; text-align: center; padding: 32px; }
  .key-box { font-family: monospace; font-size: 0.85rem; background: #e8d5b7;
    padding: 12px 18px; border-radius: 10px; word-break: break-all;
    max-width: 420px; width: 100%; border: 1.5px solid #c9956a; user-select: all; }
  button { padding: 10px 22px; background: #9b5c3a; color: white; border: none;
    border-radius: 10px; font-family: monospace; cursor: pointer; }
  .instructions { background: white; border: 1.5px solid #e8d5b7; border-radius: 12px;
    padding: 16px 20px; max-width: 420px; font-family: monospace; font-size: 0.75rem;
    color: #9b5c3a; line-height: 1.9; text-align: left; }
</style>
</head>
<body>
<div id="catEmoji" style="font-size:3rem">🐱</div>
<h2 id="msg" style="font-style:italic;color:#9b5c3a">Reading your key...</h2>

<div id="success" style="display:none;width:100%;max-width:420px;
  display:none;flex-direction:column;align-items:center;gap:14px">
  <p style="font-family:monospace;font-size:0.8rem;color:#c9956a;margin:0">
    Your key — click to copy, then paste into the game:
  </p>
  <div class="key-box" id="keyBox" onclick="copyKey()" title="Click to copy"></div>
  <button onclick="copyKey()" id="copyBtn">📋 Copy Key</button>
  <div class="instructions">
    <strong>How to connect:</strong><br>
    1. Copy the key above<br>
    2. Switch back to the game tab<br>
    3. Paste it into the <strong>Step 2</strong> box<br>
    4. Click <strong>Let's go →</strong>
  </div>
</div>

<p id="errorMsg" style="display:none;font-family:monospace;font-size:0.8rem;color:#c0392b">
  No key found — did you cancel? Close this tab and try again.
</p>

<script>
(function() {
  // Parse from full href — itch.io strips window.location.hash but
  // GitHub Pages does not, so this works correctly here
  const href     = window.location.href;
  const hashIdx  = href.indexOf('#');
  const fragment = hashIdx !== -1 ? href.slice(hashIdx + 1) : '';
  const key      = new URLSearchParams(fragment).get('api_key');

  if (key) {
    document.getElementById('catEmoji').textContent = '😻';
    document.getElementById('msg').textContent = "You're connected! Copy your key below.";
    document.getElementById('keyBox').textContent = key;
    const s = document.getElementById('success');
    s.style.display = 'flex';

    // Bonus: try BroadcastChannel in case game is open in same browser/origin
    try { new BroadcastChannel('pollinations_auth')
      .postMessage({ type: 'POLLINATIONS_KEY', key }); } catch(e) {}
  } else {
    document.getElementById('catEmoji').textContent = '😿';
    document.getElementById('msg').textContent = 'No key found.';
    document.getElementById('errorMsg').style.display = 'block';
  }
})();

function copyKey() {
  const key = document.getElementById('keyBox').textContent;
  navigator.clipboard.writeText(key).then(() => {
    const btn = document.getElementById('copyBtn');
    btn.textContent = '✓ Copied!';
    btn.style.background = '#8aad8a';
    setTimeout(() => { btn.textContent = '📋 Copy Key'; btn.style.background = '#9b5c3a'; }, 2000);
  });
}
</script>
</body>
</html>
```

---

## Authorize URL Parameters

```
https://enter.pollinations.ai/authorize
  ?redirect_url=YOUR_GITHUB_PAGES_URL   ← required
  &models=flux,openai                   ← restrict allowed models
  &expiry=30                            ← key lifetime in days
  &permissions=profile,balance          ← enable balance API
  &budget=10                            ← max Pollen spend per key
```

| Parameter | Example | Notes |
|---|---|---|
| `redirect_url` | `https://you.github.io/auth` | Must be GitHub Pages or similar. **Not itch.io.** |
| `models` | `flux,openai` | Restricts the key. Protects users from unexpected spend on paid models. |
| `expiry` | `30` | Key lifetime in days. Default 30. |
| `permissions` | `profile,balance` | Required to read balance via `/account/balance`. |
| `budget` | `10` | Hard spending cap on the key. |

---

## Showing the Pollen Balance

```js
async function fetchPollenBalance(apiKey) {
  const res = await fetch('https://gen.pollinations.ai/account/balance', {
    headers: { 'Authorization': 'Bearer ' + apiKey }
  });
  if (!res.ok) return null;
  const data = await res.json();
  // Try known field names — the response shape may vary
  return data?.balance ?? data?.pollen ?? data?.total ?? null;
}
```

> **Important:** `&permissions=profile,balance` must be in your authorize URL or the balance endpoint returns 403. If a user connected before you added this, they need to disconnect and reconnect.

Link your top-up button to **[enter.pollinations.ai](https://enter.pollinations.ai)**.

---

## Do/Don't

| ✅ Do | ❌ Don't |
|---|---|
| Use `window.open(url, '_blank')` to open auth | Redirect the game iframe with `window.location.href` |
| Point `redirect_url` at GitHub Pages | Use an itch.io page as `redirect_url` |
| Show the paste box immediately as Step 2 | Expect the key to arrive automatically via postMessage |
| Save key to `localStorage` for returning users | Ask users to copy from the browser URL bar themselves |
| Scope `models` to only what your game needs | Leave models unrestricted (users could accidentally spend on expensive models) |
| Add a Disconnect button that clears `localStorage` | Hardcode your own API key in the game frontend |

---

## Troubleshooting

**Auth page shows "No key found" but the key is visible in the URL bar**
→ The page is loading inside an itch.io iframe. The `redirect_url` must point to GitHub Pages, not itch.io.

**"GitHub refused to connect" / blank white screen**
→ The auth URL is being opened inside the game iframe. Use `window.open(url, '_blank')`.

**Balance shows `—` or returns 403**
→ The key was issued without balance permission. Add `&permissions=profile,balance` to your authorize URL and have users reconnect.

**GitHub Pages shows 404**
→ Either the file isn't named `index.html`, or Pages hasn't finished deploying. Check the **Actions** tab for a green tick.

**itch.io shows "Failed to find index.html"**
→ The ZIP must contain `index.html` at the root (not inside a subfolder). Also ensure **"This file will be played in the browser"** is ticked.

**AI responses are identical every time**
→ Rotate the user message each call and set `temperature: 1.0` or higher. A fixed prompt with a deterministic seed produces near-identical responses.

**402 error on API calls**
→ User's Pollen balance is empty. Show a top-up message linking to [enter.pollinations.ai](https://enter.pollinations.ai).

---

## Verified API Endpoints (March 2026)

| Endpoint | Purpose |
|---|---|
| `POST https://gen.pollinations.ai/v1/chat/completions` | Chat (OpenAI-compatible) |
| `GET https://gen.pollinations.ai/image/{prompt}` | Image generation |
| `GET https://gen.pollinations.ai/account/balance` | Pollen balance |
| `GET https://enter.pollinations.ai/authorize` | BYOP OAuth flow |
| `GET https://enter.pollinations.ai` | Dashboard / top up |

---

## Useful Links

- 📖 [Pollinations API docs](https://gen.pollinations.ai/api/docs)
- 🔑 [Get API keys & top up Pollen](https://enter.pollinations.ai)
- 🐙 [Pollinations GitHub](https://github.com/pollinations/pollinations)
- 🎮 [itch.io HTML games docs](https://itch.io/docs/creators/html)
- 📄 [GitHub Pages docs](https://pages.github.com)

---

## Live Example

**[🐱 Mochi — The Cozy Cat](https://k4l4m.itch.io/mochi-cat)** — the game this guide was built from.
Callback page source: this repo's `index.html`.

---

*Built by [@kalamishere](https://github.com/kalamishere). PRs and corrections welcome.*
