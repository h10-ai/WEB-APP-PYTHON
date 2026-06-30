# Lumis — Tech stack explained

This document answers one question for every piece of technology in this project: **why this, and not something else?** Each section explains the choice, the alternative we passed on, and a concrete example of why it matters in practice. This is meant to be readable on its own, even without me there to explain it.

---

## 1. Django (the backend framework)

**What it is**: Django is a Python web framework — a large, pre-built toolkit for handling things every web app needs (URLs, database access, user sessions, security) so you don't write them from scratch.

**Why we chose it over alternatives like Flask or FastAPI**: Django comes with a built-in database layer (the ORM), an admin panel, and a clear project structure out of the box. For a solo builder, this means less decision fatigue — Django tells you where things go. Flask gives you more freedom but also more rope to hang yourself with, since you have to choose and wire up every piece yourself.

**Example**: When we build the chat history feature (Step 3), Django's ORM lets us write:
```python
Chat.objects.filter(user=current_user).order_by('-created_at')[:5]
```
instead of writing raw SQL like `SELECT * FROM chats WHERE user_id = ? ORDER BY created_at DESC LIMIT 5`. Same result, far less room for error.

---

## 2. Django Channels (real-time WebSocket support)

**What it is**: A Django add-on that allows the server to hold a connection open and push messages to the browser at any time — not just respond when asked.

**Why we need it at all**: Plain Django (and most web frameworks by default) only understands the request-response pattern: browser asks, server answers, connection closes. A chat app needs the opposite — the server needs to send a reply the moment it's ready, without the browser having to keep re-asking "is there a reply yet?"

**Example**: Without Channels, building "live chat" would mean the browser polls your server every second asking "any new messages?" — wasteful and laggy. With Channels, the server simply pushes the message down the open WebSocket connection the instant Gemini responds. This is the same underlying mechanism WhatsApp Web or Slack uses to show messages appearing instantly.

---

## 3. SQLite (the database)

**What it is**: A database that lives as a single file on disk, with zero separate server process to install or manage. It ships built into Django by default.

**Why we chose it over Postgres for now**: Postgres is more powerful and handles many simultaneous users better, but it requires installing and running a separate database server — extra setup, extra moving parts. Since Lumis is currently a solo project, not yet under real multi-user load, SQLite gives the same Django ORM experience with zero infrastructure overhead. We can migrate to Postgres later — Django makes this a config change, not a rewrite — if our hosting platform doesn't persist SQLite files between deploys (something we'll need to verify once we pick Render or Railway).

**Example**: Right now, your entire chat history database is one file: `db.sqlite3`. You can literally copy that file to back up every conversation ever had with Lumis. With Postgres, you'd need a separate backup/export process.

---

## 4. Gemini API, Bring-Your-Own-Key (BYOK)

**What it is**: Instead of you paying for every API call your users make, each user supplies their *own* Gemini API key, stored only in their browser.

**Why we chose this over you holding one shared key**: If you held a single API key on your server, every user's usage would come out of your wallet — a single popular post could bankrupt you in API costs. BYOK shifts that cost to each user, who already gets free Gemini API quota from Google. This directly serves your "keep cost down" goal from day one, not as an afterthought.

**Example**: If 50 students use Lumis simultaneously, with a shared key you'd be paying for all 50 conversations. With BYOK, each student pays nothing extra beyond their own free Gemini quota — your hosting bill stays at $0 regardless of how many people use the app.

---

## 5. localStorage (where the API key lives)

**What it is**: A small storage space built into every browser, tied to a specific website, that persists even after the tab closes.

**Why we chose it over storing keys on your server**: Storing user API keys on your server means you become responsible for securing them — encryption, access control, breach risk. By keeping the key in the browser only, your server never sees it, never stores it, and can never leak it. This is the simplest possible security posture: you can't lose what you never had.

**Example**: A user pastes their Gemini key into Lumis once. The browser remembers it for their next visit (via localStorage) without ever sending it to your database. If your server were ever hacked, there would be no API keys for an attacker to steal — they were never there.

---

## 6. Gemini Live API (for Speech-to-Speech, Step 5)

**What it is**: A version of the Gemini API designed specifically for real-time audio conversations — you stream audio in, it streams audio back, with no separate "convert to text first" step.

**Why this connects directly from the browser, not through Django**: Since the user's key lives in their browser (not on your server), the most natural place to make this call is also the browser — there's no reason to route audio through your server first when your server doesn't even hold the credentials to make the call itself. This also means your server has zero audio-processing load, which keeps hosting costs at $0.

**Example**: When a user speaks into the mic in STS mode, that audio goes straight from their browser to Google's servers and back — your Django backend is completely uninvolved in that exchange, the same way your backend isn't involved when a user makes a direct Google Maps API call from their browser.

---

## 7. Web Speech API (fallback / Step 1 starting point — superseded by Gemini Live)

**What it is**: A free, browser-built-in speech recognition and text-to-speech feature.

**Why it was the original starting recommendation, and why we moved past it**: It's free and simple, which made it a good "prove the pipeline works" first step. But you decided Speech-to-Speech via Gemini Live is a *crucial* feature, not a nice-to-have — and Web Speech API can't deliver true real-time speech-to-speech quality or the multiple distinct voice options your spec requires. We're keeping this note in the document for context, but the live architecture is now built around Gemini Live API directly, as confirmed.

---

## Summary table

| Layer | Technology | Core reason |
|---|---|---|
| Backend framework | Django | Batteries-included, less setup decisions |
| Real-time layer | Django Channels | Lets server push messages instantly |
| Database | SQLite (Django default) | Zero setup, upgradeable to Postgres later |
| AI text/voice engine | Gemini API | Your choice, strong free tier |
| Key handling | BYOK + localStorage | Keeps your cost at $0, removes security liability |
| Speech-to-speech | Gemini Live API (direct from browser) | True real-time audio, bypasses your server entirely |
| Hosting | Render / Railway free tier | $0 cost to start |

---

*This document reflects architecture decisions as of the current build plan. If a decision changes (e.g. migrating to Postgres, adding a paid hosting tier), update the relevant section rather than starting a new document — keeping one source of truth avoids confusion later.*
