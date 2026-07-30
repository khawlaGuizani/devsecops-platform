# Guide — Démarrage, Déploiement et Tests

Ce document résume tout ce qui a été fait pour remettre en ordre le déploiement Docker/DevSecOps du projet **Transport** (Spring Boot + Angular + SQL Server), comment démarrer la stack, comment le déploiement automatique fonctionne, et comment tester chaque partie. Il complète `README-DEVSECOPS.md` (qui documente en détail chaque outil de sécurité de la pipeline CI).

---

## 1. Architecture finale

```
pfe-khawla/
├── backend/                     # Spring Boot API — repo GitHub "Backend"
├── frontend/                    # Angular SPA — repo GitHub "Frontend"
├── devsecops-platform/          # Orchestration — repo GitHub "devsecops-platform"
│   ├── docker-compose.yml       # PRODUCTION — image-only, aucun build:
│   ├── docker-compose.override.yml  # DEV LOCAL — ajoute build: pour backend/frontend
│   ├── .env.example / .env
│   └── .github/workflows/deploy.yml
└── actions-runner/               # Runner GitHub Actions auto-hébergé (hors dépôt git)
```

**Les 3 services orchestrés par `docker-compose.yml`** :

| Service | Rôle |
|---|---|
| `sqlserver` | SQL Server 2022, volume persistant `mssql-data` |
| `db-init` | Tâche unique : crée la base `vanoiseee` si elle n'existe pas |
| `backend` | Spring Boot, dépend de `db-init` (terminé avec succès) |
| `frontend` | Angular servi par nginx, dépend de `backend` (healthy), proxy `/api/*` |

**Flux CI/CD complet** :

```
push sur main (Backend ou Frontend)
   → build, tests, SonarCloud, Gitleaks, Dependency-Check, Semgrep, Trivy/Syft/Grype
   → Security Gate (bloque si build/tests ou Gitleaks échouent)
   → si Gate vert sur main : repository_dispatch vers devsecops-platform
        → deploy.yml s'exécute SUR vm1 (runner auto-hébergé, pas de SSH)
        → docker compose pull && up -d
        → smoke-test /actuator/health
```

**Pourquoi un runner auto-hébergé et pas du SSH classique ?**
`vm1` (`192.168.205.150`) est sur un réseau privé, injoignable depuis les runners cloud de GitHub. Le runner installé directement sur `vm1` va chercher les jobs sur GitHub au lieu que GitHub vienne le contacter — aucune IP publique ni port ouvert n'est nécessaire.

---

## 2. Tout ce qui a été trouvé et corrigé

Ordre chronologique des problèmes diagnostiqués et corrigés pendant cette session :

