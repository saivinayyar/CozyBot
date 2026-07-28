# CozyBot

A supportive chat companion for children and young adults, served by a small Express
backend so the Gemini API key never reaches the browser.

## Project structure

```
cozybot-app/
├── package.json
├── server.js          # Express app, exposes POST /api/chat
├── render.yaml         # Optional Render Blueprint for one-click setup
├── .env.example        # Copy to .env for local development
├── .gitignore
└── public/
    └── cozybot.html    # Frontend, HTML/CSS unchanged, JS calls /api/chat
```

## How it works

- The browser only ever talks to your own server, at `POST /api/chat`, sending the
  conversation history as `{ messages: [{ role: 'user' | 'assistant', content: '...' }] }`.
  This contract did not change when the provider changed, so `public/cozybot.html` did
  not need any edits.
- `server.js` prepends the CozyBot system prompt (including the safety and crisis
  rules) server-side, so it cannot be read or edited from the browser, converts the
  conversation into Gemini's `contents` format, and calls Google's Gemini API using
  the official `@google/genai` Node.js SDK.
- The `GEMINI_API_KEY` is read from an environment variable and is never sent to
  the client.
- CORS is restricted to a single origin, your GitHub Pages site, so only pages
  served from there can call `/api/chat` from a browser.
- A per-IP rate limiter (20 requests/minute) and input validation (message count,
  message length, allowed roles) are included as a baseline.

## Model

The server uses `gemini-3.5-flash`, Google's current generally available (GA),
stable Flash model at the time this was written. It is a reasonable default for a
sustained, safety-sensitive conversational use case: strong instruction-following
for the system prompt and crisis protocol, low latency, and GA status means Google
supports it for production traffic rather than experimental use. If a newer stable
model supersedes it, update the `GEMINI_MODEL` constant near the top of `server.js`;
check `https://ai.google.dev/gemini-api/docs/models` for the current recommendation
before switching, since model names and availability change over time.

## Running locally

Requires Node.js 18 or later.

```bash
cd cozybot-app
npm install
cp .env.example .env
# edit .env and set GEMINI_API_KEY=...
npm start
```

Then open `http://localhost:3000` in your browser.

Note: if you are testing the frontend from GitHub Pages against a locally running
server, the CORS origin restriction in `server.js` will block it unless you
temporarily add `http://localhost` style origins, or simply open
`public/cozybot.html` by visiting `http://localhost:3000` directly, which is
same-origin and unaffected by CORS.

## Getting a Gemini API key

1. Go to Google AI Studio: `https://aistudio.google.com/apikey`.
2. Sign in with a Google account and click **Create API key**.
3. Copy the generated key, you will paste it into Render's environment variables
   below (or into your local `.env` file).

## Deploying to Render

### Option A: Blueprint (uses render.yaml)

1. Push this project to a GitHub repository.
2. In the Render dashboard, click **New +** → **Blueprint**, and connect that
   repository. Render will detect `render.yaml` automatically.
3. When prompted, paste your Gemini API key as the value for `GEMINI_API_KEY`.
4. Click **Apply**. Render builds and deploys automatically.

### Option B: Manual Web Service

1. Push this project to a GitHub repository.
2. In the Render dashboard, click **New +** → **Web Service**, and connect that
   repository.
3. Configure the service:
   - **Environment**: `Node`
   - **Build Command**: `npm install`
   - **Start Command**: `npm start`
   - **Instance Type**: the free tier is sufficient to start
4. Under **Environment Variables**, add:
   - `GEMINI_API_KEY` = your Gemini API key
   - (Render sets `PORT` automatically, no action needed)
5. Click **Create Web Service**. Render will build and deploy automatically; future
   pushes to your connected branch will redeploy the service.
6. Once deployed, Render gives you a URL like
   `https://cozybot-app.onrender.com`, that is your backend's base URL.

### Connecting your GitHub Pages frontend

`server.js` only accepts requests from `https://saivinayyar.github.io`. If your
GitHub Pages site's actual origin differs from that, for example if you rename the
repository or use a custom domain, update the `ALLOWED_ORIGIN` constant in
`server.js` and redeploy.

The version of `public/cozybot.html` hosted on GitHub Pages must call your Render
URL rather than a relative path, since GitHub Pages and Render are different
origins. In `public/cozybot.html`, change:

```js
const response = await fetch('/api/chat', {
```

to:

```js
const response = await fetch('https://cozybot-app.onrender.com/api/chat', {
```

using your actual Render URL from step 6 above.

### Updating the API key later

Go to your service in the Render dashboard → **Environment** → edit
`GEMINI_API_KEY` → **Save Changes**. Render will redeploy with the new value.

## Notes on going further

This backend is a solid baseline, but before real children use this unsupervised
at any scale, consider:

- A persistent, shared rate limiter (e.g. Redis-backed) if you run more than one
  server instance.
- Logging/monitoring for repeated crisis-flagged conversations, routed to a human
  reviewer, not just surfaced to the child in the chat itself.
- A privacy policy and, depending on your audience and jurisdiction, compliance
  review (e.g. children's data protection law where you operate).
- Content moderation on top of the model's own behavior, as a second layer of
  defense.
- Gemini's own safety filters can withhold a response entirely for some inputs.
  The server currently returns a generic error in that case, logging
  `response.promptFeedback` for your own review; you may want to surface a softer,
  CozyBot-styled fallback message to the child instead.
