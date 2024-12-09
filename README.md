# Démonstration d’Indexation Géospatiale

## Prérequis
- PostgreSQL installé
- Extension PostGIS activée (PostgreSQL)
- Un jeu de données spatiales (Shapefile, GeoJSON, etc.)
- Outils CLI : `psql`, `shp2pgsql`
- Compte GitHub et Git installé localement

## 1. Préparation de la Base de Données
On commence par créer une nouvelle base de données 
```bash
sudo -u postgres createdb demo_spatial
```
On peut ensuite s'y connecter avec `psql` et activer `PostGIS`

```bash
sudo -u postgres psql -d demo_spatial
CREATE EXTENSION postgis;
```
On peut directement passer nos commande `SQL` dans `psql`, ou on peut aussi se servir de commandes directment dans le terminal en spécifiant l'usager, la base de données et la commande avec les flags `-u`,  `-d` et `-c`, respectivement. Pour cette démonstration, nous varierons nos commandes. On peut quitter `psql` avec la commande `\q`

## 2. Chargement des Données

Il faut d'abord s'assurer d'avoir les permissions nécessaires pour importer les donnés dans la base.

```bash
sudo -u postgres psql -d demo_spatial -c "ALTER SCHEMA public OWNER TO postgres;"
sudo -u postgres psql -d demo_spatial -c "GRANT ALL PRIVILEGES ON SCHEMA public TO postgres;"
```

Ensuite, on peut importer un Shapefile dans la base avec `shp2pgsql`. On doit d'abord extraire celui d'un fichier compressé, par exemple un boundary file présent à cette addresse: https://www12.statcan.gc.ca/census-recensement/2021/geo/sip-pis/boundary-limites/index2021-eng.cfm?Year=21

```bashq
sudo -u postgres shp2pgsql -I -s 3857 /tmp/Canada/lpr_000b21a_f.shp public.canada > /tmp/Canada/output.sql
sudo -u postgres psql -d demo_spatial -f /tmp/Canada/output.sql
```

Note :
-I crée déjà un index spatial par défaut. On le supprimera pour la démonstration.
-s 3857 indique le système de coordonnées.

Vérifier le chargement :

```bash
SELECT COUNT(*) FROM canada;
SELECT ST_Extent(geom) FROM canada;
```

Les données ont été chargées avec succès dans la table canada, qui contient 13 lignes et une boîte englobante (`ST_Extent`) décrivant l'étendue spatiale des géométries.

## 3. Conversion de à une autre géométrie et Création de l’Index pour la Démonstration

les coordonnées sont, en ce moment, au format web mercator (SRID 3857). On peut convertir les coordonnées de notre table au format longitude/lattitude qu'on connaît mieux:

```bash
ALTER TABLE canada
  ALTER COLUMN geom TYPE geometry(MultiPolygon, 4326)
  USING ST_Transform(geom, 4326);
```

On peut tester un première commande pour vérifier l'aire (en mêtres cubes) de que couvre la table:

```bash
SELECT SUM(ST_Area(geom::geography)) AS total_area FROM canada;
```

## 4. Performance

On peut supprimer l’index créé automatiquement pour vérifier l'impact de l'index sur la performance
```bash
DROP INDEX IF EXISTS canada_geom_idx;
```

Tester une requête spatiale sans index :

```bash
EXPLAIN ANALYZE
SELECT * FROM canada
WHERE ST_Intersects(
  geom,
  ST_MakeEnvelope(-73.6, 45.5, -73.5, 45.6, 4326)
);
```

Ce résultat de `EXPLAIN ANALYZE` dans `PostgreSQL` fournit des détails sur une requête qui vérifie les géométries de la table `canada` intersectant un polygone spécifique à l'aide de la fonction `ST_Intersects`. Noter le temps d’exécution et le plan (« Seq Scan »).

Ensuite, on peut créer l’index GiST :

```bash
CREATE INDEX idx_canada_geom
ON canada 
USING GIST (geom);
```

Retester la même requête :

```bash
EXPLAIN ANALYZE
SELECT * FROM canada
WHERE ST_Intersects(
  geom,
  ST_MakeEnvelope(-73.6, 45.5, -73.5, 45.6, 4326)
);
```
On peut constater l’utilisation de l’index et la réduction du temps. Montrer comment l’index spatial permet une recherche hiérarchique plutôt qu’un parcours exhaustif.


## 5. Manipulation des données


On va commencer par créer un table nommée 'cities':

```bash
CREATE TABLE cities (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    geom geometry(Point, 4326)
);
```

On peut y ajouter une ville : 
```bash
INSERT INTO cities (name, geom)
VALUES ('Montreal', ST_SetSRID(ST_MakePoint(68.5687, 11.0735), 4326));
```

Avec la commande suivante, on peut vérifier si une ville est dans un rayon de 50 km de d'un point:

```bash
SELECT name, ST_Distance(geom::geography, ST_SetSRID(ST_MakePoint(-73.5673, 45.5017), 4326)::geography) AS distance_km
FROM cities
WHERE ST_DWithin(geom::geography, ST_SetSRID(ST_MakePoint(-73.5673, 45.5017), 4326)::geography, 50000)
ORDER BY distance_km ASC;
```

Finalement, avec la commande suivante, on peut vérifier si la ville en question est présente dans la bonne province (Quebecm gid = 5)

```bash
SELECT c.name AS city_name, r.gid AS region_id
FROM cities c
JOIN canada r
ON ST_Intersects(c.geom, r.geom);
```


## 6. Visualisation dans un Outil SIG (QGIS)

Visualiable avec QGIS
