# Loxis - Gestion Immobiliere

Ce projet implemente une solution de gestion immobiliere. L'architecture est modulaire (Domain-Driven Design) ou chaque domaine metier possede sa propre application Django dans le dossier `apps/`.

## Mise a jour (26/03/2026) - Reservations et Baux

- **Processus de Reservation complet** :
  - Un locataire peut deposer une demande de reservation avec un message et des dates souhaitees.
  - Le proprietaire peut accepter ou refuser la demande via son tableau de bord.
  - L'acceptation d'une reservation change automatiquement le statut du bien en `RESERVE`.
- **Creation automatique de Bail** :
  - Redirection intelligente apres acceptation vers un formulaire de bail pre-rempli avec les donnees de la reservation (ID bien, ID locataire, loyer, dates).
  - Validation du bail avec transition automatique du bien vers le statut `LOUE`.
- **Audit et Logs** :
  - Journalisation de chaque etape du cycle de vie de la reservation et du bail.
  - Alertes automatiques envoyees aux parties concernees lors d'une mise a jour de statut.

## Mise a jour rapide (17/03/2026)

- Core renforce : journal d'audit enrichi (severite, source, request id), alertes actionnables (priorite, URL d'action, metadata), scan operationnel, endpoints de marquage lu et marquage global.
- Leases corrige : services fiabilises (creation/resiliation/revision), audit via `core.services.log_audit`, filtrage role owner/tenant, routes root `/api/baux/...` + aliases legacy.
- Finances corrige : calcul des dettes (impaye + partiel), resume financier annuel, securite d'acces owner, journalisation et alertes impayes, correction generation PDF quittance.
- Tests ajoutes : couverture metier leases/finances + permissions API (6 tests OK).
- Nouvelle app `view3d` implementee : scene 3D liee au bien, publication/depublication, journal de visionnage, permissions admin/owner/tenant (bail actif requis), audit automatique, endpoints dedies.
- Tests `view3d` : 7 tests OK (creation, permissions, publication, journalisation).
- Migrations appliquees : `core`, `view3d`, plus alignement `BigAutoField` sur `users` et `properties`.

---

## Prerequis

- Python 3.12+
- pip (gestionnaire de paquets Python)

## Installation

```bash
# Cloner le projet
git clone <url-du-repo>
cd Loxis-projet-poo

# Installer les dependances
py -m pip install -r requirements.txt

# Appliquer les migrations (base SQLite)
py manage.py migrate

# Lancer le serveur de developpement
py manage.py runserver
```

## Base de donnees

Le projet utilise **SQLite** par defaut (fichier `db.sqlite3` a la racine).
La configuration se trouve dans `config/settings/base.py`.

---

## Documentation API (Swagger)

Une fois le serveur lance, la documentation interactive de l'API est accessible :

| URL | Description |
|-----|-------------|
| `http://127.0.0.1:8000/api/docs/` | Interface Swagger UI |
| `http://127.0.0.1:8000/api/redoc/` | Interface ReDoc |
| `http://127.0.0.1:8000/api/schema/` | Schema OpenAPI brut (JSON/YAML) |

La documentation est generee automatiquement par **drf-spectacular**.

---

## Roles utilisateurs

Le systeme gere 3 roles distincts :

| Role | Description |
|------|-------------|
| **ADMIN** | Administrateur - configuration globale, gestion des utilisateurs, journal d'audit |
| **OWNER** | Proprietaire - gestion du patrimoine, consultation des revenus et alertes |
| **TENANT** | Locataire - consultation de son bail, paiements et quittances |

Les permissions basees sur les roles sont definies dans `common/permissions.py`.

---

## Architecture des Applications (Dossier `apps/`)

### 1. `apps/users/` - Gestion des acces et utilisateurs

- **Entites** : `User` (modele personnalise), `TenantProfile`, `Guarantor`
- **Routes API** :
  - `POST/GET /api/utilisateurs/comptes/` - Lister ou creer un compte
  - `GET/PUT /api/utilisateurs/comptes/<id>/` - Detail d'un compte
  - `POST/GET /api/utilisateurs/locataires/` - Lister ou creer un dossier locataire
  - `POST /api/utilisateurs/locataires/<id>/garants/` - Ajouter un garant

### 2. `apps/properties/` - Gestion du patrimoine immobilier

- **Entites** : `PropertyCategory`, `PropertyType`, `Property`, `PropertyPhoto`
- **Logique** : Coherence des types/categories, gestion du statut (vacant/loue/en_travaux), upload de photos avec gestion de la photo principale.
- **Routes API** :
  - `POST/GET /api/immobilier/categories-biens/` - Lister ou creer des categories
  - `POST/GET /api/immobilier/types-biens/` - Lister ou creer des types
  - `POST/GET /api/immobilier/biens/` - Lister ou creer un bien
  - `GET/PUT /api/immobilier/biens/<id>/` - Detail d'un bien
  - `POST /api/immobilier/biens/<id>/statut/` - Changer le statut d'un bien
  - `POST/GET /api/immobilier/biens/<id>/photos/` - Upload et liste des photos

