# Distrilog — architecture microservices

Projet du module **ARCE801 · Architectures applicatives modernes et microservices** (EPSI).

Distrilog est une application e-commerce extraite d'un monolithe (GESCOM) et découpée en **4 microservices Python (FastAPI)**, chacun avec **sa propre base de données**, qui communiquent en **HTTP** (synchrone) et par **messages RabbitMQ** (asynchrone). L'ensemble tourne dans **Docker** (9 conteneurs).

## Architecture

| Service | Port | Rôle | Base |
|---|---|---|---|
| Catalogue | 8001 | produits et prix | `pg-catalogue` |
| Stock | 8002 | quantités, réservation atomique | `pg-stock` |
| Commande | 8003 | orchestre Catalogue + Stock, publie l'événement | `pg-commande` |
| Livraison | 8004 | écoute RabbitMQ, crée les bons de livraison | `pg-livraison` |
| RabbitMQ | 5672 / 15672 | broker de messages (exchange `distrilog.events`) | — |

```
Client ──POST /commandes──► Commande
                              ├─► Catalogue  GET /produits/{sku}        (HTTP, on attend)
                              ├─► Stock      POST /stock/{sku}/reserve  (HTTP, on attend)
                              ├─► enregistre la commande → 201
                              └─► RabbitMQ   « commande.passee »         (message, on n'attend pas)
                                                 └─► Livraison crée le bon de livraison
```

## Décisions d'architecture

1. **Un service par métier**, jamais par technique (Catalogue ≠ Stock, reliés par le `sku`).
2. **Chaque service possède sa base** : aucun service ne lit la base d'un autre, il passe par son API.
3. **Besoin d'une réponse immédiate → appel HTTP** (prix, réservation du stock).
4. **Pas besoin d'attendre → événement RabbitMQ** (livraison) : découplage, la commande réussit même si Livraison est en panne.
5. **Un service va tomber** : timeout (2 s), 3 essais avec attente croissante (tenacity), disjoncteur (pybreaker : ouvert 10 s après 3 échecs), réponse **503** claire.

Points techniques notables :
- **Réservation atomique** du stock (`UPDATE … WHERE quantite >= demandée`) : pas de survente en cas d'accès concurrents ; 0 ligne modifiée → **409**.
- **Idempotence** côté Livraison : chaque événement porte un `event_id` (UUID) qui est la clé primaire de la table `bons` ; un message reçu deux fois est ignoré.
- **Configuration par variables d'environnement** (`DATABASE_URL`, `CATALOGUE_URL`, `STOCK_URL`, `RABBIT_HOST`) : le même code tourne en local (SQLite, `localhost`) et dans Docker (PostgreSQL, noms de services).

## Lancer le projet

### Avec Docker (recommandé)

```bash
docker compose up -d --build   # construit et lance les 9 conteneurs
docker compose ps              # vérifie qu'ils tournent
docker compose logs -f commande  # logs d'un service
docker compose down            # arrête (données conservées)
```

- APIs : http://localhost:8001/docs … http://localhost:8004/docs
- RabbitMQ : http://localhost:15672 (guest / guest)

### En local (sans Docker)

```bash
python3.12 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
docker run -d --name rabbit -p 5672:5672 -p 15672:15672 rabbitmq:3-management
# puis, dans 4 terminaux (un par service) :
cd catalogue && uvicorn main:app --port 8001 --reload
cd stock     && uvicorn main:app --port 8002 --reload
cd livraison && uvicorn main:app --port 8004 --reload   # avant la 1re commande
cd commande  && uvicorn main:app --port 8003 --reload
```

## Scénario de test (Postman)

| # | Requête | Corps | Attendu |
|---|---|---|---|
| 1 | `POST :8001/produits` | `{"sku":"PAL-01","nom":"Palette eau","prix":12.5}` | 201 |
| 2 | `PUT :8002/stock/PAL-01` | `{"quantite":10}` | 200 |
| 3 | `POST :8003/commandes` | `{"sku":"PAL-01","quantite":2}` | 201, total 25.0 |
| 4 | `GET :8004/livraisons` | — | 200 + 1 bon |
| 5 | `POST :8003/commandes` | `{"sku":"PAL-01","quantite":999}` | 409 |
| 6 | `POST :8003/commandes` | `{"sku":"NOPE","quantite":1}` | 404 |

**Panne** : arrêter Stock (`docker compose stop stock`) puis refaire la requête 3 → **503** rapide, puis instantané (disjoncteur ouvert) ; après redémarrage et 10 s → 201.

## Limites connues

| Problème | Conséquence | Solution possible |
|---|---|---|
| Événement perdu | commande enregistrée mais RabbitMQ en panne : le message n'est jamais publié | Pattern **Outbox** |
| Réservation orpheline | stock réservé puis l'enregistrement de la commande échoue | **Compensation / Saga** |
| Aucune authentification | les services se font confiance sans vérification | **Jetons entre services** (JWT, mTLS) |

## Stack

Python 3.12 · FastAPI · Pydantic · SQLAlchemy · httpx · pika · tenacity · pybreaker · PostgreSQL 16 · RabbitMQ 3 · Docker Compose
