# Suppression de la v1 (legacy) & promotion de la v2 — design

**Date** : 2026-06-12
**Branche / worktree** : `chore/remove-v1-promote-v2` (worktree
`.worktrees/remove-v1-promote-v2`, basé sur le tip de `feat/refactor-backend-architecture`).

## Objectif

Le dépôt contenait deux générations de chaque brique (`backend/` + `backend-v2/`,
`frontend/` + `frontend-v2/`). On supprime la v1 (legacy) et on promeut la v2
comme **seule** génération présente, avec des noms de répertoire canoniques
(`backend/`, `frontend/`). Toutes les configs de déploiement, la CI et la
documentation sont mises à jour en conséquence.

## Décisions (validées avec l'utilisateur)

1. **Nommage** : après suppression de la v1, `backend-v2/` → `backend/` et
   `frontend-v2/` → `frontend/` (`git mv` pour préserver l'historique).
2. **Suite e2e** (`tests/e2e/`, Playwright ciblant la stack v1) : **supprimée**
   (pas de port). La couverture e2e v2 sera reconstruite plus tard si besoin.
3. **`docker-compose.yml`** : conserver un service `frontend` → créer un
   **Dockerfile Next.js** (`output: "standalone"`) pour `frontend/`.

## Périmètre

### 1. Suppressions (v1)
- `git rm -r backend/` (v1 : `main:app`, sans Alembic) — inclut son `Dockerfile`,
  `.dockerignore`, `CLAUDE.md`, `README.md`.
- `git rm -r frontend/` (v1 : React + Vite) — inclut `Dockerfile`, `vercel.json`,
  `README.md`.
- `git rm -r tests/` (ne contient que `tests/e2e/`, qui démarre la stack v1).

### 2. Renommage v2 → canonique
- `git mv backend-v2 backend`
- `git mv frontend-v2 frontend`

### 3. Configs de déploiement & CI
- **`render.yaml`** : `rootDir: backend`,
  `startCommand: alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port $PORT`
  (ajoute la migration Alembic, passe de `main:app` à `app.main:app`).
- **`.github/workflows/ci.yml`** : `backend-v2/**` → `backend/**`,
  `working-directory: backend`, `cache-dependency-path: backend/requirements-dev.txt`,
  titre `CI backend-v2` → `CI backend`.
- **`docker-compose.yml`** : service `backend` (`context: ./backend`) + service
  `frontend` (`build: ./frontend`, `BACKEND_URL=http://backend:8000`, `3000:3000`).
- **`frontend/next.config.ts`** : `output: "standalone"`.
- **`frontend/Dockerfile`** (nouveau) + `frontend/.dockerignore` : build Next.js
  multi-stage, run de la sortie `standalone` (port 3000).

### 4. Références internes (suite au renommage)
`backend/scripts/audit_scrapers.py`, `backend/README.md`, `frontend/README.md`,
`frontend/lib/types.ts`, `frontend/package.json` + `package-lock.json`,
`docs/test/2026-06-08-tests-api-bruno.md`.

### 5. Documentation racine
- **`AGENTS.md`** + **`README.md`** : réécriture pour ne décrire que la v2 (chemins
  `backend/` + `frontend/`).
- Specs historiques sous `docs/superpowers/specs/` : **inchangées** (archives datées).

## Hors périmètre
- Déploiement effectif (Render/Vercel/Supabase) : seuls les fichiers de config sont
  mis à jour ; la bascule reste une action manuelle.
- Réécriture des tests e2e contre l'UI v2 (sous-projet ultérieur).

## Vérification (acceptance)
- `backend/` : `ruff check .` + `pytest -m "not integration"` verts.
- `frontend/` : `npm run build` (prod) + `npm test` (Vitest) verts.
- `grep -rI "backend-v2\|frontend-v2"` ne renvoie plus que les archives
  `docs/superpowers/`.
- Plus aucun répertoire `backend-v2/` / `frontend-v2/` / `tests/`.

## Note d'exécution
Le worktree a d'abord été créé sur `835b4c6`, mais la branche cible a intégré
PR #10 (normalisation event_type, `classify.py`, `distance_km`, fix tsconfig
ES2018) pendant la session. Le worktree a été **recréé sur le tip `6de9bab`** et la
migration rejouée, afin que le `git mv` capture aussi les nouveaux fichiers de
PR #10 et que la branche reste mergeable sans conflit.
