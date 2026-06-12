# AGENTS.md — data-triathlon

App web centralisant les résultats de compétition des membres d'un club de
triathlon (TCN). On colle une URL de chronométrage → le backend scrape, stocke,
et importe en arrière-plan tous les participants de l'épreuve.

Détails install/déploiement : voir `README.md`. Ce fichier cible les agents IA.

## Stack
- **Backend** (`backend/`) : Python 3.11+, FastAPI 0.115, SQLAlchemy 2.0 (sync),
  Pydantic v2 + pydantic-settings, **Alembic** (migrations), PostgreSQL (Supabase)
  / SQLite en dev. Scraping httpx + BeautifulSoup/lxml, fallback Playwright. Tests
  pytest, ruff. API versionnée sous `/api/v1`.
- **Frontend** (`frontend/`) : Next.js 16 (App Router) + TypeScript + Tailwind +
  shadcn/ui. Tests Vitest + RTL.
- **Déploiement** : backend → Render (`render.yaml`), frontend → Vercel, DB → Supabase.
  Stack locale conteneurisée via `docker-compose.yml` (backend + frontend).

## Commandes

```bash
# Backend (depuis backend/, venv activé)
uvicorn app.main:app --reload --port 8001  # API + /docs (endpoints sous /api/v1)
alembic upgrade head                        # applique les migrations (plus de create_all)
alembic revision --autogenerate -m "..."    # nouvelle migration après modif d'un modèle
pytest -m "not integration"                 # tests unitaires (sans réseau) — défaut CI
pytest -m integration                       # tests réseau réel (scrapers)
ruff check .                                 # lint

# Frontend (depuis frontend/)
npm run dev        # Next.js sur :3000, rewrites /api → :8001
npm run build      # build prod (strict TS + RSC)
npm test           # vitest run
npm run lint       # ESLint

# Stack locale conteneurisée (depuis la racine)
docker compose up --build   # backend :8000 + frontend :3000
```

Variable requise : `backend/.env` avec `DATABASE_URL` (voir `.env.example`).
Le schéma DB est géré par **Alembic** (`alembic upgrade head`).

## Architecture backend (`backend/`)

Archi en couches, le flux ne traverse qu'une direction
(`api → services → repositories → DB`) :

- `app/main.py` — usine `create_app()` : CORS, handlers d'erreurs, montage routers.
- `app/core/` — `config.py` (pydantic-settings), `logging.py`, `database.py`,
  `exceptions.py`, `time.py`, `club.py`.
- `app/models/` — SQLAlchemy **normalisé** : `Athlete`, `Course`, `Participation`,
  `PendingProvider` (voir « Modèle normalisé » plus bas).
- `app/schemas/` — DTO Pydantic v2 (entrée/sortie).
- `app/repositories/` — `*_repository.py` : **seule couche qui touche la Session**.
- `app/services/` — logique métier : `mapping`, `cache` (TTL), `scrape_service`,
  `import_service`, `stats_service`, `geocode_service`, `reclassify`.
- `app/api/` — `deps.py` + `v1/` (routers fins : validation + délégation au service),
  agrégés dans `v1/router.py`, montés sous `/api/v1`. Une future v2 vivra dans `v1/`→`v2/`.
- `app/scrapers/` — `registry.py` (registre **Protocol**, fin des `if-else`) +
  un module par provider. `classify.py` = classifieur unique de disciplines
  (event_type + distance_km), auquel les scrapers délèguent la détection.
  `base.py` = `ScrapedResult`, `utils.py` = helpers de normalisation.
- `alembic/` — migrations (révision initiale = schéma complet).
- `tests/` — `test_repositories/`, `test_services/`, `test_api/`, `test_classify.py`,
  un module de tests par scraper, etc.

### Modèle normalisé

