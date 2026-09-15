# CardWise Waitlist System — How Everything Works

See the companion flowchart (`cardwise-system-flowchart.mermaid`) alongside this document — it maps visually to every section below.

---

## 1. The big picture, in plain words

Four separate services work together, even though none of them "know" about each other directly — they just call one another over the internet:

1. **GitHub** — stores your website's code (just the files, doesn't run anything)
2. **GitHub Pages / Cloudflare DNS** — actually serves your website to visitors when they type `poshwaytech.com`
3. **Cloudflare Workers + D1** — your backend: receives form submissions, checks rules, stores data
4. **Resend** — a separate company that sends the actual emails on your behalf

Nothing here is one single "app" — it's four small pieces, each doing one job, connected by simple web requests (the same kind your browser makes every time you visit any website).

---

## 2. What is a Cloudflare Worker, really?

Think of a Worker as **a tiny program that wakes up only when someone visits a specific URL, does one job, then goes back to sleep.**

- It's not a computer running 24/7 somewhere — Cloudflare runs it on-demand, on servers spread around the world, so it responds fast no matter where the visitor is
- It has no memory between requests — every time it runs, it starts fresh. That's *why* we need D1 (the database): the Worker itself can't "remember" who signed up, so it saves that to D1 instead
- Your Worker only knows how to do what's written in `worker.js` — right now, that's exactly three things: report the spot count, accept a signup, and process an unsubscribe

This is different from a traditional backend (like a FastAPI server) which runs continuously on one server. Workers are "serverless" — you don't manage a server at all; Cloudflare just runs your code whenever it's needed.

---

## 3. What is D1?

D1 is Cloudflare's built-in database — think of it as a simple spreadsheet living inside Cloudflare, with rows and columns, that only your Worker is allowed to read or write.

Your one table, `waitlist`, has these columns:

| Column | What it stores |
|---|---|
| `id` | A number that auto-increases with each signup — this doubles as "queue position" |
| `email` | The person's email, lowercased |
| `created_at` | Exact timestamp of when they signed up |
| `notified` | Not actively used yet — reserved for future tracking of who's been emailed |
| `unsub_token` | A random, unique code — like a secret password — used only in unsubscribe links |

The Worker is the *only* thing that can talk to this database. Nobody on the internet can query it directly — it's not a public website, it's a private resource the Worker reaches internally.

---

## 4. What is Resend, and how does it connect?

Resend is a separate company (not part of Cloudflare) that specializes in reliably delivering emails. Your Worker doesn't "send emails" itself — it makes a request to Resend's servers saying, essentially, *"please send this email to this address,"* and Resend handles all the technical delivery work (formatting, routing through the internet's email systems, and trying to land in the inbox instead of spam).

**How the connection is secured:** your Worker holds a secret API key (`RESEND_API_KEY`) that proves to Resend "yes, this request is really coming from your account." This key is stored using `wrangler secret put`, which keeps it encrypted and hidden — it's never visible in your code or on GitHub.

**Domain verification:** Resend needed proof that you actually own `poshwaytech.com` before letting you send email *as* that domain (this stops random people from sending fake emails pretending to be you). That's what the DNS records you added in Cloudflare were for.

---

## 5. Every file, and exactly when it's used

| File | Lives where | Used when |
|---|---|---|
| `index.html` | GitHub repo `cardwise-legal` | Every time someone visits `poshwaytech.com` — this is the actual webpage, including the signup form and the JavaScript that talks to the Worker |
| `worker.js` | Your WSL folder `~/cardwise-worker/`, deployed to Cloudflare | Runs automatically, instantly, every time someone visits `/api/waitlist`, `/api/waitlist/count`, or `/api/unsubscribe` — this is your actual backend logic |
| `wrangler.toml` | Same WSL folder | Only read when you run a `wrangler` command (like `wrangler deploy`) — tells Wrangler which Worker this is and which database to connect it to. Never runs on its own |
| `schema.sql` | Same WSL folder | Only used once, manually, when you first created the `waitlist` table. Not used automatically — it's just a record of the table structure, useful if you ever need to recreate it |
| `send-updates.js` | Same WSL folder (or wherever you saved it) | Only runs when *you* manually type `node send-updates.js` — never runs automatically. This is how progress-update and launch emails go out |
| `waitlist.json` | Created temporarily in your WSL folder | Only exists after you run the `wrangler d1 execute ... > waitlist.json` export command — it's a snapshot of your database at that moment, which `send-updates.js` then reads. You regenerate this fresh every time you want to send an update, so it always reflects the current list |

**Important nuance about `waitlist.json`:** it's not a live connection to the database — it's a one-time snapshot. If someone signs up *after* you've exported it, they won't be in that file until you export again. That's why the instructions always say "export, then send" as two separate steps, done fresh each time.

---

## 6. Full data flow, step by step

**When someone signs up:**
1. Visitor's browser runs the JavaScript inside `index.html`
2. That JavaScript sends a `POST` request to your Worker's URL with their email
3. The Worker (`worker.js`) checks: is the email valid? Are there spots left? Is it a duplicate?
4. If all checks pass, the Worker writes a new row into the D1 `waitlist` table, generating a random `unsub_token` for that row
5. The Worker immediately calls Resend's API, asking it to send the confirmation email — with the unsubscribe link built using that same token
6. The Worker replies back to the browser with success + their queue position, which the page displays

**When you send an update or launch email:**
1. You run a `wrangler` command in WSL that reads the current D1 table and saves it to `waitlist.json`
2. You run `node send-updates.js`, which opens `waitlist.json`, loops through every person, and calls Resend once per person with their personalized email (position number + their own unique unsubscribe link)
3. Resend delivers each one independently

**When someone unsubscribes:**
1. They click the link in any email, which points to your Worker's `/api/unsubscribe` endpoint with their personal token attached
2. The Worker looks up that token in D1, finds the matching row, and deletes it entirely
3. They see a simple confirmation page; they're now completely gone from the database and won't receive anything further

---

## 7. Why nothing here needs a traditional server

You may notice there's no "server" running 24/7 that you have to maintain, restart, or pay for constantly. That's the nature of this stack:
- The website is static files, served by GitHub Pages
- The backend logic only runs in short bursts, exactly when triggered by a request (Cloudflare Workers)
- The database is managed entirely by Cloudflare — no setup, no patching, no backups to configure yourself
- Email sending is outsourced entirely to Resend's infrastructure

This is why the whole thing costs close to $0 at your current scale — you're only ever "renting" tiny slices of computing time, not a whole server sitting idle most of the day.
