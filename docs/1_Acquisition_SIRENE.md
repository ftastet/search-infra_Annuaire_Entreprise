# 1 – Acquisition SIRENE (stock + flux)

## 1. Objectif du module SIRENE

L’objectif est de récupérer les données SIRENE (stock complet + flux quotidiens) et de les déposer dans MinIO (`insee/stock/` et `insee/flux/`) pour alimenter l’ETL, l’indexation Elasticsearch et les exports Data.gouv.  
Ce module ne réalise aucun enrichissement : il assure uniquement l’acquisition fiable des données sources fournies par l’INSEE.

---

## 2. DAG `data_processing_sirene_stock`

**Chemin** : `workflows/data_pipelines/sirene/stock/`  
**dag_id** : `data_processing_sirene_stock`  
**Schedule** : `0 0 * * *` (tous les jours à 00h00)  
**Source** : ZIP SIRENE (stock complet)  
**Sortie** : MinIO `insee/stock/`

### Vue d’ensemble
1. Préparation du dossier temporaire  
2. Téléchargement du stock SIRENE  
3. Upload MinIO  
4. Nettoyage

> Les notifications ne sont pas gérées par une tâche dédiée : elles passent via les callbacks Airflow.

---

## 3. DAG `data_processing_sirene_flux`

**Chemin** : `workflows/data_pipelines/sirene/flux/`  
**dag_id** : `data_processing_sirene_flux`  
**Schedule** : `0 4 * * *` (tous les jours à 04h00)  
**Source** : API INSEE (flux unités légales / établissements modifiés)  
**Sorties** : MinIO `insee/flux/`, métadonnées de dernière modification

### Vue d’ensemble
1. Préparation / nettoyage du dossier temporaire  
2. Appel API flux (unités légales puis établissements)  
3. Écriture des fichiers du jour  
4. Mise à jour des métadonnées (ex : dernière date de mise à jour)  
5. Upload vers MinIO  
6. Nettoyage  

> Cas particulier : le 1er jour du mois, les tâches flux génèrent des CSV vides avec en-têtes (unités légales et établissements), les compressent, puis les envoient vers MinIO en l’absence de données renvoyées par l’API.  
> Les notifications sont gérées via les callbacks Airflow, pas via une tâche spécifique.

---

## 4. Chronologie et dépendances

1. `data_processing_sirene_stock` → stock complet en MinIO  
2. `data_processing_sirene_flux` → flux du jour en MinIO  
3. L’ETL `extract_transform_load_db` lit :
   - `insee/stock/`
   - `insee/flux/`

Le couple **stock + flux** constitue le socle de la reconstruction SIRENE dans l’ETL avant intégration des données RNE.

---

## 5. Points à creuser plus tard

- Gestion avancée des dates flux  
- Structure exacte des dossiers MinIO  
- Gestion des erreurs (API INSEE, fichiers incomplets, flux du premier jour)  
- Alignement des horaires avec RNE et l’ETL  
- Format précis des métadonnées générées  
