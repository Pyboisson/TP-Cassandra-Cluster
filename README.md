TP Cassandra Cluster
====================

Ce dépôt contient une configuration Docker Compose pour déployer un cluster Cassandra de 3 nœuds destiné aux TPs.

Structure du dépôt
------------------

- docker-compose.yml — cluster Cassandra (3 nœuds: cass1, cass2, cass3) sur un réseau bridge local
- README.md — ce fichier
- docs/compte-rendu.md — document à compléter avec vos observations et résultats

Prérequis
---------

- Docker Desktop et Docker Compose
- Ports libres sur la machine hôte: 9042 (cass1), 9043 (cass2), 9044 (cass3)

Démarrage du cluster
--------------------

Dans le dossier du dépôt:

```powershell
docker compose up -d cass1
# attendre ~60-90s que cass1 finisse de démarrer
docker compose up -d cass2
docker compose up -d cass3
```

Vérifications de base
---------------------

1) État du cluster

```powershell
docker exec cass1 nodetool status
```

Attendu:

- Datacenter: dc1
- 3 nœuds en UN (Up/Normal)
- Racks: rack1, rack2, rack3

2) Connexion CQL

```powershell
docker exec -it cass1 cqlsh
```

3) Nom du cluster et snitch

```powershell
docker exec cass1 nodetool describecluster
```

Création du keyspace (RF=3)
---------------------------

Dans cqlsh:

```sql
CREATE KEYSPACE IF NOT EXISTS overwatch
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 3
};
```

Vérifier:

```powershell
docker exec cass1 cqlsh -e "DESCRIBE KEYSPACE overwatch;"
```

Chargement des données (exemple)
--------------------------------

Si vous avez un script d’ingestion (ex: get_overwatch.py) côté hôte:

```powershell
$env:CASSANDRA_HOSTS="127.0.0.1"
$env:CASSANDRA_KEYSPACE="overwatch"
python .\script\get_overwatch.py
```

Niveaux de cohérence (lecture)
------------------------------

Dans cqlsh:

```sql
USE overwatch;

CONSISTENCY ONE;
SELECT count(*) FROM heroes;

CONSISTENCY QUORUM;
SELECT count(*) FROM heroes;

CONSISTENCY ALL;
SELECT count(*) FROM heroes;
```

Simulation de panne
-------------------

- Arrêter cass3: `docker stop cass3`
- Vérifier: `docker exec cass1 nodetool status` (cass3 doit passer en DN)
- Tester lectures ONE/QUORUM/ALL (ALL doit échouer si RF=3 et un nœud down)
- Redémarrer: `docker start cass3`
- Forcer la synchronisation si besoin: `docker exec cass1 nodetool repair overwatch`

Arrêt et nettoyage
------------------

```powershell
docker compose down
# Pour supprimer les données locales des nœuds
Remove-Item -Recurse -Force .\data\cass1, .\data\cass2, .\data\cass3
```

Notes
-----

- Le compose utilise GossipingPropertyFileSnitch et définit dc1 / rack1-3.
- Les volumes `./data/cass*` conservent l’état (dont le DC/rack initial). Si vous changez la topologie, recréez les volumes.
