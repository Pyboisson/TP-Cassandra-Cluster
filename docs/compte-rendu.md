---
title: Compte-rendu TP Cassandra Cluster
author: Pierre-Yves BOISSON
date: 07/10/2026
---

# Contexte

Le but du TP est de créer un cluster Cassandra de 3 nœuds, fonctionnant selon une architecture distribuée sans nœud maître, et containerisé avec Docker.


L'API choisie pour ce projet est OverFast API (https://overfast-api.tekrop.fr/#tag/Heroes). Cette API reprend les données des personnages d'Overwatch, un jeu de tir en ligne  développé et publié par Blizzard Entertainment. C'est une API créée par une personne tierce en se basant sur les données fournies par Blizzard.
Puisque certaines données accessibles via l'API sont de grandes données texte, ou des liens url, seules certaines colonnes ont été choisies pour les besoins du TP :
- `hero_key`
- `name`
- `role`
- `subrole`
- `location`
- `age`
- `health`
- `shields`
- `armor`
- `total_hp`



# Création du KEYSPACE
```sql
CREATE KEYSPACE overwatch
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 3
};
```

Le keyspace a été créé dès le départ avec un facteur de réplication de 3 dans le datacenter dc1.

# Ingestion des données
Les données sont récupérées et injectées à l'aide d'un script python get_overwatch.py
Des requêtes métiers permettent d'afficher les données d'intérêt (par exemple, tous les héros ayant le rôle "tank")

USE overwatch;


DESCRIBE TABLES;

![alt text](image-2.png)


SELECT hero_key, name, role, total_hp
FROM heroes
LIMIT 10;

Voici un exemple des données reccueillies

![alt text](image.png)

La table contient bien 53 lignes, correspondant aux héros importés.

![alt text](image-3.png)

# Démarrage du cluster

Lancement de chaque node un par un

docker compose up -d cass1

docker compose up -d cass2

docker compose up -d cass3



# Topologie et vérifications

- Sortie `nodetool status`

docker exec cass1 nodetool status

![alt text](image-12.png)

Les 3 nodes sont en UN

- Sortie `nodetool describecluster`

docker exec cass1 nodetool describecluster

![alt text](image-4.png)

Description du cluster

- Sortie `nodetool ring`

docker exec cass1 nodetool ring

![alt text](image-5.png)

Chaque nœud possède 16 tokens. Avec 3 nœuds, le cluster contient donc 48 tokens au total, utilisés pour répartir les partitions dans l’anneau Cassandra.


- Sortie `nodetool getendpoints`

docker exec cass1 nodetool getendpoints overwatch heroes winston 

![alt text](image-11.png)

La commande retourne les 3 endpoints responsables de la partition `winston`. Avec un facteur de réplication de 3 et 3 nœuds dans `dc1`, cela signifie que cette partition est répliquée sur les 3 nœuds du cluster.


# Schéma et réplication

- `CREATE KEYSPACE overwatch` avec NetworkTopologyStrategy, dc1:3
- `DESCRIBE KEYSPACE overwatch`

![alt text](image-6.png)

# Données métier

- `SELECT count(*) FROM overwatch.heroes;`

![alt text](image-8.png)

- Exemples de requête

Pour permettre une requête efficace sur l’âge sans utiliser ALLOW FILTERING, une table orientée requête a été créée : `heroes_by_age_bucket`.


CREATE TABLE IF NOT EXISTS overwatch.heroes_by_age_bucket (
  age_bucket text,
  age int,
  hero_key text,
  name text,
  role text,
  total_hp int,
  armor int,
  location text,
  PRIMARY KEY ((age_bucket), age, hero_key)
) WITH CLUSTERING ORDER BY (age ASC, hero_key ASC);

-- Requête:
SELECT hero_key, name, role, age
FROM overwatch.heroes_by_age_bucket
WHERE age_bucket = 'all' AND age < 18;

![alt text](image-7.png)


# Niveaux de cohérence

- `avec les 3 nodes UN`

CONSISTENCY ONE;

![alt text](image-9.png)

SELECT hero_key, name, role, total_hp
FROM heroes
WHERE hero_key = 'winston'; 

![alt text](image-10.png)

La lecture fonctionne car le niveau ONE nécessite la réponse d’un seul réplica disponible.

