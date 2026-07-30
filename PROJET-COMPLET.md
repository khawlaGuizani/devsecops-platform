# Transport — Description complète du projet

Document de référence unique décrivant **toutes** les fonctionnalités métier et **toute** l'architecture technique/DevSecOps du projet, tel qu'il existe aujourd'hui. Objectif : servir de contexte complet (prompt) pour présenter, expliquer ou reprendre le projet sans avoir à ouvrir tout le code.

---

## 1. Présentation

**Transport** est une application de gestion logistique : des utilisateurs (**demandeurs**) créent des demandes de mouvement de marchandises (entrée/sortie de stock) entre deux sites, via un camion et un fournisseur donnés ; ces demandes sont ensuite **validées ou rejetées** par un validateur/admin, ce qui met à jour le stock, génère un rapport CSV et notifie le demandeur par email.

- **Backend** : Spring Boot 4 / Java 17, API REST, JWT, SQL Server
- **Frontend** : Angular 21 SPA
- **Infrastructure** : Docker Compose (SQL Server + backend + frontend), pipeline CI/CD DevSecOps complète, déploiement automatique sur serveur privé via runner GitHub auto-hébergé

Trois dépôts GitHub séparés : `Backend`, `Frontend`, `devsecops-platform` (orchestration/déploiement uniquement).

---

## 2. Fonctionnalités métier

### 2.1 Rôles utilisateurs

| Rôle | Peut faire |
|---|---|
| **DEMANDEUR** | Créer des demandes, consulter ses propres demandes |
| **VALIDATEUR** | Consulter les demandes en attente, valider/rejeter (avec motif), télécharger le CSV |
| **ADMIN** | Tout ce que fait VALIDATEUR + gérer les données de référence (sites, camions, types de camion, fournisseurs, articles, utilisateurs) + créer de nouveaux comptes |

### 2.2 Cycle de vie d'une demande