| # | Problème | Cause | Correction |
|---|---|---|---|
| 1 | `docker-compose.yml` n'avait ni `sqlserver` ni `db-init` alors que le backend s'y connectait déjà | Services supprimés du fichier lors d'une précédente tentative, mais conteneurs laissés tourner manuellement | Services `sqlserver` + `db-init` réintégrés, avec volume persistant `mssql-data` |
| 2 | Le conteneur `sqlserver` existant n'avait **aucun volume** — toutes les données auraient été perdues à la moindre recréation | Conteneur démarré à la main (`docker run`), jamais via compose avec volume | Recréé proprement via compose avec `mssql-data:/var/opt/mssql` |
| 3 | `build:` pointant vers `../backend`/`../frontend` dans le même fichier que celui déployé sur le serveur | Ces dossiers n'existent pas sur la cible de déploiement | Séparation en `docker-compose.yml` (image-only) + `docker-compose.override.yml` (build, dev local uniquement) |
| 4 | Pas de `.dockerignore` sur le frontend → `node_modules` de l'hôte pouvait écraser celui installé dans le conteneur | `COPY . .` après `npm ci` copiait tout, y compris `node_modules` local | `.dockerignore` ajouté (backend + frontend) |
| 5 | Service `db-init` échouait (`Login timeout`, connexion à son propre hostname au lieu de `sqlserver`) | Piège YAML : `entrypoint: bash -c` + `command:` en chaîne pliée → seul le 1er mot était exécuté, le reste devenait des `$0 $1 ...` ignorés | `command:` réécrit en liste avec un seul élément (chaîne complète) |
| 6 | Le `HEALTHCHECK` du frontend échouait en permanence (`Connection refused`) alors que l'appli fonctionnait | nginx n'écoute qu'en IPv4 (`0.0.0.0:8080`), mais `wget http://localhost/...` résolvait `localhost` en IPv6 (`::1`) en premier | Healthcheck pointé sur `127.0.0.1` au lieu de `localhost` (Dockerfile + docker-compose.yml) |
| 7 | **Le vrai bug bloquant** : toute requête `/api/xxx` renvoyait 403 | `nginx.conf` : `proxy_pass $backend_upstream/api/;` — avec une **variable**, nginx n'ajoute plus le reste du chemin demandé ; toute requête arrivait au backend comme `/api/` nu, refusé par Spring Security (403 par défaut, pas de `formLogin` configuré) | `proxy_pass $backend_upstream$request_uri;` — transmet le chemin complet |
| 8 | Login échouait avec 401 | Base de données neuve, aucun compte utilisateur | Passage par `/register` avant `/login` |
| 9 | Rejeter/valider une demande plantait avec 500 | `DemandeService.valider()`/`refuser()` laissaient une erreur d'envoi d'email (SMTP sans identifiants) faire échouer toute l'action métier | `try/catch` autour de `emailService.sendEmail()` — échec loggé en `WARN`, l'action métier réussit quand même |
| 10 | Email/CSV de validation trop génériques | — | CSV et email enrichis : date de validation, site de départ/destination, article(s) + quantité(s) |
| 11 | Une image pouvait être poussée sur GHCR même si Gitleaks échouait en parallèle | `docker-image-security` ne dépendait que de `build-test-sonar`, pas de `gitleaks` | `needs: [build-test-sonar, gitleaks]` |
| 12 | Aucun déclenchement automatique du déploiement après la Security Gate | Étape documentée dans le README mais jamais ajoutée au workflow | Étape "Trigger deployment" ajoutée à `security-gate` (repository_dispatch) |
| 13 | Le workflow était rejeté par GitHub (`Unrecognized named-value: 'secrets'`) | `secrets` n'est pas utilisable directement dans un `if:` | Passage par `env:` au niveau de l'étape, test sur `env.DISPATCH_TOKEN` |
| 14 | Le job "Trigger deployment" échouait (401) avant que le secret existe, faisant échouer tout `security-gate` | `curl -f` sans garde | Condition `&& env.DISPATCH_TOKEN != ''` : l'étape est ignorée tant que le secret n'existe pas |
| 15 | `deploy.yml` (SSH) n'aurait jamais pu joindre `vm1` | IP privée (`192.168.205.150`), inaccessible depuis les runners cloud GitHub | Runner auto-hébergé installé sur `vm1`, `deploy.yml` réécrit sans SSH |

---

## 3. Comment démarrer le projet

### 3.1 En local (développement, avec build)

```bash
cd devsecops-platform
cp .env.example .env      # si pas déjà fait — éditer MSSQL_SA_PASSWORD, MAIL_USERNAME/PASSWORD
docker compose up -d --build
docker compose ps          # attendre que les 3 services soient "healthy"
```

`docker-compose.override.yml` est automatiquement fusionné par Compose (aucun `-f` nécessaire) : `backend`/`frontend` sont reconstruits depuis `../backend`/`../frontend`.

### 3.2 En production (sur `vm1`, via le runner auto-hébergé)

Rien à faire manuellement — c'est le but du CI/CD :

- **Automatique** : un push sur `main` dans `Backend` ou `Frontend`, une fois la Security Gate passée, déclenche `devsecops-platform/deploy.yml` tout seul.
- **Manuel** : GitHub → dépôt `devsecops-platform` → onglet **Actions** → workflow **Deploy (self-hosted runner)** → **Run workflow**.

Le job s'exécute directement sur `vm1` (le runner y est installé en service systemd) : `docker compose -f docker-compose.yml pull && up -d --remove-orphans` — jamais de build, uniquement des images déjà publiées sur GHCR.

### 3.3 Vérifier que le runner est actif

```bash
sudo systemctl status actions.runner.*
```
ou
```bash
cd ~/pfe-khawla/actions-runner && sudo ./svc.sh status
```

---

## 4. Comment tout tester

### 4.1 Santé des conteneurs

```bash
cd devsecops-platform
docker compose ps
```
Les 3 services doivent afficher `healthy`.

```bash
curl http://localhost:8080/actuator/health   # → {"status":"UP"}
curl -o /dev/null -w "%{http_code}\n" http://localhost:8081/   # → 200
```

### 4.2 Persistance des données SQL Server

