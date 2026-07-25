# Moodle Client

**ENSPY — Département du Génie Informatique**  
Année académique 2025–2026 | 4ème Année / Master 1   
UE : SMA et Projets SE  

---

## Table des matières

1. [À propos du projet](#1-à-propos-du-projet)
2. [Architecture](#2-architecture)
3. [Prérequis](#3-prérequis)
4. [Lancer le projet](#4-lancer-le-projet)
   - [Avec Docker](#41-avec-docker)
   - [Sans Docker — développement local](#42-sans-docker--développement-local)
5. [Variables d'environnement](#5-variables-denvironnement)
6. [Fonctionnement de la synchronisation](#6-fonctionnement-de-la-synchronisation)

---

## 1. À propos du projet

**Moodle Client** est une application de web/bureau offline-first qui synchronise votre environnement Moodle en local. Une fois la synchronisation effectuée, vous accédez à vos cours, devoirs, ressources et notes **sans connexion internet ou au serveur**.

Les données sont stockées sur la machine de l'utilisateur dans des bases SQLite individuelles. La synchronisation est bidirectionnelle : le contenu du cours descend du serveur, et les devoirs rédigés hors ligne remontent au serveur lors de la prochaine connexion.

L'application est destinée aux **étudiants** (consultation des cours, rendu de devoirs, suivi des notes) et aux **enseignants** (gestion des cours, notation des devoirs, gestion des participants).

---

## 2. Architecture

```
moodle-client-project/
├── backend/     API REST locale — Node.js 22, Express 5, Prisma 7, SQLite
├── frontend/    Interface utilisateur — React, Vite, TailwindCSS, shadcn/ui
└── docker-compose.yml
```

```
Serveur Moodle (existant)
       ↕  Web Services API
  Backend Express :5000
       ↕  HTTP / SSE
  Frontend React  :5173  (dev) | :80 (Docker)
       ↓
  SQLite local (une base par utilisateur)
```

Le backend gère deux bases SQLite distinctes :
- `master.db` — registre global des profils locaux
- `[email].db` — données Moodle propres à chaque utilisateur (cours, devoirs, notes…)

---

## 3. Prérequis

### Serveur Moodle (obligatoire)

Un serveur Moodle existant doit être accessible avec les Web Services configurés :

1. `Administration du site → Plugins → Web services → Activer les web services` ✅
2. `Administration du site → Plugins → Web services → Services externes` → activer **moodle_mobile_app** (ou créer un service personnalisé : option recommandée)
3. `Administration du site → Plugins → Authentification → Gérer l'authentification` → activer **Services web**

### Pour lancer avec Docker

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installé et lancé

### Pour lancer sans Docker

- Node.js 22+
- npm

---

## 4. Lancer le projet

### 4.1 Avec Docker

**Étape 1 — Configurer les variables Moodle dans `docker-compose.yml`**

Ouvrir `docker-compose.yml` et renseigner :
```yaml
MOODLE_URL: http://votre-serveur-moodle.fr   # URL de votre instance Moodle
MOODLE_SERVICE: moodle_mobile_app             # Nom du service Web Services
JWT_SECRET: une_chaine_aleatoire_longue       # Clé secrète pour les tokens JWT
```

**Étape 2 — Lancer**

```bash
docker compose up --build
```

L'application est disponible sur **http://localhost** une fois les containers démarrés.  
Le backend est accessible sur **http://localhost:5000**.

> Les bases SQLite sont persistées dans un volume Docker — vos données survivent aux redémarrages.

---

### 4.2 Sans Docker — développement local

#### Backend

```bash
cd backend

# 1. Configurer l'environnement
cp .env.example .env
# Éditer .env : renseigner MOODLE_URL, MOODLE_SERVICE, JWT_SECRET (obligatoires)

# 2. Installer les dépendances
npm install

# 3. Préparer les bases de données (à faire une seule fois)
npm run db:migrate:master    # base centrale des profils
npm run db:migrate:user      # schéma des bases utilisateurs
npm run db:generate          # génération des clients Prisma

# 4. Lancer
npm run dev
# → http://localhost:5000
```

#### Frontend

```bash
cd frontend

# 1. Configurer l'environnement
cp .env.example .env
# VITE_API_URL=http://127.0.0.1:5000/api/v1  (valeur par défaut, à conserver)

# 2. Installer et lancer
npm install
npm run dev
# → http://localhost:5173
```

#### Créer son premier profil

Une fois les deux serveurs lancés, ouvrir `http://localhost:5173` et cliquer sur **Créer un profil**. Fournir vos identifiants Moodle (username + mot de passe serveur) ainsi qu'un mot de passe local pour la connexion hors ligne. Une connexion internet est requise uniquement à cette étape.

---

## 5. Variables d'environnement

### Backend (`backend/.env`)

| Variable | Obligatoire | Description | Exemple |
|---|---|---|---|
| `NODE_ENV` | ✅ | Environnement d'exécution | `development` |
| `MOODLE_URL` | ✅ | URL du serveur Moodle, sans slash final | `http://moodle.univ.fr` |
| `MOODLE_SERVICE` | ✅ | Nom du service Web Services Moodle | `moodle_mobile_app` |
| `JWT_SECRET` | ✅ | Clé secrète pour les tokens locaux | chaîne longue aléatoire |
| `HOST` | ✅ | `true` pour exposer sur le réseau local | `false` |
| `CLIENT_PORT` | — | Port du backend (défaut : `5000`) | `5000` |
| `DATABASE_DIR` | — | Dossier des bases SQLite (défaut : `./data`) | `./data` |
| `JWT_EXPIRES_IN` | — | Durée de validité du token local | `1d` |
| `CORS_ORIGIN` | — | Origines autorisées (séparées par des virgules) | `http://localhost:5173` |

### Frontend (`frontend/.env`)

| Variable | Description |
|---|---|
| `VITE_API_URL` | URL complète de l'API backend — `http://127.0.0.1:5000/api/v1` |

---

## 6. Fonctionnement de la synchronisation

| Étape | Action | Connexion requise |
|---|---|---|
| 1 | S'inscrire à un cours depuis le catalogue | ✅ |
| 2 | Lancer la synchronisation — télécharge cours, devoirs, notes | ✅ |
| 3 | Consulter les cours, rédiger des devoirs, voir ses notes | ❌ hors ligne |
| 4 | Synchroniser à nouveau — envoie les devoirs rédigés hors ligne | ✅ |

La synchronisation est **incrémentale** : seules les données modifiées depuis la dernière sync sont transférées.
