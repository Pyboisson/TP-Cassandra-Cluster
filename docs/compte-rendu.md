---
title: Compte-rendu TP Cassandra Cluster
author: 
date: 
---

# Contexte

Décrivez brièvement l’objectif du TP et l’environnement (Docker, 3 nœuds, dc1, RF=3…).

# Démarrage du cluster

- Commandes exécutées
- Problèmes rencontrés / solutions

# Topologie et vérifications

- Sortie `nodetool status`
- Sortie `nodetool describecluster`
- Sortie `nodetool ring`

# Schéma et réplication

- `CREATE KEYSPACE overwatch` avec NetworkTopologyStrategy, dc1:3
- `DESCRIBE KEYSPACE overwatch`

# Données métier

- Re-création des tables et ingestion (si applicable)
- `SELECT count(*) FROM overwatch.heroes;`
- Exemples de requêtes

# Niveaux de cohérence

- Tests `CONSISTENCY ONE`, `QUORUM`, `ALL` sur une clé réelle
- Résultats observés

# Simulation de panne

- `docker stop cass3`
- `docker exec cass1 nodetool status` (DN attendu)
- Lectures ONE/QUORUM/ALL (ALL doit échouer)
- `docker start cass3` puis éventuel `nodetool repair overwatch`

# Observations et conclusion

- Répartition des données (tokens, endpoints)
- Impact de la panne sur les différents niveaux de cohérence
- Bonnes pratiques retenues
