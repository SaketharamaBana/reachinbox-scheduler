# Email Job Scheduler (BullMQ + Redis + Postgres + Elasticsearch)

Production-style email scheduler with a React dashboard. Emails are stored in Postgres, scheduled as **BullMQ delayed jobs** (no cron anywhere) and sent through **Ethereal** fake SMTP.

## Quick start

```bash
docker compose up -d                 # Postgres, Redis (AOF on), Elasticsearch

cd backend
cp .env.example .env                 # fill in Google + Slack credentials
npm install
npm run dev                          # API + embedded worker on :4000

cd ../frontend
npm install
npm run dev                          # http://localhost:5173
```

- **BullMQ dashboard (live):** http://localhost:4000/admin/queues
- **Run the worker separately** (for horizontal scaling): set `EMBED_WORKER=false` on the API and run `npm run worker` on as many instances as you like.
- **Ethereal:** no manual setup. On first boot the backend calls `nodemailer.createTestAccount()` `SENDER_COUNT` times and saves the accounts in the `senders` table. Each sent email row in the UI links to its Ethereal preview URL.

### Credentials you must create
| Service | Where | Redirect URI |
|---|---|---|
| Google OAuth | Google Cloud Console → Credentials → OAuth client (Web) | `http://localhost:4000/api/auth/google/callback` |
| Slack app | api.slack.com/apps → enable **Incoming Webhooks**, OAuth scope `incoming-webhook` | `https://<ngrok>/api/slack/callback` (Slack requires HTTPS; run `ngrok http 4000`) |

### Environment variables (`backend/.env`)
| Var | Default | Meaning |
|---|---|---|
| `WORKER_CONCURRENCY` | 5 | parallel jobs per worker process |
| `MIN_DELAY_BETWEEN_EMAILS_MS` | 2000 | min gap between any two sends (**I chose 2 s**) |
| `MAX_EMAILS_PER_HOUR_PER_SENDER` | 50 | default hourly cap per sender (overridable per campaign in the UI) |
| `SENDER_COUNT` | 3 | number of Ethereal senders (round-robin) |

## Architecture

```
React ──> Express API ──> Postgres (source of truth)
               │  └────> Elasticsearch (search index)
               └──> BullMQ queue (Redis) ──> Worker(s) ──> Ethereal SMTP
                                               ├─ Redis counters (hourly limit)
                                               └─ Slack webhook (on limit hit)
```

### How scheduling works
`POST /api/emails/schedule` validates input, assigns each recipient a sender (round-robin) and a send time (`start + i × delay`), bulk-inserts rows in Postgres, then adds one BullMQ **delayed job per email** with `jobId = emailId` and `delay = scheduledAt − now`. No polling loop and no cron.

### Persistence across restarts
- Jobs live in Redis (AOF enabled in `docker-compose.yml`), rows live in Postgres. Stopping/starting the server loses nothing; workers pick delayed jobs up when they become due.
- On boot, `reconcileJobs()` compares pending DB rows with the queue and re-enqueues any missing job (e.g. Redis was wiped). Same `jobId` means it can never double-enqueue.

### Idempotency (no duplicate sends)
1. `jobId = emailId` → the queue holds at most one job per email.
2. The worker starts with an atomic `UPDATE … SET status='processing' WHERE id=? AND status='scheduled' RETURNING *`. If zero rows come back (already sent / claimed by another worker) the job exits without sending.
3. A `processing` row can only be re-claimed after 2 minutes (crash recovery).

### Concurrency & minimum delay
- `WORKER_CONCURRENCY` sets parallel jobs per worker. State is only touched via atomic SQL/Redis ops, so parallelism is safe.
- The BullMQ **limiter** `{max: 1, duration: MIN_DELAY_BETWEEN_EMAILS_MS}` is enforced in Redis for the whole queue, i.e. across all workers/instances: at most one send per 2 s.

### Hourly rate limiting
- Redis key `rl:{senderId}:{hourWindow}` incremented by a **Lua script** (atomic "increment if under limit"), so it is correct across workers and instances. Limit = campaign's hourly limit, else `MAX_EMAILS_PER_HOUR_PER_SENDER`.
- If the limit is hit the job is **not dropped or failed**: the DB row is moved to the next hour window and the job is moved with `job.moveToDelayed`. Order is preserved via a per-window Redis overflow counter: `resumeAt = nextHour + (slot−1) × minDelay`.
- **Slack alert:** on the first hit per sender per window, a real message is POSTed to the user's Slack incoming webhook (obtained via OAuth). No Slack connected → skipped silently. Connect/disconnect works at runtime with no redeploy.

### Behaviour under load (1000+ emails at once)
Bulk insert + `addBulk` in chunks of 500 keeps scheduling fast. Workers drain at ≤ 1 email / 2 s; each sender's hourly cap is enforced; overflow jobs slide into the next hour in original order. Nothing is lost, and a Slack alert fires once per sender per hour.

### Search
Every scheduled/sent/rescheduled email is indexed in Elasticsearch (`emails` index). `GET /api/emails/search?q=text` does fuzzy multi-field search (recipient, subject, body) scoped to the current user. Indexing failures never block sending.

## Features
**Backend:** delayed-job scheduler · restart persistence + reconcile · idempotent sends · configurable concurrency · min delay between sends · Redis-backed per-sender hourly limit with rescheduling · multiple Ethereal senders · Slack OAuth + live notification · Elasticsearch indexing/search · live Bull Board · Google OAuth (cookie JWT).
**Frontend:** Google login · header with name/email/avatar/logout · Scheduled & Sent tabs · Compose modal (subject, body, CSV upload with detected count, start time, delay, hourly limit) · loading/empty/error states · toasts · Connect Slack · typed API layer, reusable UI components, 5 s live polling.

## API
`GET /api/auth/google` · `GET /api/auth/me` · `POST /api/auth/logout` · `POST /api/emails/schedule` · `GET /api/emails?status=scheduled|sent` · `GET /api/emails/search?q=` · `GET /api/slack/connect|status` · `DELETE /api/slack`

## Assumptions / trade-offs
- The Figma link was not included in the brief, so the UI follows the described layout; restyle to match your Figma.
- A failed SMTP send is marked `failed` and **not retried** (favours at-most-once over duplicates).
- The rate-limit counter is consumed at send time; a send that fails after consuming still counts.
- Bull Board is unauthenticated for demo convenience; put it behind auth in production.
- Rescheduling beyond one window cascades naturally (overflow jobs re-check the limit in the next hour).

## Demo video checklist (≤ 5 min)
1. Log in with Google → Compose with `sample-leads.csv` → Schedule.
2. Show Scheduled → Sent tabs and Ethereal preview links, plus `/admin/queues`.
3. Schedule 2 min ahead → stop the server → restart → email still sends once.
4. Set hourly limit to 2 with 5 leads and Slack connected → show the Slack message and the rows sliding to the next hour.