### 3. `apps/leases/` - Gestion locative et baux

- **Entites** : BAIL, REVISION DE LOYER
- **Logique** : Validation du chevauchement des dates des baux, creation/resiliation/revision avec audit.
- **Routes API** :
- `POST/GET /api/baux/` - Lister ou creer un bail
- `GET/PUT /api/baux/<id>/` - Detail ou modification d'un bail
- `POST /api/baux/<id>/resilier/` - Resilier un bail
- `POST/GET /api/baux/<lease_id>/revisions/` - Lister ou creer une revision de loyer

### 4. `apps/finances/` - Comptabilite et transactions

- **Entites** : PAIEMENT DE LOYER, QUITTANCE, CATEGORIE DE DEPENSE, DEPENSE
- **Logique** : Suivi des impayes, generation automatique des quittances, suivi des charges, resume financier annuel.
- **Routes API** :
- `POST/GET /api/finances/paiements/` - Lister ou creer un paiement de loyer
- `GET /api/finances/paiements/<id>/` - Detail d'un paiement
- `GET /api/finances/paiements/<id>/quittance/` - Telecharger la quittance de paiement
- `POST /api/finances/paiements/<id>/renvoyer/` - Renvoyer la quittance au locataire
- `GET /api/finances/impayes/` - Lister les loyers impayes
- `POST/GET /api/finances/depenses/` - Lister ou creer une depense
- `GET /api/finances/rapport/<property_id>/` - Generer le rapport financier d'un bien
- `GET /api/finances/export/` - Exporter les donnees financieres

### 5. `apps/core/` - Elements transverses et systeme

- **Entites** : `AuditLog`, `Alert`
- **Logique** : Notifications automatisees, tracabilite immuable des actions.
- **Routes API** :
  - `GET /api/systeme/journal-audit/` - Consulter le journal d'audit
  - `GET /api/systeme/alertes/` - Lister les alertes
  - `POST /api/systeme/alertes/<id>/marquer-lu/` - Marquer une alerte comme lue
  - `POST /api/systeme/alertes/marquer-tout-lu/` - Marquer toutes les alertes comme lues
  - `POST /api/systeme/alertes/scan-operationnel/` - Lancer le scan d'alertes metier

### 6. `apps/view3d/` - Visionnage 3D des biens

- **Entites** : `Scene3D`, `SceneViewLog`
- **Logique** : upload scene 3D, publication/depublication, controle d'acces par role et bail actif, journalisation des visionnages.
- **Routes API** :
  - `GET/POST /api/visites-3d/` - Lister ou creer une scene 3D
  - `GET /api/visites-3d/<id>/` - Consulter une scene 3D
  - `POST /api/visites-3d/<id>/publier/` - Publier une scene
  - `POST /api/visites-3d/<id>/depublier/` - Depublier une scene
  - `POST /api/visites-3d/<id>/journaliser-visionnage/` - Journaliser une consultation
  - `GET /api/visites-3d/health/` - Healthcheck de l'app

---

## Structure Interne de Chaque Application

| Fichier | Responsabilite |
|---------|----------------|
| `models.py` | **Donnees pures** - Definitions des tables, relations, regles d'unicite |
| `services.py` | **Logique Metier (Ecriture)** - Creation, mise a jour, suppression, appel au journal d'audit |
| `selectors.py` | **Requetes complexes (Lecture)** - Agregations, calculs dynamiques |
| `apis.py` | **Points d'entree reseau (Vues DRF)** - Exposition web, appelle services/selectors |
| `serializers.py` | Validation et formatage des donnees JSON |
| `urls.py` | Routage local de l'application |

---

## Dossier `common/` - Code partage

| Fichier | Description |
|---------|-------------|
| `permissions.py` | Permissions DRF basees sur les roles (`IsAdminRole`, `IsOwnerRole`, `IsTenantRole`) |
| `utils.py` | Gestionnaire d'exceptions personnalise pour formater les erreurs API |

---

## Dossier `config/` - Configuration Django

| Fichier | Description |
|---------|-------------|
| `settings/base.py` | Configuration commune (apps, middleware, DRF, Swagger, SQLite) |
| `settings/local.py` | Surcharges pour le developpement local |
| `settings/production.py` | Surcharges pour la production |
| `urls.py` | Routeur principal - regroupe toutes les routes API et Swagger |
| `wsgi.py` | Point d'entree WSGI pour le deploiement |

---

## Upload de photos

Les photos des biens immobiliers peuvent etre uploadees via l'endpoint :

```
POST /api/immobilier/biens/<id>/photos/
```

- Format : `multipart/form-data`
- Champs : `image` (fichier), `is_main` (booleen), `display_order` (entier)
- Les fichiers sont stockes dans le dossier `media/properties/photos/`
- Une seule photo principale est autorisee par bien

---

## Synthese Financiere

Aucune table n'existe pour la Synthese Financiere. Elle sera calculee a la volee dans `apps/finances/selectors.py` en agregeant les paiements de loyers et les depenses.