- **Athlete** — `UNIQUE(nom, prenom, birth_date)`.
- **Course** — `UNIQUE(name, event_date, event_type)` ; `source_url` = clé de cache
  TTL ; `distance_km` (distance normalisée de l'épreuve).
- **Participation** — `UNIQUE(course_id, bib_number)` → plus de doublons à l'import.
- **splits** en **JSON** (remplace les colonnes figées swim/t1/bike/t2/run) →
  couvre tous les sports (duathlon course1/course2, swimrun…). Temps = strings.
  Les scrapers rangent les segments dans 5 slots positionnels triathlon
  (`swim/t1/bike/t2/run` de `ScrapedResult`) ; `services/mapping.build_splits`
  ré-étiquette ces slots selon `event_type` via le gabarit `_SPLIT_KEYS_BY_SPORT`
  (ex. duathlon → `course1`/`course2`, mono-sport → un seul segment) et omet les
  slots non pertinents. *Limite* : plafonné à 5 segments — un swimrun multi-legs
  reste collapsé. Évolution future si besoin : porter une **liste ordonnée de
  segments étiquetés** dès `ScrapedResult` (touche tous les scrapers).

### Cache TTL

`services/cache.py` : `is_fresh(course)` → 10 min si course en cours (une
participation sans `total_time`), sinon 30 j. `scrape_service` court-circuite le
re-scraping si frais. Réglable via `CACHE_TTL_IN_PROGRESS_SECONDS` /
`CACHE_TTL_FINISHED_SECONDS`.

### Conventions scrapers

- Tout nouveau fournisseur : créer `app/scrapers/<nom>.py`, exposer `scrape()` (et
  `scrape_event_all()` si l'import de masse est possible), puis l'enregistrer dans
  `app/scrapers/registry.py` (registre Protocol). Provider inconnu → `playwright`.
- **La détection de discipline est centralisée dans `classify.py`** : les scrapers
  délèguent (pas de logique de classification dupliquée par provider).
- **Breizh Chrono réutilise la logique Klikego** (`klikego._parse_detail`) — ne pas
  dupliquer, factoriser dans `klikego.py`.
- Identification club lors d'un import épreuve : filtre `city=nantais` de l'API
  (plus fiable que le nom de club, qui varie : « TCN », « TRIATHLON CLUB NANTAIS »…).
- Les temps restent des **strings** (`"01:23:45"`), normalisés via `utils.py`.
  Splits adaptés au sport : dans `splits` (JSON) + `raw_data` (JSON).

## Architecture frontend (`frontend/`)

Next.js 16 (App Router), TypeScript strict, Tailwind CSS, shadcn/ui, consommant
`/api/v1` du backend.

- `app/` — App Router : `dashboard`, `resultats`, `athletes/[id]`, `courses/[id]`,
  `club`, `carte`, `ajouter`, `admin`.
- `components/` — `scrape/` (ScrapeForm, ProviderDetector, ImportProgress),
  `results/` (ResultCard, ResultsList), `club/` (ClubView, AthleteDialog),
  `map/` (MapView), `dashboard/` (StatsCards, RecentCourses), `ui/` (shadcn).
- `lib/api/` — `client.ts` (appels `/api/v1`), `sse.ts` (streaming import SSE).
- `lib/types.ts` — types TypeScript partagés (miroir des schémas Pydantic du backend).
- `lib/constants.ts`, `lib/sport-colors.ts` — libellés de disciplines et couleurs.
- `next.config.ts` — rewrites `/api/*` → `BACKEND_URL` ; `output: "standalone"` (Docker).
- Déploiement : Vercel, variables `BACKEND_URL` + `API_URL`.

## Conventions générales

- **Langue** : UI, commentaires et messages en **français** (avec accents).
- Commits : Conventional Commits (`feat:`, `fix:`…), déjà en place dans l'historique.
- Schéma DB : migrations **Alembic** (`alembic revision --autogenerate` après modif
  d'un modèle, puis `alembic upgrade head`).
- Tests unitaires **sans réseau** ; le réseau réel est isolé derrière le marker
  `integration` (`pytest.ini`).

## Fournisseurs supportés

Klikego, Breizh Chrono, TimePulse, Wiclax/G-Live, ProLiveSport, Sport Innovation
(import d'épreuve complète). Types : Triathlon XS/S/M/L/XL, Duathlon XS/S/M/L,
SwimRun S/M/L, Aquathlon, Aquarun, Bike & Run.