1. **Création** (DEMANDEUR) : choix site de départ, site de destination, fournisseur, camion, et une ou plusieurs lignes (article + quantité + type ENTREE/SORTIE) → statut `EN_ATTENTE`.
2. **Traitement** (VALIDATEUR/ADMIN) :
   - **Validation** : vérifie automatiquement les règles métier (lignes non vides, quantités > 0, capacité du camion suffisante, site de départ ≠ site d'arrivée, stock suffisant pour les sorties) — si une règle échoue, la demande est **automatiquement rejetée** avec le motif de l'échec. Sinon : le stock de chaque article est mis à jour (+/- selon ENTREE/SORTIE), statut → `VALIDE`, un **CSV** est généré (date de validation, site départ, site destination, article, quantité) et téléchargé automatiquement côté frontend, un **email de notification détaillé** part au demandeur.
   - **Rejet manuel** : motif obligatoire, statut → `REJETE`, email de notification avec le motif.
   - Dans les deux cas, un échec d'envoi d'email ne bloque jamais l'opération (best-effort, loggé en warning).

### 2.3 Gestion des données de référence (ADMIN uniquement côté UI)

CRUD complet pour : **Sites**, **Camions** (rattachés à un **Type de camion**), **Fournisseurs**, **Articles** (qui portent aussi le niveau de stock), **Utilisateurs** (création de comptes avec rôle, réinitialisation de mot de passe).

### 2.4 Authentification

Inscription (`/register`) et connexion (`/login`) par email/mot de passe (BCrypt), JWT contenant email + rôle, vérifié à chaque requête protégée par un filtre (`JwtFilter`).

---

## 3. Modèle de données

| Entité | Champs principaux |
|---|---|
| `Utilisateur` | id, nom, email, motDePasse (hashé), role, createdAt |
| `Demande` | id, libelle, capacite, dateDemande, dateValidation, statut, typeMouvement, siteDepart, siteArrivee, demandeur, fournisseur, camion, lignes[] |
| `LigneDemande` | id, quantite, unite, article, type (ENTREE/SORTIE), description |
| `Article` | id, codeArticle, unit, quantite (= stock actuel) |
| `Site` | id, codeSite, libelle, adresse, ville, actif |
| `Fournisseur` | id, nom, contact, email, actif |
| `Camion` | id, immatriculation, disponible, capaciteReelle, annee, typeCamion |
| `TypeCamion` | id, libelle, capaciteMax, description |

**Enums** : `Role` {ADMIN, DEMANDEUR, VALIDATEUR} · `StatutDemande` {EN_ATTENTE, VALIDE, REJETE} · `TypeMouvement` {ENTREE, SORTIE}

---

## 4. API REST (backend)

| Contrôleur | Endpoints |
|---|---|
| `AuthController` `/api/auth` | POST `/register`, POST `/login` (publics) |
| `DemandeController` `/api/demandes` | POST `` (DEMANDEUR) · GET `/mes-demandes` (DEMANDEUR) · GET `/en-attente`, `/all?statut=` (VALIDATEUR/ADMIN) · PUT `/{id}/valider`, `/{id}/rejeter` (VALIDATEUR/ADMIN) · GET `/{id}/csv` (VALIDATEUR/ADMIN) |
| `ArticleController` `/api/articles` | CRUD — GET (ADMIN, DEMANDEUR), POST/PUT/DELETE (ADMIN) |
| `SiteController`, `CamionController`, `FournisseurController`, `TypeCamionController`, `UtilisateurController` | CRUD complet chacun — **protégés uniquement par "utilisateur connecté"**, pas de restriction de rôle au niveau API (voir §7, point d'attention) |

---

## 5. Frontend (Angular)

| Route | Accès | Contenu |
|---|---|---|
| `/` | public | Page d'accueil |
| `/login`, `/register` | public / ADMIN | Connexion / création de compte |
| `/demandes` | DEMANDEUR, VALIDATEUR, ADMIN | Vue différente selon le rôle : formulaire de création (demandeur) ou listes en attente/traitées + actions valider/rejeter (validateur/admin) |
| `/admin` | ADMIN | Tableau de bord à onglets : Utilisateurs, Articles, Types Camion, Camions, Fournisseurs, Sites |

---

## 6. Architecture technique & infrastructure

### 6.1 Conteneurisation (`devsecops-platform/docker-compose.yml`)

```
sqlserver (SQL Server 2022, volume mssql-data)
   └─ db-init (tâche unique : CREATE DATABASE vanoiseee si absente)
        └─ backend (Spring Boot, volume rasp-logs, port 8080)
             └─ frontend (nginx + Angular, port 8081, proxy /api/ → backend)
```
- `docker-compose.yml` : **production**, image-only (aucun `build:`), déployable tel quel.
- `docker-compose.override.yml` : **dev local uniquement**, ajoute les `build:` — auto-fusionné par Compose, jamais déployé.

### 6.2 Sécurité applicative

- JWT (rôle inclus dans le token)
- BCrypt pour les mots de passe
- **RASP maison** (Runtime Application Self-Protection) : 4 couches indépendantes — filtre HTTP (signatures SQLi/XSS/path traversal/command injection/XXE/LDAP/expression injection), garde JDBC (inspecte le SQL généré), filtre de désérialisation JVM-wide, garde SSRF sur les appels sortants. Logs NDJSON dans `rasp-logs/`. Détail complet dans `README-DEVSECOPS.md` §8.

### 6.3 Pipeline CI/CD DevSecOps (Backend et Frontend, indépendantes)

Build/tests → SonarCloud → Gitleaks (bloquant) → OWASP Dependency-Check / npm audit → Semgrep → CodeQL → License scan → Docker build + Trivy/Syft/Grype (dépend de Gitleaks) → **Security Gate** (bloque si build/tests ou Gitleaks échouent) → si vert sur `main` : déclenche automatiquement le déploiement.

### 6.4 Déploiement automatique

- Images publiées sur **GHCR** (`ghcr.io/khawlaguizani/{backend,frontend}`) uniquement après passage de la Security Gate.
- `devsecops-platform/deploy.yml` s'exécute sur un **runner GitHub Actions auto-hébergé installé directement sur le serveur cible** (`vm1`, réseau privé — inaccessible depuis les runners cloud, d'où ce choix plutôt que du SSH classique) : `docker compose pull && up -d`, jamais de build côté serveur.
- Déclenchement automatique via `repository_dispatch` (`security-gate-passed`) envoyé par Backend/Frontend une fois la Security Gate passée.

---

## 7. Points d'attention connus (non corrigés, à trancher plus tard)

- **Gap d'autorisation API** : `SiteController`, `CamionController`, `FournisseurController`, `TypeCamionController`, `UtilisateurController` n'ont aucune annotation `@PreAuthorize` — un DEMANDEUR authentifié pourrait appeler ces endpoints CRUD directement (l'UI les cache, mais l'API ne les protège pas par rôle). Seul `ArticleController` restreint par rôle. À corriger si l'exigence métier est que ces opérations restent réservées à l'ADMIN.
- `EmailService` a l'adresse d'expéditeur codée en dur (`k99959510@gmail.com`) plutôt que dérivée de `MAIL_USERNAME`.
- Pas de `@ControllerAdvice` global : les erreurs métier passent par `ResponseStatusException`/`RuntimeException` bruts selon les contrôleurs.

---

## 8. Documents complémentaires dans ce dépôt

- `README-DEVSECOPS.md` — détail de chaque outil de sécurité, secrets requis, RASP en profondeur
- `GUIDE-DEMARRAGE-ET-TESTS.md` — comment démarrer, déployer et tester, avec l'historique complet des 15 bugs corrigés lors de la remise en ordre du déploiement