```bash
docker compose restart sqlserver
PW=$(grep MSSQL_SA_PASSWORD .env | cut -d= -f2)
docker exec sqlserver /opt/mssql-tools18/bin/sqlcmd -C -S localhost -U sa -P "$PW" \
  -Q "SELECT name FROM sys.databases WHERE name='vanoiseee'"
```
La base doit toujours exister après le redémarrage (volume persistant).

### 4.3 Chaîne complète Angular → nginx → Spring Boot → SQL Server

1. Ouvrir `http://192.168.205.150:8081/register` → créer un compte (rôle `ADMIN` ou `DEMANDEUR`)
2. Se connecter sur `/login`
3. En tant que `DEMANDEUR` : créer une demande
4. En tant que `ADMIN`/`VALIDATEUR` : la valider → le CSV se télécharge automatiquement, un email de notification part au demandeur

Vérification en ligne de commande équivalente :
```bash
# inscription
curl -X POST http://localhost:8081/api/auth/register -H "Content-Type: application/json" \
  -d '{"nom":"Test","email":"test@test.com","motDePasse":"Test1234!","role":"ADMIN"}'

# connexion
TOKEN=$(curl -s -X POST http://localhost:8081/api/auth/login -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","motDePasse":"Test1234!"}' | python3 -c "import json,sys;print(json.load(sys.stdin)['token'])")

# validation d'une demande existante (id à adapter)
curl -X PUT http://localhost:8081/api/demandes/1/valider -H "Authorization: Bearer $TOKEN"

# téléchargement du CSV généré
curl http://localhost:8081/api/demandes/1/csv -H "Authorization: Bearer $TOKEN"
```

### 4.4 Vérifier qu'aucun build n'est possible côté déploiement

```bash
docker compose -f docker-compose.yml config | grep -c "build:"   # doit afficher 0
```

### 4.5 Vérifier la pipeline CI/CD

Sur GitHub, dans `Backend` ou `Frontend` → onglet **Actions** → dernier run sur `main` :
- Tous les jobs doivent être verts (`Build, Test & SonarCloud`, `Secret Scan (Gitleaks)`, `Dependency Scan`, `SAST`, `License Scan`, `Docker Build & Image Security`, `Security Gate`)
- `Docker Build & Image Security` doit attendre que `Secret Scan (Gitleaks)` soit terminé (dépendance ajoutée au point 11 du tableau ci-dessus)

### 4.6 Vérifier le déclenchement automatique du déploiement

Une fois les secrets `DISPATCH_TOKEN` (Backend + Frontend) et `MSSQL_SA_PASSWORD`/`MAIL_USERNAME`/`MAIL_PASSWORD` (devsecops-platform) configurés :
1. Faire un petit commit/push sur `Backend` ou `Frontend` (`main`)
2. Vérifier que `security-gate` déclenche bien l'étape "Trigger deployment"
3. Sur `devsecops-platform` → Actions → le workflow **Deploy (self-hosted runner)** doit démarrer automatiquement (déclenché par `repository_dispatch`)
4. Vérifier `docker compose ps` sur `vm1` : les conteneurs doivent avoir été recréés avec les images fraîchement pullées

---

## 5. Secrets GitHub requis (récapitulatif)

| Dépôt | Secret | Utilisé pour |
|---|---|---|
| Backend, Frontend | `SONAR_TOKEN` (optionnel) | Analyse SonarCloud |
| Backend | `NVD_API_KEY` (optionnel) | Accélère OWASP Dependency-Check |
| Backend, Frontend | `DISPATCH_TOKEN` | Déclenche le déploiement automatique après la Security Gate |
| devsecops-platform | `MSSQL_SA_PASSWORD` | Écrit dans le `.env` du déploiement |
| devsecops-platform | `MAIL_USERNAME` / `MAIL_PASSWORD` | Notifications email de validation/rejet |
| — | `GITHUB_TOKEN` | Fourni automatiquement, push GHCR + upload SARIF |

---

## 6. Checklist finale

- [ ] `docker compose ps` → 3 services `healthy` en local
- [ ] Inscription + connexion fonctionnent (pas de 403/401 inattendu)
- [ ] Validation d'une demande → CSV téléchargé + email reçu (vérifier aussi le dossier Spam)
- [ ] Rejet d'une demande fonctionne même sans configuration email
- [ ] `docker compose restart sqlserver` → données toujours présentes
- [ ] Runner auto-hébergé actif sur `vm1` (`sudo ./svc.sh status`)
- [ ] Secrets configurés dans les 3 dépôts GitHub
- [ ] Un push sur `main` (Backend/Frontend) déclenche tout le pipeline jusqu'au déploiement automatique
