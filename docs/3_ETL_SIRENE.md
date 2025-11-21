# 3 – ETL SIRENE (construction de la base sirene.db)

## 1. Objectif du module ETL SIRENE

L’ETL reconstruit chaque jour une base SQLite `sirene.db` en combinant :
- les données SIRENE (stock + flux),
- la base consolidée RNE,
- les jeux de données secondaires (liste non exhaustive),
- des contrôles de volumétrie et validations internes.

Le résultat final est déposé dans MinIO (`sirene/database/`) et alimente :
- l’index Elasticsearch,
- les exports Data.gouv,
- les usages internes.

---

## 2. DAG `extract_transform_load_db` (ETL principal)

**Chemin** : `workflows/data_pipelines/etl/`  
**dag_id** : valeur de la variable Airflow `AIRFLOW_ETL_DAG_NAME`  
**Schedule** : `0 5 * * *` (tous les jours à 05h00)

### Sources utilisées
- SIRENE stock (`insee/stock/`)
- SIRENE flux (`insee/flux/`)
- Base RNE consolidée (`rne/database/`)
- Jeux de données secondaires (via `DatabaseTableConstructor`)

### Sorties
- Base SQLite `sirene.db.gz` déposée dans MinIO (`sirene/database/`)
- Fichier `data_source_last_modified.json`
- Déclenchement du DAG d’indexation Elasticsearch

---

## 3. Vue d’ensemble du traitement ETL

1. Préparation du répertoire temporaire  
2. Chargement SIRENE stock  
3. Application des flux SIRENE  
4. Intégration de la base RNE  
5. Contrôles de cohérence / volumétrie  
6. Enrichissements via les jeux de données secondaires  
7. Génération de `sirene.db`  
8. Compression et upload vers MinIO (`sirene/database/`)  
9. Génération de `data_source_last_modified.json`  
10. Suppression locale du fichier SQLite  
11. Déclenchement du DAG d’indexation Elasticsearch

> Les notifications, logs et callbacks Airflow sont gérés directement par les décorateurs/tasks et non documentés ici.

---

## 4. Rôle des enrichissements secondaires

Les datasets secondaires apportent des attributs complémentaires dans des tables annexes.
Ils ne modifient pas la mécanique centrale (SIRENE + RNE) mais enrichissent la base finale.

L’intégration est orchestrée par `DatabaseTableConstructor` dans un bloc dédié de l’ETL.

---

## 5. Chronologie et dépendances

1. Acquisition SIRENE (stock + flux)  
2. Acquisition RNE (stock + flux + base consolidée)  
3. **ETL** :
   - lit SIRENE et RNE  
   - construit la base SQLite `sirene.db`  
   - génère les métadonnées  
   - publie l’archive dans MinIO  
4. Déclenchement du DAG Elasticsearch  
5. La base publiée est ensuite utilisée par Data.gouv et d’autres consommateurs

---

## 6. Points à creuser plus tard

- Structure exacte des tables créées  
- Détails des validations (volumétrie, cohérence UL/établissements)  
- Mécanique interne de `DatabaseTableConstructor`  
- Format exact de `data_source_last_modified.json`  
- Gestion des erreurs / retries Airflow
