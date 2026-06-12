# Triathlon Club — Résultats de compétition

Application web pour centraliser les résultats de compétitions des membres du club TCN.
Collez une URL d'épreuve de chronométrage — le backend scrape, stocke et importe en
arrière-plan tous les participants de l'épreuve.

- **Backend** (`backend/`) : FastAPI + SQLAlchemy 2.0, architecture en couches,
  modèle normalisé, migrations Alembic, API versionnée sous `/api/v1`.
- **Frontend** (`frontend/`) : Next.js 16 (App Router) + TypeScript + Tailwind + shadcn/ui.

---

## Fonctionnalités

- **Import d'une épreuve** : coller une URL de chronométrage → tous les participants
  de l'épreuve sont scrapés et importés, avec une barre de progression temps réel (SSE)
- **Résultats** : liste complète par épreuve, avec filtres (nom, type, date)
- **Club TCN** : statistiques et résultats filtrés sur le club — les co-membres présents
  sur la même épreuve apparaissent automatiquement
- **Dashboard** : chiffres clés et répartition par discipline, filtrés sur le club
- **Carte** : géolocalisation des épreuves
- **Recherche globale** : navigation instantanée vers les résultats
- **Interface responsive** : navigation mobile, mise en page adaptative

---

## Prérequis

- **Python 3.11+**
- **Node.js 20+** (avec npm)
- **PostgreSQL** via [Supabase](https://supabase.com) (gratuit) — ou SQLite en local

---

## Installation locale

### 1. Cloner le projet

```bash
git clone https://github.com/TON_USERNAME/data-triathlon.git
cd data-triathlon
```

### 2. Base de données

**Option A — Supabase (recommandé pour la prod)**

1. Créer un projet sur [supabase.com](https://supabase.com)
2. **Connect** → **Direct** → copier l'URI de connexion
3. Créer `backend/.env` :

```env
DATABASE_URL=postgresql://postgres.VOTRE_REF:VOTRE_MDP@aws-0-eu-west-1.pooler.supabase.com:5432/postgres
```

**Option B — SQLite (dev local uniquement)**

```env
DATABASE_URL=sqlite:///./triathlon.db
```

### 3. Backend (FastAPI)

```bash
cd backend
python -m venv .venv && source .venv/bin/activate   # .venv\Scripts\activate sous Windows

pip install -r requirements-dev.txt   # requirements.txt seul en prod

alembic upgrade head                   # crée le schéma (géré par Alembic, plus de create_all)
uvicorn app.main:app --reload --port 8001
```

Backend : `http://localhost:8001` — Docs API : `http://localhost:8001/docs`

> Les endpoints sont versionnés sous **`/api/v1`** et le schéma DB est géré par
> **Alembic**. Voir [`backend/README.md`](backend/README.md) pour le détail.

### 4. Frontend (Next.js)

```bash
cd frontend
cp .env.local.example .env.local   # BACKEND_URL / API_URL → backend
npm install
npm run dev
```

Frontend : `http://localhost:3000`

> Les appels `/api/*` sont proxifiés vers `http://localhost:8001` via les rewrites Next.js.

---

## Providers supportés

| Site | Import épreuve complète |
|------|:-----------------------:|
| **Klikego** (`klikego.com`) | ✅ |
| **Breizh Chrono** (`resultats.breizhchrono.com`) | ✅ |
| **TimePulse** (`timepulse.fr`) | ✅ |
| **Wiclax / G-Live / ChronoSmetron** | ✅ |
| **ProLiveSport** (`prolivesport.fr`) | ✅ |
| **Sport Innovation** (`sportinnovation.fr`) | ✅ |

### Types d'épreuves supportés

Triathlon (XS/S/M/L/XL), Duathlon (XS/S/M/L), SwimRun (S/M/L), Aquathlon, Aquarun, Bike & Run.

### Identification des membres du club

Lors de l'import d'une épreuve, les co-membres sont identifiés par filtre sur le nom du
club (`nantais|TCN`). Les résultats sans club renseigné (certains providers) sont importés
sans filtre.

---

## Tests

### Backend — tests unitaires (sans réseau)

```bash
cd backend
pip install -r requirements-dev.txt
pytest -m "not integration"   # tests par couche
ruff check .                  # lint
```

Tests par couche : `test_repositories/`, `test_services/`, `test_api/`, `test_classify.py`,
plus un module par scraper (Klikego, TimePulse, Wiclax, ProLiveSport…).

### Backend — tests d'intégration (réseau réel)

```bash
pytest -m integration         # appels réels aux APIs des chronométreurs
```

### Frontend — tests Vitest

```bash
cd frontend
npm test                      # tests Vitest + RTL
npm run build                 # build prod (typage strict + RSC)
```

---

## Structure du projet

```
data-triathlon/
├── backend/                     # FastAPI — architecture en couches
│   ├── app/
│   │   ├── main.py              # create_app() : CORS, handlers d'erreurs, routers
│   │   ├── core/               # config (pydantic-settings), logging, database, exceptions
│   │   ├── models/             # SQLAlchemy normalisé : Athlete, Course, Participation
│   │   ├── schemas/            # DTO Pydantic v2
│   │   ├── repositories/       # accès données (seule couche qui touche la Session)
│   │   ├── services/           # métier : mapping, cache TTL, scrape, import, stats, reclassify
│   │   ├── api/v1/             # routers fins montés sous /api/v1
│   │   └── scrapers/           # registre Protocol + classify.py + un module par provider
│   ├── alembic/                # migrations (révision initiale = schéma complet)
│   ├── tests/                  # test_repositories / test_services / test_api / test_classify
│   ├── Dockerfile
│   └── README.md
├── frontend/                    # Next.js 16 (App Router) + TypeScript + Tailwind + shadcn/ui
│   ├── app/                    # routes : dashboard, resultats, athletes, courses, club, carte, ajouter, admin
│   ├── components/             # scrape/ results/ club/ map/ dashboard/ ui/
│   ├── lib/                    # api/client.ts, api/sse.ts, types.ts, constants.ts
│   ├── Dockerfile              # build Next.js « standalone »
│   └── README.md
├── docs/
│   ├── WORKFLOW-IA.md
│   └── superpowers/            # specs & plans de refonte
├── docker-compose.yml           # stack locale conteneurisée (backend + frontend)
└── render.yaml                  # config déploiement Render (backend)
```

---

## Déploiement

### Backend → Render.com

1. Connecter le repo GitHub sur [render.com](https://render.com)
2. `render.yaml` configure le service Python (`rootDir: backend`)
3. Ajouter la variable d'environnement `DATABASE_URL` (Supabase Session Pooler)

Le `startCommand` applique les migrations (`alembic upgrade head`) puis démarre
`uvicorn app.main:app`.

### Frontend → Vercel

1. Importer le repo sur [vercel.com](https://vercel.com)
2. **Root Directory** : `frontend`
3. Variables d'environnement :
   - `BACKEND_URL` — URL interne du backend Render (rewrites client)
   - `API_URL` — URL du backend pour les Server Components

### Stack locale (Docker)

```bash
docker compose up --build       # backend :8000 + frontend :3000
```

---

## Contribuer avec les outils IA (Superpowers + Speckit)

Ce projet embarque deux outils d'assistance IA préconfigurés pour le vibe coding.
Pour savoir quel outil utiliser (bugfix vs vraie feature, quand lancer les
sous-agents…) : voir [`docs/WORKFLOW-IA.md`](docs/WORKFLOW-IA.md).
