# Connecting Forelark to a Shopify Store

A beginner-friendly, step-by-step guide to run this app locally and connect it
to a Shopify store — **and exactly where every credential comes from.**

> This complements [DEPLOY.md](DEPLOY.md). Use **this** file to get the app
> running on your machine against a **development store**. Use DEPLOY.md when
> you are ready to host it publicly (Render + Supabase).

---

## What you need before you start

| Thing | Cost | Why | Where |
| --- | --- | --- | --- |
| **Node.js** 20.19+ or 22.12+ | free | Runs the app | <https://nodejs.org> |
| **Shopify Partner account** | free | Registers the app + gives you test stores | <https://partners.shopify.com> |
| **A development store** | free | The store you install the app onto for testing | Created in the Partner Dashboard |
| **A Postgres database** | free | The app stores sessions, settings, suppliers, POs here | Supabase free tier, or local Docker Postgres |
| **Shopify CLI** | free | Already bundled — run it with `npx shopify ...` | `package.json` dependency |

You do **not** need a paid Shopify plan, a credit card, or a public server to
develop locally.

---

## Where do I get `SHOPIFY_API_KEY` and `SHOPIFY_API_SECRET`?

These are the app's **Client ID** and **Client secret**. They are generated for
you when you create an app in the **Shopify Partner Dashboard** — you never
invent them yourself.

### Step A — Create a Partner account
1. Go to <https://partners.shopify.com> and sign up (free).
2. Confirm your email.

### Step B — Create your own app
1. In the Partner Dashboard sidebar: **Apps → Create app → Create app manually**.
2. Name it anything (e.g. `Forelark-Bilal`).

> **Why create your own?** The `shopify.app.toml` in this repo already has a
> `client_id` that belongs to the **original developer's** app. You cannot get
> that app's secret because you don't own it. Creating your own app gives you a
> Client ID **and** a Client secret that are yours.

### Step C — Copy the two values
Inside your app, open **Settings** (or **Overview** / **Configuration** —
wording varies) and find the **Client credentials** section:

| Label in the Dashboard | Copy into `.env` as |
| --- | --- |
| **Client ID** (a.k.a. "API key") | `SHOPIFY_API_KEY` |
| **Client secret** (click **Reveal** 👁) | `SHOPIFY_API_SECRET` |

That's it — those are the only two secret values that come from Shopify.

> 💡 **Shortcut:** For local development you often don't even need to paste
> these by hand. Running `npx shopify app config link` links the repo to your
> app, and `npm run dev` injects the key and secret automatically during the
> dev session. You only set them manually when **deploying to a host** (see
> DEPLOY.md).

---

## Where do I get `DATABASE_URL`?

The app needs a Postgres database. Pick **one** option:

### Option 1 — Supabase (free, cloud, recommended)
1. Create a project at <https://supabase.com>.
2. **Project Settings → Database → Connection string → Session pooler**
   (port `5432`, IPv4).
3. Append the schema so Supabase's Data API never exposes the tables:
   ```
   postgresql://<role>.<project-ref>:<password>@aws-<n>-<region>.pooler.supabase.com:5432/postgres?schema=stocksense
   ```

### Option 2 — Local Postgres with Docker (fully offline)
```shell
docker run --name forelark-db -e POSTGRES_PASSWORD=postgres \
  -p 5432:5432 -d postgres:16
```
Then use:
```
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/postgres?schema=stocksense
```

---

## Where do I get the development store?

1. Partner Dashboard → **Stores → Add store → Create development store**.
2. Pick **"Create a store to test and build"**.
3. You become the store owner automatically. It's free and you can create as
   many as you want.

The `npm run dev` command below will let you pick this store and install the
app onto it in one step.

---

## Putting it together — run it locally

```shell
# 1. Install dependencies
npm install

# 2. Create your .env from the template, then fill in the values above
cp .env.example .env
#   Edit .env and set at minimum:
#     SHOPIFY_API_KEY     = your Client ID
#     SHOPIFY_API_SECRET  = your Client secret   (dev: optional, CLI injects it)
#     DATABASE_URL        = your Postgres URL
#   SCOPES is already correct; SHOPIFY_APP_URL is set automatically in dev.

# 3. Generate the Prisma client + run migrations (creates the tables)
npm run setup        # = prisma generate && prisma migrate deploy

# 4. Link this repo to YOUR app (writes your client_id into shopify.app.toml)
npx shopify app config link

# 5. Start the dev server — opens a tunnel and installs onto your dev store
npm run dev
```

During `npm run dev` the Shopify CLI will:
- log you into your Partner account,
- create a public HTTPS **tunnel** and set `SHOPIFY_APP_URL` to it automatically,
- let you **select the development store**,
- install the app and open it embedded inside Shopify Admin.

Approve the requested scopes (`read_inventory, read_products, read_orders,
read_locations`) when prompted, and the dashboard loads with your store's data.

---

## Access scopes this app requests

Configured in `shopify.app.toml` → `[access_scopes]` and must match the
`SCOPES` env var:

```
read_inventory, read_products, read_orders, read_locations
```

- `read_orders` gives the last **60 days** of orders (Shopify's default).
- To forecast on up to **365 days**, also request `read_all_orders` — Shopify
  must approve this scope for the app; the code uses the longer window
  automatically once granted.

---

## Verify everything works (no store needed)

All of these pass on a clean checkout once `npm install` and
`npx prisma generate` have run:

```shell
npm test             # 199 unit tests (forecast engine, parsers, billing, webhooks)
npm run typecheck    # tsc --noEmit
npm run lint         # eslint
npm run build        # production build
```

> **Common first-run gotcha:** if `npm run typecheck` reports errors like
> *"Namespace 'Prisma' has no exported member ..."* or *"implicitly has an
> 'any' type"*, it just means the **Prisma client hasn't been generated yet**.
> Run `npx prisma generate` (or `npm run setup`) once and they all disappear —
> the Prisma client is generated **from** `prisma/schema.prisma` into
> `node_modules`, so it doesn't exist until you generate it.

---

## Credential cheat-sheet

| Env var | What it is | Where to get it |
| --- | --- | --- |
| `SHOPIFY_API_KEY` | App **Client ID** | Partner Dashboard → your app → Client credentials |
| `SHOPIFY_API_SECRET` | App **Client secret** | Partner Dashboard → your app → Client credentials (Reveal) |
| `SCOPES` | Access scopes | Already set; must match `shopify.app.toml` |
| `SHOPIFY_APP_URL` | Public HTTPS base URL | Dev: set by `npm run dev`. Prod: your Render URL |
| `DATABASE_URL` | Postgres connection string | Supabase session pooler, or local Docker Postgres |

Never commit real credentials. `.env` is already in `.gitignore`.
