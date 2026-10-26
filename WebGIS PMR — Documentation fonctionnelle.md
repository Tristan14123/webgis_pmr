# WebGIS PMR — Documentation fonctionnelle

ceci est une v1 du Oct 11, 2026 · @tristan
La v2 est sur webgis_pmr

Application WebGIS de planification de déplacements accessibles pour l'agglomération fictive **Losse-en-Gelaisse**, conforme au standard CNIG Accessibilité (schéma AOM/LOM).

## Présentation générale

Losse-en-Gelaisse est une agglomération fictive jouant le rôle d'une AOM (Autorité Organisatrice de la Mobilité) dans le cadre d'un démonstrateur technique. L'application répond aux obligations de la loi LOM (2019) qui impose aux AOM la mise à disposition d'un plan de déplacements accessibles aux personnes à mobilité réduite (PMR).

L'outil permet à un usager ou un agent de mobilité de :

- Calculer un **itinéraire piéton PMR** en tenant compte des contraintes réelles du trottoir (pente, dévers, largeur, obstacles temporaires)
- Visualiser les **couches d'accessibilité** conformément au standard CNIG Accessibilité (tronçons OK / dégradés / interdits)
- Consulter et filtrer les **chantiers en cours** qui modifient le cheminement
- Calculer des **isochrones d'accessibilité** à 15 minutes depuis les arrêts de transport prioritaires

Le standard de données sous-jacent est le **schéma CNIG Accessibilité** (profil AOM/LOM), publié par le Conseil National de l'Information Géolocalisée, qui normalise la description de la voirie, des équipements PMR et des discontinuités du réseau piéton.

## Architecture technique

L'application est organisée en trois couches indépendantes.

&#91;embedded content: architecture · 3 couches, 4 schémas PostGIS\]

**Navigateur** — une page HTML autonome (`webgis-pmr-leaflet.html`) sans build step ni bundler. Leaflet 1.9.4 est chargé depuis cdnjs ; sa CSS est inlinée pour contourner la CSP de l'hébergeur d'artefacts. La carte occupe tout l'écran (`position: absolute`) ; la sidebar de 320 px se superpose à gauche.

**Backend (optionnel en démo)** — un micro-service FastAPI en Python expose quatre routes REST qui appellent les fonctions pgRouting stockées en base. En mode démo, ces appels sont simulés par des GeoJSON mock directement dans le JS côté client.

**Base de données** — PostgreSQL 15 + PostGIS 3 + pgRouting 3. Les données sont réparties en quatre schémas métier (voir section suivante). La vue matérialisée `geostandard.mv_graphe_pmr` précalcule les coûts PMR pour que Dijkstra n'ait pas à recalculer les pénalités à chaque requête.

| Composant | Version | Rôle |
| --- | --- | --- |
| PostgreSQL | 15 | SGBD relationnel |
| PostGIS | 3.4 | Géométries et index spatiaux |
| pgRouting | 3.5 | Dijkstra + pgr\_drivingDistance |
| FastAPI | 0.110 | API REST asynchrone |
| Leaflet | 1.9.4 | Rendu cartographique navigateur |
| Inter (Google Fonts) | — | Police d'interface |

## Modèle de données

La base est divisée en quatre schémas PostGIS à responsabilités séparées.

### schéma `geostandard` (référence CNIG)

Contient les objets du standard CNIG Accessibilité. Les tables principales :

| Table | Description |
| --- | --- |
| `troncon_cheminement` | Segments de voirie avec `pente`, `devers`, `largeur_utile`, `revetement` |
| `equipement_pmr` | Rampes, bateaux de trottoir, passages piétons abaissés |
| `arret_prioritaire` | Arrêts de TC classifiés accessibles (liaison avec schéma `transport`) |
| `stationnement_pmr` | Places GIC/GIG géolocées |
| `obstacle` | Obstacles permanents (bennes, plots, poteaux) |

La **vue matérialisée** `mv_graphe_pmr` aplatit ces tables en un graphe orienté pour pgRouting. Chaque arête porte un coût en **mètres équivalents** calculé ainsi :

```
cost = longueur
     × pente_factor          -- 1.0 si ≤5%, 3.0 si >5%
     + devers_penalty         -- +20 m si dévers >2%
     + largeur_penalty        -- prohibitif si largeur_utile <0.9 m
     + travaux_penalty        -- prohibitif si chantier actif
```

### schéma `transport` (GTFS)

Import du flux GTFS de l'agglomération : `stops`, `routes`, `trips`, `stop_times`. Utilisé pour lier les arrêts prioritaires aux isochrones.

### schéma `travaux` (chantiers temporels)

