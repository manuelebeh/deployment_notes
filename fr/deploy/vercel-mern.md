# Déployer un projet MERN sur Vercel

[🇬🇧 Read in English](../../deploy/vercel-mern.md)

Hébergez le frontend React et le backend Express comme **deux projets Vercel** (un par dossier de votre dépôt), avec la base de données sur un service managé (par ex. MongoDB Atlas).

- [1. Backend Express (sans configuration)](#1-backend-express-sans-configuration)
- [2. Frontend React (réécritures React Router)](#2-frontend-react-réécritures-react-router)
- [3. Monorepo](#3-monorepo)
- [4. Se connecter à MongoDB depuis Vercel](#4-se-connecter-à-mongodb-depuis-vercel)
- [5. Ancien `vercel.json` pour Express](#5-ancien-verceljson-pour-express)
- [Limites à connaître](#limites-à-connaître)

---

## 1. Backend Express (sans configuration)

Depuis fin 2025, Vercel détecte Express automatiquement — **pas besoin de `vercel.json`**.

Vercel cherche un fichier qui importe `express` à l'un de ces emplacements :

```text
app.js   index.js   server.js
src/app.js   src/index.js   src/server.js
```

(`.cjs`, `.mjs`, `.ts`, `.cts`, `.mts` fonctionnent aussi.)

Le fichier doit **exporter l'app** (ou appeler `app.listen`) :

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
<summary>Version CommonJS</summary>

```js
const express = require('express');
const app = express();

app.get('/api/health', (req, res) => res.json({ ok: true }));

module.exports = app;
```

</details>

Déployer :

```bash
npm i -g vercel
vercel          # preview deployment (first run links the project)
vercel --prod   # production
```

Ou importez le dépôt sur [vercel.com/new](https://vercel.com/new) et pushez pour déployer.

Ajoutez vos variables d'environnement (`MONGODB_URI`, `JWT_SECRET`, `CLIENT_URL`…) dans *Project → Settings → Environment Variables*, ou :

```bash
vercel env add MONGODB_URI production
```

**Fichiers statiques :** mettez-les dans `public/`. `express.static()` est ignoré sur Vercel.

Lancer en local comme sur Vercel : `vercel dev` (CLI ≥ 47.0.5).

## 2. Frontend React (réécritures React Router)

Sans cela, rafraîchir n'importe quelle page autre que `/` renvoie une 404. Créez `vercel.json` **à la racine du projet frontend** :

```json
{
  "rewrites": [
    { "source": "/(.*)", "destination": "/index.html" }
  ]
}
```

Les vrais fichiers (JS, CSS, images de `dist/`) restent servis normalement — la réécriture ne s'applique que si aucun fichier ne correspond.

Réglages du projet pour une app Vite (normalement détectés automatiquement) :

| Réglage | Valeur |
|---|---|
| Framework Preset | Vite |
| Build Command | `npm run build` |
| Output Directory | `dist` |
| Variable d'env | `VITE_API_URL=https://your-backend.vercel.app` |

> Les projets Next.js n'ont pas besoin de ce fichier — le routage est géré par le framework.

## 3. Monorepo

Pour un dépôt comme :

```text
your-repo/
├── backend/    → Vercel project #1 (Root Directory: backend)
└── frontend/   → Vercel project #2 (Root Directory: frontend)
```

1. Importez **deux fois** le même dépôt sur vercel.com/new.
2. Dans chaque projet, réglez *Settings → Build and Deployment → Root Directory* sur `backend` ou `frontend`.
3. Chaque `vercel.json` va dans son propre dossier, pas à la racine du dépôt.
4. Renseignez `CLIENT_URL` (backend) et `VITE_API_URL` (frontend) avec l'URL de l'autre projet. Ajoutez vos domaines personnalisés dans *Settings → Domains*.

## 4. Se connecter à MongoDB depuis Vercel

Les fonctions ont une durée de vie courte et se multiplient avec le trafic : **réutilisez la connexion** entre les appels au lieu d'en ouvrir une nouvelle à chaque requête :

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

MongoDB Atlas : les IP sortantes de Vercel ne sont pas fixes. Soit vous autorisez `0.0.0.0/0` dans *Network Access* d'Atlas (avec un mot de passe solide), soit vous utilisez l'[intégration MongoDB Atlas](https://vercel.com/marketplace) officielle du Marketplace Vercel.

## 5. Ancien `vercel.json` pour Express

Les anciens tutoriels (et les versions précédentes de ce guide) utilisaient la syntaxe `builds` + `routes`. Elle fonctionne encore mais est **obsolète** — préférez la méthode sans configuration ci-dessus. Gardez-la seulement pour les anciens projets qui en ont besoin :

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

Quand `builds` est présent, Vercel ignore les réglages de build du tableau de bord.

---

## Limites à connaître

Votre app Express devient **une seule Vercel Function** (Fluid compute). Concrètement :

- **Pas de serveur WebSocket persistant** (Socket.IO avec des connexions longues). Utilisez un VPS ([guide](mern-on-vps.md)) ou un service temps réel (Ably, Pusher, Supabase Realtime…).
- **Pas de tâches de fond / `setInterval`** qui tournent en permanence — utilisez les [Vercel Cron Jobs](https://vercel.com/docs/cron-jobs) qui appellent une route.
- **Pas de stockage sur disque** pour les uploads — utilisez un stockage objet (Vercel Blob, S3, Cloudinary…).
- **Limite de bundle** de 250 Mo.
- **Gérez vous-même les erreurs** : ajoutez un middleware d'erreur Express pour qu'une requête en échec ne laisse pas la fonction dans un état instable.

Si vous atteignez ces limites, un [déploiement sur VPS](mern-on-vps.md) est souvent plus simple et moins cher.

---

**Références :** [Express sur Vercel](https://vercel.com/docs/frameworks/backend/express) · [Backends Express sans configuration (changelog)](https://vercel.com/changelog/zero-configuration-express-backends) · [Configuration de projet](https://vercel.com/docs/project-configuration)
