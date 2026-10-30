# Deploy a MERN Project on Vercel

Host the React frontend and the Express backend as **two Vercel projects** (one per folder of your repo), with your database on a managed service (e.g. MongoDB Atlas).

- [1. Express backend (zero config)](#1-express-backend-zero-config)
- [2. React frontend (React Router rewrites)](#2-react-frontend-react-router-rewrites)
- [3. Monorepo setup](#3-monorepo-setup)
- [4. Connecting to MongoDB from Vercel](#4-connecting-to-mongodb-from-vercel)
- [5. Legacy `vercel.json` for Express](#5-legacy-verceljson-for-express)
- [Limitations to know](#limitations-to-know)

---

## 1. Express backend (zero config)

Since late 2025, Vercel detects Express automatically — **no `vercel.json` needed**.

Vercel looks for a file that imports `express` at one of these paths:

```
app.js   index.js   server.js
src/app.js   src/index.js   src/server.js
```

(`.cjs`, `.mjs`, `.ts`, `.cts`, `.mts` also work.)

The file must **export the app** (or call `app.listen`):

```js
// server.js  — "type": "module" in package.json
import express from 'express';
import cors from 'cors';

const app = express();
app.use(cors({ origin: process.env.CLIENT_URL }));
app.use(express.json());

app.get('/api/health', (req, res) => res.json({ ok: true }));

// Run locally with `node server.js`, export for Vercel
if (!process.env.VERCEL) {
  app.listen(process.env.PORT || 4000);
}

export default app;
```

<details>
<summary>CommonJS version</summary>

```js
const express = require('express');
const app = express();

app.get('/api/health', (req, res) => res.json({ ok: true }));

module.exports = app;
```
</details>

Deploy:

```bash
npm i -g vercel
vercel          # preview deployment (first run links the project)
vercel --prod   # production
```

Or import the repo at [vercel.com/new](https://vercel.com/new) and push to deploy.

Add your environment variables (`MONGODB_URI`, `JWT_SECRET`, `CLIENT_URL`…) in *Project → Settings → Environment Variables*, or:

```bash
vercel env add MONGODB_URI production
```

**Static files:** put them in `public/`. `express.static()` is ignored on Vercel.

Run locally the Vercel way: `vercel dev` (CLI ≥ 47.0.5).

## 2. React frontend (React Router rewrites)

Without this, refreshing any page other than `/` returns a 404. Create `vercel.json` **at the root of the frontend project**:

```json
{
  "rewrites": [
    { "source": "/(.*)", "destination": "/index.html" }
  ]
}
```

Real files (JS, CSS, images in `dist/`) are still served normally — the rewrite only applies when no file matches.

Project settings for a Vite app (normally auto-detected):

| Setting | Value |
|---|---|
| Framework Preset | Vite |
| Build Command | `npm run build` |
| Output Directory | `dist` |
| Env var | `VITE_API_URL=https://your-backend.vercel.app` |

> Next.js projects don't need this file — routing is handled by the framework.

## 3. Monorepo setup

For a repo like:

```
your-repo/
├── backend/    → Vercel project #1 (Root Directory: backend)
└── frontend/   → Vercel project #2 (Root Directory: frontend)
```

1. Import the same repo **twice** at vercel.com/new.
2. In each project, set *Settings → Build and Deployment → Root Directory* to `backend` or `frontend`.
3. Each `vercel.json` goes inside its own folder, not at the repo root.
4. Set `CLIENT_URL` (backend) and `VITE_API_URL` (frontend) to each other's URLs. Add your custom domains in *Settings → Domains*.

## 4. Connecting to MongoDB from Vercel

Functions are short-lived and scale out, so **reuse the connection** between invocations instead of opening a new one per request:

```js
// db.js
import mongoose from 'mongoose';

let cached = globalThis.mongoose ?? (globalThis.mongoose = { conn: null, promise: null });

export async function connectDB() {
  if (cached.conn) return cached.conn;
  cached.promise ??= mongoose.connect(process.env.MONGODB_URI, { maxPoolSize: 10 });
  cached.conn = await cached.promise;
  return cached.conn;
}
```

```js
// in server.js
app.use(async (req, res, next) => {
  try {
    await connectDB();
    next();
  } catch (err) {
    next(err);
  }
});
```

MongoDB Atlas: Vercel's outgoing IPs are not fixed, so either allow `0.0.0.0/0` in Atlas *Network Access* (and use a strong password), or use the official [MongoDB Atlas integration](https://vercel.com/marketplace) from the Vercel Marketplace.

## 5. Legacy `vercel.json` for Express

Older tutorials (and earlier versions of this guide) used the `builds` + `routes` syntax. It still works but is **legacy** — prefer zero config above. Only keep it for old projects that need it:

```json
{
  "version": 2,
  "builds": [
    { "src": "server.js", "use": "@vercel/node", "config": { "includeFiles": ["dist/**"] } }
  ],
  "routes": [
    { "src": "/(.*)", "dest": "server.js" }
  ]
}
```

When `builds` is present, Vercel ignores the project's dashboard build settings.

---

## Limitations to know

Your Express app becomes **one Vercel Function** (Fluid compute). That means:

- **No persistent WebSocket server** (Socket.IO with long-lived connections). Use a VPS ([guide](mern-on-vps.md)) or a realtime service (Ably, Pusher, Supabase Realtime…).
- **No background jobs / `setInterval`** that run forever — use [Vercel Cron Jobs](https://vercel.com/docs/cron-jobs) calling a route.
- **No local disk storage** for uploads — use object storage (Vercel Blob, S3, Cloudinary…).
- **Bundle limit** 250 MB.
- **Handle errors yourself**: add an Express error middleware so a failing request doesn't leave the function in a bad state.

If you hit these limits, a [VPS deployment](mern-on-vps.md) is usually simpler and cheaper.

---

**References:** [Express on Vercel](https://vercel.com/docs/frameworks/backend/express) · [Zero-configuration Express backends (changelog)](https://vercel.com/changelog/zero-configuration-express-backends) · [Project configuration](https://vercel.com/docs/project-configuration)
