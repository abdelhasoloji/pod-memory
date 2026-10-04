# ddostester — Architecture (WEDOS ShieldTest control plane)

Résumé de l'architecture du dépôt `abdelhasoloji/ddostester` (plateforme interne de **tests de résilience** : orchestration, gouvernance, mesure et scoring de campagnes de charge contre des cibles autorisées).

## Vue d'ensemble

- **Backend** : Fastify + TypeScript (ESM, Node ≥ 22), API REST + SSE.
- **Frontend** : React + Vite (dashboard, login/RBAC, campagnes, WRS, audit, progression live).
- **Persistance** : PostgreSQL (fallback mémoire si `DATABASE_URL` absent).
- **Périmètre défensif** : le control plane ne contient **aucun vecteur d'attaque** — uniquement un générateur HTTP borné (allowlist + RPS cap) et une sonde slow-header bornée. Les vecteurs amplification/UDP vivent dans le dépôt séparé `abdelhasoloji/shieldtest-lab` (contrat de verdicts en lecture seule).

## Fichiers clés et leur rôle

### Racine
| Fichier | Rôle |
| --- | --- |
| `CLAUDE.md` | Guide contributeurs/agents : périmètre non négociable, install/run/test, conventions, golden rule (allowlist). |
| `README.md` | Présentation, features (CAM/GEN/PROBE/WRS/RPT), API, safeguards, déploiement. |
| `package.json` | npm workspaces (backend + frontend), scripts `dev:api`, `dev:web`, `test`, `verify`. |
| `docker-compose.yml` | Stack Compose : API + Postgres (+ profils monitoring, proxy mTLS). |
| `.env.example` | Variables d'environnement de référence (DB, JWT, ALLOWLIST, caps…). |
| `deploy/` | Déploiement systemd (loopback) + PKI mTLS (`make-ca-certs.sh`). |
| `docs/` | Documentation complète : architecture, sécurité/gouvernance, boundary & lab, WRS, opérations, observabilité, API, roadmap. |

### Backend — cœur
| Fichier | Rôle |
| --- | --- |
| `backend/src/app.ts` | `buildApp()` : construction de l'app Fastify (routes + logique), injectable pour les tests. Contient le flux d'exécution des campagnes (`runCampaign`), le séquenceur de suites, l'auth/RBAC, les routes bench class-C, playbooks, users, limits, SSE live. |
| `backend/src/server.ts` | Point d'entrée : store, auth, scheduler (campagnes planifiées), rétention des données, graceful shutdown, bind loopback par défaut. |
| `backend/src/config/limits.ts` | **Source unique de vérité** des caps de sécurité : `HARD_CAPS` (maxRps 2000, maxConcurrency 500, maxDurationSec 900), `PROBE_LIMITS`, bornes de run-shape, plafonds du moteur builtin (150 rps / 300 conc.), calibration. |
| `backend/src/engine/loadTest.ts` | Générateur HTTP borné (phases, RPS cap, kill switch, backstop hard-stop, pool undici dédié). Invariante : une run se termine toujours à cap + grace. |
| `backend/src/engine/slowloris.ts` | Sonde slow-header bornée (vecteur `http-slowloris`) : mesure si la cible élimine les requêtes lentes. |
| `backend/src/agents/index.ts` | Fabrique d'agents de charge : builtin, k6, vegeta, locust, jmeter, gatling + détection de disponibilité. |
| `backend/src/store/index.ts` | Sélection du store : PostgreSQL si `DATABASE_URL`, sinon mémoire. |
| `backend/src/auth/` | JWT HS256 + RBAC (engineer/ciso/admin/viewer), gestion des utilisateurs runtime. |
| `backend/src/routes/read-routes.ts` | Routes de lecture (runs, rapports, WRS history, metrics). |
| `backend/src/routes/target-routes.ts` | Onboarding gouverné des cibles protégées. |
| `backend/src/routes/assistant-routes.ts` | Assistant conversationnel (bridge LLM OpenAI-compatible, fail-soft). |