CONSISTENCY QUORUM;

![alt text](image-17.png)

![alt text](image-18.png)

La lecture fonctionne car, avec RF=3, le niveau de cohérence QUORUM nécessite la réponse d’au moins 2 réplicas sur 3.

CONSISTENCY ALL;

![alt text](image-19.png)

![alt text](image-20.png)

La lecture réussit car le quorum est atteint : avec un facteur de réplication de 3, Cassandra doit obtenir la réponse d’au moins 2 réplicas sur 3.


- `En résumé`

Nous avons testé les niveaux de cohérence ONE, QUORUM et ALL sur la table métier overwatch.heroes avec la clé hero_key = 'winston'.
Avec RF = 3 :
- ONE réussit dès qu’un seul réplica répond ;
- QUORUM réussit lorsque deux réplicas sur trois répondent ;
- ALL réussit uniquement lorsque les trois réplicas répondent.
Lorsque les trois nœuds Cassandra sont actifs, les trois lectures réussissent. 


# Simulation de panne

- `Arrêter cass3`

docker stop cass3

- `Vérification arrêt cass3 `

docker exec cass1 nodetool status

cass3 apparaît en DN

![alt text](image-21.png)

- `Vérification arrêt cass3 2`

docker ps -a --filter "name=cass3"

![alt text](image-13.png)

- `Test de cohérence avec cass3 stoppé`

![alt text](image-14.png)

- `relancer cass3`

docker start cass3

- `Vérification relance et récupération des données 1`

docker exec cass1 nodetool getendpoints overwatch heroes winston

![alt text](image-15.png)

- `Vérification relance et récupération des données 2`

docker exec cass3 cqlsh -e "SELECT count(*) FROM overwatch.heroes;"

![alt text](image-16.png)

On obtient bien les 53 lignes de données

# Observations et conclusion

- Répartition des données (tokens, endpoints)

Les données du keyspace overwatch sont répliquées sur les trois nœuds grâce au facteur de réplication RF=3.


- Impact de la panne sur les différents niveaux de cohérence

La panne n'a eu aucun impact sur l'intégrité des données. Rien n'a été perdu dans les autres nodes, tout a été récupéré dans le node cassé. Cependant, les différents niveaux de cohérences n'étaient pas tous respecté. ONE et QUORUM n'ont pas bloqué les requêtes car les données étaient bien présente sur au moins un, ou au moins deux nodes sur trois. Par contre le niveau de cohérence ALL n'était pas satisfaits du fait de la panne de cass3 et a bloqué la requête

- Bonnes pratiques retenues

Tu peux mettre quelque chose comme ça dans **“Bonnes pratiques retenues”** :


# Bonnes pratiques retenues

- Définir correctement la topologie Cassandra dès le démarrage du cluster : nom du cluster, datacenter, racks et snitch (`GossipingPropertyFileSnitch`).

- Vérifier l’état du cluster avant toute opération importante avec `nodetool status`, afin de s’assurer que les nœuds sont bien en état `UN` (Up/Normal).

- Créer le keyspace avec une stratégie de réplication adaptée au cluster. Dans ce TP, `NetworkTopologyStrategy` avec `dc1: 3` permet de répliquer chaque partition sur trois réplicas.

- Contrôler la réplication avec `DESCRIBE KEYSPACE` et `nodetool getendpoints` pour vérifier quels nœuds sont responsables d’une partition donnée.

- Utiliser les niveaux de cohérence selon le besoin :
  - `ONE` privilégie la disponibilité ;
  - `QUORUM` offre un compromis entre disponibilité et cohérence ;
  - `ALL` garantit une cohérence forte mais nécessite que tous les réplicas soient disponibles.

- Tester les comportements en cas de panne afin de comprendre l’impact des niveaux de cohérence sur les lectures et écritures.

- Après le redémarrage d’un nœud, vérifier son retour dans le cluster avec `nodetool status` et contrôler que les données sont accessibles.


- Éviter les requêtes avec `ALLOW FILTERING` en Cassandra. Il est préférable de créer des tables orientées requêtes, comme les tables `heroes_by_*`.

- Documenter les commandes utilisées et les résultats observés pour faciliter la reproduction du cluster et l’analyse des incidents.
