Reconstruction complète du projet (from scratch)
================================================

Objectif
--------

Recréer l’intégralité de la stack technique du projet dans un dossier neuf :

- Cluster Cassandra 3 nœuds sous Docker (dc1, rack1-3, GossipingPropertyFileSnitch)
- Keyspace overwatch (NetworkTopologyStrategy, dc1:3)
- Tables métier et tables orientées requêtes
- Ingestion des données Overwatch via script Python
- Vérifications et tests de cohérence

Pré-requis
----------

- Docker Desktop + Docker Compose
- Python 3.10+ (ou équivalent) et pip
- Accès Internet pour l’API OverFast publique

1) Créer un nouveau dossier de travail
--------------------------------------

```powershell
New-Item -ItemType Directory -Path "C:\\path\\to\\New-TP" | Out-Null
cd "C:\\path\\to\\New-TP"
```

2) Initialiser le dépôt Git (optionnel)
---------------------------------------

```powershell
git init
git remote add origin https://github.com/Pyboisson/TP-Cassandra-Cluster
```

3) Créer le docker-compose.yml
------------------------------

Créez le fichier `docker-compose.yml` avec ce contenu :

```yaml
version: "3.8"

services:
  cass1:
    image: cassandra:4.1
    container_name: cass1
    hostname: cass1
    environment:
      CASSANDRA_CLUSTER_NAME: tp2-cluster
      CASSANDRA_DC: dc1
      CASSANDRA_RACK: rack1
      CASSANDRA_ENDPOINT_SNITCH: GossipingPropertyFileSnitch
      CASSANDRA_SEEDS: cass1,cass2
      CASSANDRA_BROADCAST_RPC_ADDRESS: cass1
      MAX_HEAP_SIZE: 512M
      HEAP_NEWSIZE: 100M
    ports:
      - "9042:9042"
    volumes:
      - ./data/cass1:/var/lib/cassandra
    networks:
      - cassandra-net
    restart: unless-stopped

  cass2:
    image: cassandra:4.1
    container_name: cass2
    hostname: cass2
    depends_on:
      - cass1
    environment:
      CASSANDRA_CLUSTER_NAME: tp2-cluster
      CASSANDRA_DC: dc1
      CASSANDRA_RACK: rack2
      CASSANDRA_ENDPOINT_SNITCH: GossipingPropertyFileSnitch
      CASSANDRA_SEEDS: cass1,cass2
      CASSANDRA_BROADCAST_RPC_ADDRESS: cass2
      MAX_HEAP_SIZE: 512M
      HEAP_NEWSIZE: 100M
    ports:
      - "9043:9042"
    volumes:
      - ./data/cass2:/var/lib/cassandra
    networks:
      - cassandra-net
    restart: unless-stopped

  cass3:
    image: cassandra:4.1
    container_name: cass3
    hostname: cass3
    depends_on:
      - cass1
    environment:
      CASSANDRA_CLUSTER_NAME: tp2-cluster
      CASSANDRA_DC: dc1
      CASSANDRA_RACK: rack3
      CASSANDRA_ENDPOINT_SNITCH: GossipingPropertyFileSnitch
      CASSANDRA_SEEDS: cass1,cass2
      CASSANDRA_BROADCAST_RPC_ADDRESS: cass3
      MAX_HEAP_SIZE: 512M
      HEAP_NEWSIZE: 100M
    ports:
      - "9044:9042"
    volumes:
      - ./data/cass3:/var/lib/cassandra
    networks:
      - cassandra-net
    restart: unless-stopped

networks:
  cassandra-net:
    name: cassandra-net
    driver: bridge
```

4) Démarrer le cluster Cassandra
--------------------------------

```powershell
docker compose up -d cass1
# Attendre ~60-90s que cass1 termine son bootstrap
docker compose up -d cass2
docker compose up -d cass3
```

Vérifier l’état :

```powershell
docker exec cass1 nodetool status
```

Attendu : Datacenter: dc1, 3 nœuds en UN, racks rack1/2/3.

5) Créer le keyspace overwatch (RF=3)
-------------------------------------

```powershell
docker exec cass1 cqlsh -e "CREATE KEYSPACE IF NOT EXISTS overwatch WITH replication = {'class': 'NetworkTopologyStrategy', 'dc1': 3};"
```

Vérifier :

```powershell
docker exec cass1 cqlsh -e "DESCRIBE KEYSPACE overwatch;"
```

6) Préparer l’environnement Python
----------------------------------

Créer un virtualenv (optionnel) et installer les dépendances :

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1

New-Item -ItemType Directory -Path .\script | Out-Null
Set-Content -Path .\requirements.txt -Value @"
cassandra-driver==3.30.1
requests==2.34.2
"@

pip install -r requirements.txt
```

7) Créer le script d’ingestion get_overwatch.py
-----------------------------------------------

Créez le fichier `.\script\get_overwatch.py` en vous basant sur la version du TP2. Les points clés :

- Variables d’env : `CASSANDRA_HOSTS=127.0.0.1`, `CASSANDRA_KEYSPACE=overwatch`.
- Le script crée les tables si nécessaire (idempotent) et importe les héros via OverFast.

Exécutez ensuite :

```powershell
$env:CASSANDRA_HOSTS="127.0.0.1"
$env:CASSANDRA_KEYSPACE="overwatch"
$env:OVERFAST_FALLBACK_PUBLIC="1"
python .\script\get_overwatch.py
```

8) Vérifications post-ingestion
-------------------------------

```powershell
docker exec cass1 cqlsh -e "SELECT table_name FROM system_schema.tables WHERE keyspace_name = 'overwatch';"
docker exec cass1 cqlsh -e "SELECT count(*) FROM overwatch.heroes;"
docker exec cass1 cqlsh -e "SELECT hero_key, name, role, total_hp FROM overwatch.heroes LIMIT 10;"
```

9) Tests de distribution et de cohérence (optionnel mais recommandé)
--------------------------------------------------------------------

```powershell
docker exec cass1 nodetool describecluster
docker exec cass1 nodetool ring
docker exec cass1 nodetool getendpoints overwatch heroes winston

docker exec cass1 cqlsh -e "CONSISTENCY ONE; SELECT count(*) FROM overwatch.heroes;"
docker exec cass1 cqlsh -e "CONSISTENCY QUORUM; SELECT count(*) FROM overwatch.heroes;"
docker exec cass1 cqlsh -e "CONSISTENCY ALL; SELECT count(*) FROM overwatch.heroes;"
```

10) Nettoyage / Rebuild (si besoin de repartir propre)
------------------------------------------------------

```powershell
docker compose down
Remove-Item -Recurse -Force .\data\cass1, .\data\cass2, .\data\cass3
docker compose up -d cass1
docker compose up -d cass2
docker compose up -d cass3
```

Notes
-----

- Toujours vérifier le snitch effectif :
  ```powershell
  docker exec cass1 sh -c "grep -E '^(endpoint_snitch|cluster_name):|^dc=|^rack=' /etc/cassandra/cassandra.yaml /etc/cassandra/cassandra-rackdc.properties"
  ```
  Attendu : `endpoint_snitch: GossipingPropertyFileSnitch`, `dc=dc1` et `rack=rack1` sur cass1.

- Après modification de la réplication, un `nodetool repair overwatch` peut être utile :
  ```powershell
  docker exec cass1 nodetool repair overwatch
  ```

- Éviter `ALLOW FILTERING` : préférez créer des tables orientées requêtes (ex: `heroes_by_*`).
