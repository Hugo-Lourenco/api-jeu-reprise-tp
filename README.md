# API Catalogue de jeux

API REST en FastAPI pour gérer un catalogue de jeux vidéo et leurs éditeurs, avec des comptes utilisateurs et une authentification par jeton JWT. Elle sert de backend au front du catalogue.

## Prérequis

- Python 3.12 (la version de la CI ; fonctionne aussi en 3.14)
- Git

## Démarrage rapide

Le démarrage le plus court utilise une base SQLite locale : aucune base à installer.

```bash
git clone https://github.com/Hugo-Lourenco/api-jeu-reprise-tp.git
cd api-jeu-reprise-tp
python -m venv .venv
source .venv/bin/activate          # Windows : .venv\Scripts\activate
pip install -r requirements-dev.txt
cp .env.example .env               # Windows : copy .env.example .env
```

Ouvrez `.env` et modifiez ces trois lignes :

```ini
DATABASE_URL=sqlite:///./jeux.db
CLE_SECRETE=remplacez-moi
ORIGINES_AUTORISEES=["http://localhost:5173"]
```

- `CLE_SECRETE` : générez votre propre valeur avec `python -c "import secrets; print(secrets.token_urlsafe(32))"`.
- `ORIGINES_AUTORISEES` : gardez les crochets et les guillemets. Sans eux, l'API refuse de démarrer avec `SettingsError: error parsing value for field "origines_autorisees"`.

Remplissez la base avec le catalogue de démonstration, puis lancez l'API :

```bash
python scripts/peupler.py
fastapi dev app/main.py
```

`peupler.py` affiche `Jeux créés : 8`. Le serveur affiche ensuite `Uvicorn running on http://127.0.0.1:8000`. Ouvrez http://127.0.0.1:8000/docs : la liste des routes s'affiche.

## Configuration

Les variables sont lues dans `.env` par `app/config.py`. Sans une variable obligatoire, l'API refuse de démarrer avec un message explicite.

| Variable | Rôle | Obligatoire | Valeur par défaut |
|---|---|---|---|
| `DATABASE_URL` | Adresse de la base : `sqlite:///./jeux.db`, ou `postgresql+psycopg://utilisateur:motdepasse@hote:5432/base` | Oui | aucune |
| `CLE_SECRETE` | Clé de signature des jetons JWT. Ne jamais la versionner | Oui | aucune |
| `ALGORITHME_JETON` | Algorithme de signature des jetons | Non | `HS256` |
| `DUREE_JETON_MINUTES` | Durée de validité d'un jeton, en minutes | Non | `30` |
| `ORIGINES_AUTORISEES` | Origines autorisées par CORS, en liste JSON | Non | `["http://localhost:5173"]` |
| `ENVIRONNEMENT` | `developpement`, `test` ou `production` | Non | `developpement` |
| `NIVEAU_JOURNAL` | Niveau des journaux : `DEBUG`, `INFO`, `WARNING`… | Non | `INFO` |
| `ECHO_SQL` | Affiche les requêtes SQL dans le terminal | Non | `false` |
| `MAX_TENTATIVES_CONNEXION` | Échecs de connexion tolérés avant blocage | Non | `5` |
| `FENETRE_TENTATIVES_MINUTES` | Fenêtre de comptage des échecs de connexion, en minutes | Non | `15` |

## Utilisation

La documentation interactive de toutes les routes est générée par FastAPI : http://127.0.0.1:8000/docs. Les routes métier sont préfixées par `/api/v1`.

Lister les jeux de genre RPG, triés par note :

```bash
curl "http://127.0.0.1:8000/api/v1/jeux?genre=RPG&tri=note"
```

Résultat attendu : une page JSON avec `elements` (Undertale et Disco Elysium) et `"total":2`.

Obtenir les statistiques du catalogue :

```bash
curl http://127.0.0.1:8000/api/v1/jeux/statistiques
```

Résultat attendu : `{"nombre":8,"moyenne":8.5,"meilleure_note":10,"par_genre":{...}}`.

Se connecter avec le compte administrateur de démonstration créé par `scripts/peupler.py` :

```bash
curl -X POST http://127.0.0.1:8000/api/v1/connexion -d "username=admin@example.com&password=motdepasse123"
```

Résultat attendu : un JSON qui commence par `{"access_token":"eyJ...`. Le jeton se passe ensuite dans l'en-tête `Authorization: Bearer <jeton>` pour créer, modifier ou supprimer un jeu. Ce compte ne sert qu'en développement : `python scripts/peupler.py --mot-de-passe <autre>` en choisit un autre.

## Tests

```bash
pytest
ruff check .
```

Résultat attendu : tous les tests passent, et ruff affiche `All checks passed!`. Les tests utilisent une base SQLite en mémoire et ne touchent pas à votre `.env`. La CI (`.github/workflows/verifications.yml`) lance les deux commandes sur chaque pull request.

## Architecture

Une requête traverse les couches dans un seul sens : chaque couche n'appelle que celle du dessous.

```mermaid
flowchart LR
    Client -->|HTTP| Routeurs[app/routeurs]
    Routeurs --> Services[app/services]
    Services --> Depots[app/depots]
    Depots --> Tables[app/tables]
    Tables --> Base[(SQLite ou PostgreSQL)]
    Routeurs -.valide avec.-> Modeles[app/modeles]
    Services -.lève.-> Exceptions[app/exceptions.py]
    Exceptions -.traduites en HTTP par.-> Main[app/main.py]
```

| Dossier ou fichier | Rôle |
|---|---|
| `app/main.py` | Assemble l'application : middlewares, gestionnaires d'erreurs, routeurs |
| `app/routeurs/` | Les routes HTTP : reçoivent, délèguent au service, répondent |
| `app/services/` | La logique métier ; n'importe pas FastAPI et lève des exceptions métier |
| `app/depots/` | L'accès aux données : lit et écrit en base, ne décide de rien |
| `app/tables/` | Les tables SQLAlchemy : ce qui est stocké |
| `app/modeles/` | Les modèles Pydantic : ce qui entre dans l'API et en sort |
| `app/config.py` | La configuration, validée au démarrage |
| `scripts/` | Scripts en ligne de commande : peupler, importer, exporter la base |
| `tests/` | Les tests pytest |

## Contribuer

1. Ouvrez une issue qui décrit le bug ou la fonctionnalité.
2. Partez d'un `main` à jour et créez une branche `type/numero-description`, par exemple `fix/1-statistiques-catalogue-vide`.
3. Committez au format Conventional Commits (`fix(jeux): …`, `docs: …`), avec `Refs #numero`.
4. Ouvrez une pull request (contexte, changements, impact, comment tester, `Closes #numero`). Un autre membre la relit et l'approuve, et la CI doit être verte.
5. Fusionnez par **Squash and merge**, puis supprimez la branche. On ne pousse jamais directement sur `main`.