Table `chantier` avec `date_debut`, `date_fin`, `geom` (polygone ou ligne) et `impact_niveau` (1 = information, 2 = déviation, 3 = blocage). La fonction `fn_troncons_travaux(date_debut, date_fin)` joint ce schéma au graphe pour activer les pénalités temporelles.

### schéma `qualite` (retours terrain)

Photos géotagguées et signalements de discontinuité remontés par les agents de mobilité. Alimente les alertes affichées dans les résultats d'itinéraire.

## Modules fonctionnels

L'interface est divisée en quatre onglets dans la sidebar.

### Onglet Itinéraire

L'usager pose deux marqueurs A (départ) et B (arrivée) en cliquant sur la carte, ou en saisissant des coordonnées. Il sélectionne un **profil de mobilité** :

| Code | Profil | Paramètres activés |
| --- | --- | --- |
| FRE | Fauteuil roulant électrique | Pente max 5%, largeur min 0.9 m, évite dévers |
| FRM | Fauteuil roulant manuel | Pente max 8%, largeur min 1.0 m |
| PMR | Piéton PMR | Pente max 12%, trottoir min 0.6 m |
| CAN | Canne / béquille | Qualité de revêtement prioritaire |

L'appel `POST /api/itineraire` renvoie un GeoJSON `FeatureCollection` de tronçons ordonnés. Chaque feature porte `niveau` (`ok`/`degrade`/`ko`) et un tableau `alertes` (obstacle, chantier, dévers, etc.). La route est affichée avec la sémiologie CNIG et une liste de précautions est affichée sous le tracé.

En base, la fonction `geostandard.fn_itineraire_pmr(lon_dep, lat_dep, lon_arr, lat_arr)` appelle `pgr_dijkstra` sur `mv_graphe_pmr`, récupère les tronçons ordonnés avec leurs coûts et génère les textes d'alerte.

### Onglet Couches

Case à cocher par type de couche CNIG :

- **Tronçons OK** — trait vert, épaisseur proportionnelle à `largeur_utile`
- **Tronçons dégradés** — trait orange pointillé
- **Tronçons ko** — trait rouge plein
- **Obstacles** — marqueurs rouges (picto panneau)
- **Arrêts prioritaires** — marqueurs bleus, popup avec `nom_arret` et `niveau_accessibilite`
- **Stationnements PMR** — marqueurs bleu clair, popup avec `nb_places`

Chaque clic sur un tronçon ouvre une popup détaillée : `id_local`, `pente` (%), `devers` (%), `largeur_utile` (m), `revetement`, `etat_surface`, alertes actives.

### Onglet Travaux

Deux champs `date_debut` / `date_fin` filtre les chantiers actifs sur la période. L'appel `GET /api/travaux?debut=&fin=` appelle `fn_troncons_travaux` et renvoie un GeoJSON de polylignes ainsi qu'une liste des chantiers avec `libelle`, `impact_niveau` et les dates. Les chantiers sont dessinés en orange hachuré sur la carte ; ceux qui bloquent le cheminement sont indiqués en rouge.

### Onglet Isochrone

Sélection d'un arrêt prioritaire dans une liste déroulante. L'appel `GET /api/isochrone/{id_arret}?distance=1000` appelle `geostandard.fn_isochrone_pmr`, qui lance `pgr_drivingDistance` depuis le nœud le plus proche de l'arrêt et retourne un polygone `ST_ConcaveHull` des nœuds atteignables. L'isochrone est affiché en bleu translucide sur la carte. Trois indicateurs clés sont affichés : surface couverte (km²), nombre d'équipements PMR dans la zone, et longueur de réseau accessible (km).

## Interface Leaflet

### Layout

