# Guideline_ajout_mail_tel.md

## 1. Objectif
Décrire comment adapter le pipeline RNE pour intégrer des informations de contact (email, téléphone) lorsqu’elles sont détectées dans les JSON sources.

L’objectif est de préciser les étapes nécessaires pour :
- étendre les modèles JSON (Pydantic),
- étendre les fonctions de mapping RNE → SQLite,
- étendre le schéma SQLite,
- adapter les parties concernées du pipeline,
- mettre à jour la documentation existante.

---

## 2. Pré-requis

Avant toute modification, il faut :
1. Identifier précisément les champs JSON contenant des données contact.
2. Documenter pour chaque champ :
   - chemin complet dans le JSON,
   - type de contact (email, téléphone),
   - entité concernée (unité légale, siège, dirigeant…).
3. Déterminer dans quelles tables SQLite ajouter ces informations : Ce sera dirigeant_pp ou dirigeant_pm en fonction du champ source. 
     
---

## 3. Étape 1 – Étendre les modèles JSON

Les champs email ou téléphone doivent être ajoutés dans les modèles décrivant les JSON RNE.

Fichiers concernés (à adapter selon le repo) :
- rne_model.py
- modèles Pydantic utilisés dans le parsing RNE

Principes :
- ajouter un attribut email si disponible,
- ajouter un attribut telephone si disponible,
- ajouter les sous-modèles nécessaires si les contacts sont dans un bloc « contact ».

Les champs doivent être optionnels afin de ne pas casser le parsing en cas d’absence.

---

## 4. Étape 2 – Adapter le mapping RNE → SQLite

Fichiers impliqués :
- process_rne.py
- fonctions de mapping par table
  - map_rne_company_to_ul,
  - mapping du siège,
  - mapping des dirigeants PP et PM,
  - ou toute fonction similaire.

Pour chaque champ contact identifié :
- ajouter email et telephone dans les dictionnaires renvoyés,
- définir une logique en cas de plusieurs valeurs (première valeur, concaténation, sérialisation).

Entités possibles :
- pour l’unité légale : email et telephone,
- pour le siège : email_siege et telephone_siege,
- pour les dirigeants : email_dirigeant et telephone_dirigeant.

---

## 5. Étape 3 – Modifier le schéma SQLite

Il faut ajouter les colonnes correspondantes dans les tables RNE.

Selon la structure existante, il faudra modifier :
- les instructions CREATE TABLE si la base est reconstruite à chaque run,
- ou utiliser ALTER TABLE si le schéma doit évoluer progressivement.

Colonnes possibles :
- unite_legale.email,
- unite_legale.telephone,
- siege.email,
- siege.telephone,
- dirigeant_pp.email_dirigeant,
- dirigeant_pp.telephone_dirigeant,
- dirigeant_pm.email_dirigeant,
- dirigeant_pm.telephone_dirigeant.

Les types recommandés : TEXT pour toutes les colonnes contact.

---

## 6. Étape 4 – Adapter le DAG si la source n’est pas exclusivement RNE

Si les données contact proviennent :
- d’un autre flux INPI,
- de SIRENE,
- d’un CRM interne,
- ou d’une API externe,

alors il faut identifier :
- le DAG ou script d’acquisition correspondant,
- le moment où les données doivent être fusionnées avec les données RNE.

Dans un tel cas, l’enrichissement peut être fait :
- directement dans la base SQLite RNE,
- ou dans un datawarehouse / datalake en downstream.

---

## 7. Étape 5 – Tests et validation

1. Préparer un JSON contenant des champs contact.
2. Lancer le pipeline ou les fonctions de parsing/mapping.
3. Vérifier :
   - que le JSON est parsé sans erreur,
   - que les valeurs contact sont récupérées,
   - qu’elles sont insérées dans la base SQLite,
   - que les colonnes existent bien dans les tables,
   - que la logique est correcte en cas d’absence ou pluralité de valeurs.

4. Tester différents scénarios :
   - aucun champ contact,
   - un email mais pas de téléphone,
   - plusieurs téléphones,
   - valeurs vides ou mal formées.

---

## 8. Étape 6 – Mise à jour de la documentation

Mettre à jour :
- 2.4 (Mapping Source → Cible),
- 2.5 (Modèle de données final),
- 2.6 (Consommation de la base RNE).

Ajouter :
- les colonnes contact,
- les règles de priorité,
- les cas d’usage élargis (enrichissements CRM, services internes, notifications, etc.).

---

## 9. Checklist synthèse

- [ ] Les champs JSON contact ont été identifiés.
- [ ] Les modèles Pydantic ont été étendus.
- [ ] Les fonctions de mapping RNE → SQLite ont été mises à jour.
- [ ] Le schéma SQLite contient les nouvelles colonnes.
- [ ] Le pipeline accepte et insère les contacts sans erreurs.
- [ ] La documentation a été mise à jour.
- [ ] Les tests de bout en bout ont été validés.

---

Fin du document.