### Backend — modules métier (`backend/src/modules/`)
| Module | Rôle |
| --- | --- |
| `campaigns/` | Modèle de campagne, workflow d'approbation (separation of duties), runner, scheduler, recovery gate. |
| `probe/` | Sonde légitime + kill switch 3 niveaux (auto / manuel / time-based). |
| `wrs/scoring.ts` | **WRS** — WEDOS Resiliency Score : fonction pure `0.20·D + 0.30·M + 0.30·A + 0.20·C`, niveaux A+→F, pas de trafic généré. |
| `wrs/history.ts` | Historique WRS par cible + détection de régression. |
| `reports/` | Rapports HTML de campagne + conformité DORA/NIS2, absorption, confiance de mesure. |
| `bench/` | Bench réseau class-C : scénarios, verdicts, coverage, attestation (contrat avec shieldtest-lab). |
| `playbooks/` | Séquences d'évaluation curées + playbooks custom. |
| `recon/` | Reconnaissance passive/pré-test de la surface publiée d'une cible autorisée. |
| `audit/` | Journal d'audit persistant (tamper-evident). |
| `metrics/` | Métriques Prometheus (`/metrics`) : HTTP, runs in-flight, hard-stop, event-loop lag. |
| `notify/` | Notifications opt-in (Slack/webhook) sur complétion, kill, régression WRS. |
| `protection/` | Intégration détection/mitigation réelle (TTD/TTM). |
| `edge-absorption/` | Corroboration bilatérale avec le rapport d'absorption de l'edge. |
| `target-telemetry/` | Télémétrie hôte cible corrélée à la run. |
| `users/` | Service de gestion des utilisateurs (officer). |
| `alerts/` | Dérivation d'alertes depuis runs/campagnes. |
| `zones/`, `posture/`, `mitigation/` | Zones, posture, panneaux mitigation (UI + logique). |

### Frontend (`frontend/src/`)
| Fichier | Rôle |
| --- | --- |
| `App.tsx` | Racine React : gate d'auth, routes (Command Center, Zones, Campaigns, Run Theater, Assistant, Settings, Account). |
| `api.ts` | Client API (fetch + SSE) vers le backend. |
| `context.tsx` | Contexte global de l'app (auth, identité, état). |
| `pages.tsx` | Pages principales : Command Center, Layout, Settings, Account. |
| `campaigns-workspace.tsx` | Workspace campagnes (création, approbation, exécution, suites). |
| `run-theater.tsx` | Vue live d'une run en cours (progression, métriques temps réel). |
| `assistant-page.tsx` | Page assistant conversationnel. |
| `zones-page.tsx` | Gestion des zones/cibles. |
| `wrs.ts` | Logique WRS côté client (affichage, niveaux). |
| `live-chart.tsx` | Graphiques live (SSE). |
| `ui.tsx` | Composants UI partagés (Login, ForcedPasswordChange…). |
| `storage.ts` | Persistance locale (tokens, préférences). |

## Flux principal

```
Frontend (React/Vite) ──► API Fastify ──► PostgreSQL (ou mémoire)
   login + RBAC              ├─ Auth (JWT) + RBAC
   dashboard + WRS           ├─ CAM  campagnes + approbation
                             ├─ GEN  générateur HTTP borné (allowlist + RPS cap)
                             ├─ PROBE sonde + kill switch (auto/manual/time-based)
                             ├─ WRS  scoring + history + régressions
                             ├─ RPT  rapport campagne + DORA/NIS2
                             ├─ Scheduler  exécution planifiée
                             ├─ Audit  log persistant
                             └─ /metrics  Prometheus
```

## Safeguards (non négociables)

- **Allowlist obligatoire** (`ALLOWED_TARGETS`) : toute cible hors liste est rejetée.
- **Attestation d'autorisation** : une campagne requiert `authorized: true`.
- **Caps immuables** (`HARD_CAPS`) : maxRps ≤ 2000, concurrency ≤ 500, duration ≤ 900 s.
- **Workflow d'approbation** : PENDING → APPROVED/REJECTED, separation of duties, validité 24 h.
- **Kill switch 3 niveaux** : auto (unavailable > 30 s), manuel (SOC/NOC), time-based.
- **Taggage du trafic** : header `X-WEDOS-TEST` sur chaque requête.
- **Deux couches de caps** : `HARD_CAPS` (plafond gelé) + limites opérationnelles (CISO, clamped ≤ plafond, auditées).

## Liens utiles

- Dépôt source : https://github.com/abdelhasoloji/ddostester
- Lab UDP isolé : https://github.com/abdelhasoloji/shieldtest-lab
- Docs : `docs/README.md` (index), `docs/architecture.md`, `docs/security-and-governance.md`, `docs/boundary-and-lab.md`.