L'interface s'inspire de [Handimap](https://handimap.org) (Someware / Rennes) : pas de barre de navigation en haut, la carte occupe tout l'écran en arrière-plan, la sidebar de **320 px** se superpose à gauche. Sur mobile (< 768 px) la sidebar est masquée par défaut et un bouton flottant `☰ Menu` la fait glisser depuis la gauche.

### Sémiologie CNIG

Les tronçons sont representés conformément au standard CNIG :

| Classe | Couleur | Style | Épaisseur | | --- | --- | --- | | OK | Vert `#27AE60` | Trait continu | Proportionnelle à `largeur_utile` | | Dégradé | Orange `#E87A1A` | Pointillés | Proportionnelle à `largeur_utile` | | Ko | Rouge `#E8281A` | Trait continu | Fixe 3 px | | Route calculée | Bleu `#1A6EE8` | Trait épais | 6 px | | Isochrone | Bleu `#1A6EE8` | Remplissage 20 % | — |

### Tokens CSS

```css
--blue:   #1A6EE8;
--green:  #27AE60;
--orange: #E87A1A;
--red:    #E8281A;
--bg:     #F5F7FA;
--text:   #1A1A2E;
--border: #DDE2EC;
```

### Accessibilité RGAA

L'application intègre les exigences RGAA 4.1 :

- Tous les éléments interactifs ont un `aria-label` ou un texte visible
- La sidebar est un `<nav role='navigation' aria-label='Outil PMR'>`
- Les onglets utilisent `role='tab'`, `aria-selected`, `aria-controls`
- La navigation clavier est complète (Tab, Enter, échap)
- Le focus est visible en permanence (contour bleu 2 px)
- Les popups Leaflet ont `role='dialog'` et `aria-label`
- Les couleurs respectent les contrastes WCAG AA (4.5:1 minimum)

## Installation et déploiement

### Prérequis

- PostgreSQL 15, PostGIS 3.4, pgRouting 3.5
- Python 3.11+ avec FastAPI et psycopg2
- Un navigateur moderne (Chrome 120+, Firefox 121+, Safari 17+)

### 1. Base de données

```sql
-- Créer la base et activer les extensions
CREATE DATABASE losse_gelaisse;
\c losse_gelaisse
CREATE EXTENSION postgis;
CREATE EXTENSION pgrouting;

-- Créer les schémas
CREATE SCHEMA geostandard;
CREATE SCHEMA transport;
CREATE SCHEMA travaux;
CREATE SCHEMA qualite;

-- Jouer les scripts dans l'ordre
\i 01_geostandard_tables.sql
\i 02_transport_gtfs.sql
\i 03_travaux.sql
\i 04_qualite.sql
\i 05_mv_graphe_pmr.sql      -- vue matérialisée du graphe
\i 06_fonctions_routage.sql  -- fn_itineraire_pmr, fn_isochrone_pmr
\i 07_jeu_de_test.sql        -- données fictives Losse-en-Gelaisse
```

Refresh après mise à jour des données :

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY geostandard.mv_graphe_pmr;
```

### 2. Backend FastAPI

```bash
pip install fastapi uvicorn psycopg2-binary geoalchemy2

export DATABASE_URL="postgresql://user:pass@localhost/losse_gelaisse"
uvicorn app.main:app --reload --port 8000
```

Les quatre routes exposées :

| Route | Méthode | Fonction PL/pgSQL appelée |
| --- | --- | --- |
| `/api/itineraire` | POST | `fn_itineraire_pmr` |
| `/api/isochrone/{id}` | GET | `fn_isochrone_pmr` |
| `/api/travaux` | GET | `fn_troncons_travaux` |
| `/api/couches/{type}` | GET | requête directe sur `geostandard` |

### 3. Frontend

Ouvrir `webgis-pmr-leaflet.html` directement dans un navigateur, ou le servir avec n'importe quel serveur HTTP statique :

```bash
python -m http.server 9000
# puis ouvrir http://localhost:9000/webgis-pmr-leaflet.html
```

Pour pointer vers le backend local, modifier la constante en haut du script :

```javascript
const API_BASE = 'http://localhost:8000/api';
```

En mode démo (valeur par défaut `null`), les appels API sont remplacés par des GeoJSON mock embarqués dans le fichier.

## Limites et évolutions

### État actuel (démonstrateur)

Le frontend embarque des **données GeoJSON mock** à la place d'appels réels au backend. L'itinéraire, les isochrones et les travaux sont simulés avec des tracés statiques sur Losse-en-Gelaisse. Le backend FastAPI et les scripts SQL sont écrits et opérationnels, mais nécessitent une base PostGIS alimentée en données réelles pour être activés.

### Limites connues

- **Données CNIG** : le tronçon de cheminement doit être saisi ou importé manuellement ; aucun connecteur automatique vers OpenStreetMap n'est implémenté
- **Multimodalité** : le routage est uniquement piéton ; la correspondance bus + marche à pied n'est pas gérée
- **Temps réel** : les perturbations GTFS-RT (retards, suppressions) ne sont pas intégrées
- **Export** : aucune fonction d'export PDF ou GPX de l'itinéraire n'est présente
- **Authentification** : l'API est ouverte ; en production il faudrait un système de clés ou d'OAuth pour les agents

### Évolutions possibles

| Évolution | Effort estimé | Dépendance |
| --- | --- | --- |
| Import GTFS automatique | 2 j | Flux GTFS AOM réel |
| Export itinéraire PDF | 1 j | WeasyPrint ou Puppeteer |
| Signalement terrain (mobile) | 3 j | schéma `qualite` déjà prêt |
| Couche temps réel travaux | 2 j | API Waze ou flux mairie |
| Multimodalité bus + marche | 5 j | pgRouting + GTFS trips |
| Mode sombre | 0.5 j | CSS tokens déjà prêts |
