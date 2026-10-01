# Plan de tests — Microservices Client, Inventory et Billing

**Version :** 1.0  
**Date :** 1 octobre 2026  
**Statut :** spécification de tests proposée, à adapter aux exigences réelles  
**Périmètre :** 3 microservices, 21 scénarios et 65 cas de test  
**Résultats d’exécution :** aucun test exécuté dans le cadre de ce document

## Sommaire

1. [Objectif et périmètre](#1-objectif-et-périmètre)
2. [Hypothèses et décisions métier](#2-hypothèses-et-décisions-métier)
3. [Contrats API à adapter](#3-contrats-api-à-adapter)
4. [Exigences et critères d’acceptation](#4-exigences-et-critères-dacceptation)
5. [Environnement et jeux de données](#5-environnement-et-jeux-de-données)
6. [Catalogue des scénarios](#6-catalogue-des-scénarios)
7. [Cas de test détaillés](#7-cas-de-test-détaillés)
8. [Matrice de traçabilité](#8-matrice-de-traçabilité)
9. [Campagnes et exécution](#9-campagnes-et-exécution)
10. [Automatisation](#10-automatisation)
11. [Suivi des anomalies et bilan](#11-suivi-des-anomalies-et-bilan)
12. [Checklist d’adaptation](#12-checklist-dadaptation)

## 1. Objectif et périmètre

Vérifier que les clients et produits sont gérés correctement, que les factures contiennent les bonnes références et les bons montants, et que les interactions entre les services restent cohérentes lors d’une erreur ou d’une panne.

Ce document reprend les parcours discutés et les transforme en cas reproductibles. Il ne constitue pas une description vérifiée de l’application : aucun code, Swagger/OpenAPI, schéma de base de données ou cahier des charges n’a été fourni.

### 1.1 Responsabilités supposées

| Service | Responsabilité | Données principales |
|---|---|---|
| Client | Gérer et consulter les clients | ID, nom, email |
| Inventory | Gérer et consulter les produits; éventuellement leur stock | ID, SKU, nom, prix, devise, stock éventuel |
| Billing | Créer et consulter des factures en utilisant les deux autres services | ID, client, lignes, quantités, prix, total, état éventuel |

### 1.2 Niveaux de test

| Niveau | Ce qu’on vérifie | Dépendances |
|---|---|---|
| API par service | Validation, opérations et persistance propres à un service | Dépendances contrôlées ou doublures selon le cas |
| Contrat | Compatibilité des requêtes/réponses entre fournisseur et consommateur | Contrats et réponses réelles; doublures en complément |
| Intégration | Interaction Billing → Client et Billing → Inventory | Services réels ou doublures ciblées pour injecter des erreurs |
| E2E API | Parcours métier via le point d’entrée du système | Tous les services et bases réels; aucune doublure |
| Sécurité | Authentification, rôles et isolation si présents | Comptes de test et politique d’accès réelle |
| Performance | Latence, charge et reprise selon des objectifs définis | Environnement dédié représentatif |

Un appel API seul ne prouve pas la cohérence globale. Pour les créations, on contrôle aussi la relecture, les références, le montant et les effets sur le stock si ce dernier est géré.

Les tests unitaires internes aux méthodes, les parcours UI, les paiements, l’envoi d’emails, la TVA, les remises et les avoirs ne sont pas spécifiés ici. Leur ajout nécessite des règles et interfaces correspondantes.

## 2. Hypothèses et décisions métier

### 2.1 Deux profils possibles

**Profil A — Facturation sans gestion transactionnelle de stock.** Inventory fournit le catalogue et les prix. Billing enrichit/crée une facture. Créer une facture ne réserve ni ne décrémente automatiquement un stock.

**Profil B — Facturation avec réservation ou décrément de stock.** La création ou confirmation d’une facture engage une quantité disponible. Les règles proposées exigent l’absence de survente et de débit résiduel après un échec définitif. Le moment exact du débit et les états intermédiaires restent à confirmer.

Les cas portant la mention **Profil B** sont **N/A en profil A**. Un refus pour stock épuisé n’est donc pas une exigence universelle de Billing.

### 2.2 Décisions à valider avant l’exécution

| Décision | Proposition utilisée dans les exemples | Action nécessaire |
|---|---|---|
| Rôle de Billing | Création d’une facture à partir d’IDs client et produits | Confirmer les endpoints et le flux réel |
| Stock | Profil A ou profil B | Choisir le profil réel |
| Débit de stock en B | Une seule diminution de la quantité après confirmation | Définir réservation, disponibilité, confirmation et compensation |
| Facture en erreur | Aucune facture confirmée; une trace en échec/en attente peut exister si prévue | Définir les états et transitions |
| Client/produit inexistant | Demande refusée avec erreur métier | Fixer le code public exact |
| Email/SKU uniques | Unicité seulement si exigée | Confirmer index et règles de normalisation |
| Prix zéro | Aucun choix imposé | Confirmer autorisation ou rejet |
| Quantités | Entiers strictement positifs dans les exemples | Confirmer unités fractionnaires et borne Qmax |
| Prix utilisé | Prix provenant d’Inventory | Confirmer source, version et instant de capture |
| Historique | Prix et données client figés uniquement si exigé | Définir instantanés et suppressions |
| Total | Somme prix × quantité; MAD; sans taxes/remises | Remplacer si TVA, remises, arrondis ou frais |
| Lignes répétées | Rejet ou traitement déterministe | Fixer la politique exacte |
| Idempotence | Clé ou référence d’opération stable si disponible | Définir portée, durée de conservation et traitement d’un payload différent |
| Pannes | Échec maîtrisé ou attente explicite selon conception | Définir erreurs, retries, réconciliation et compensation |
| Sécurité | Tests conditionnels aux protections réellement implémentées | Définir rôles et ownership/tenant |
| Délais | Tdep, Tglobal et Tcons non chiffrés | Définir des valeurs mesurables |
| Performance | Aucun SLA inventé | Fixer charge, durée, volumes et seuils |

**Point de blocage :** un cas dont le résultat attendu dépend d’une règle non tranchée reste « Bloqué — règle à préciser ». Il devient N/A uniquement si la fonctionnalité est hors périmètre. On ne choisit pas le résultat attendu après avoir observé l’application.

Pour une opération distribuée en B, définir une issue finale unique. Une réservation déjà effectuée dont la réponse est perdue exige une réconciliation; un simple retry aveugle peut produire un double débit.

## 3. Contrats API à adapter

Les routes suivantes sont des exemples documentaires. Remplacer routes, méthodes, champs et statuts par le contrat réel avant toute automatisation.

### 3.1 Interfaces indicatives

| Service | Opération | Route illustrative | Informations à confirmer |
|---|---|---|---|
| Client | Créer | POST /clients | Champs obligatoires, unicité, réponse |
| Client | Consulter | GET /clients/{id} | Schéma et erreur si absent |
| Client | Lister | GET /clients | Pagination, filtrage, tri |
| Client | Modifier | PATCH ou PUT /clients/{id} | Partiel ou remplacement complet |
| Client | Supprimer | DELETE /clients/{id} | Suppression, désactivation, références |
| Inventory | Créer | POST /products | Prix, devise, SKU, stock éventuel |
| Inventory | Consulter/lister | GET /products/{id}; GET /products | Schémas et pagination |
| Inventory | Modifier | PATCH ou PUT /products/{id} | Prix, version et autres champs |
| Inventory | Gérer le stock | Route à identifier | Ajustement/remplacement, réservation et concurrence |
| Inventory | Supprimer | DELETE /products/{id} | Références et réservations |
| Billing | Créer | POST /invoices | Client, lignes, cycle de création |
| Billing | Consulter | GET /invoices/{id} | Données figées ou références courantes |

### 3.2 Statuts publics

| Situation | Exemples possibles | Attendu à fixer |
|---|---|---|
| Création synchrone | 201 | Statut exact et ressource créée |
| Création asynchrone | 202 | ID d’opération, états finaux, méthode de suivi |
| Lecture réussie | 200 | Corps conforme |
| Suppression réussie | 204 ou réponse documentée | Comportement de relecture |
| Validation invalide | 400 ou 422 | Code, champ et message |
| Ressource absente | 404 ou erreur métier documentée | Ne pas imposer le statut de la dépendance à Billing |
| Conflit métier | 409 ou autre code documenté | Unicité, stock ou conflit de version |
| Non authentifié | 401 si contrat correspondant | Politique réelle |
| Non autorisé | 403, ou 404 si existence masquée | Politique réelle |
| Dépendance indisponible/timeout | 502, 503, 504 ou autre traitement prévu | Mapping de Billing/gateway et délai |

Une panne de Billing appelé directement peut produire une erreur de connexion et aucune réponse HTTP. Une gateway peut, elle, renvoyer un statut HTTP. Le test doit préciser le point d’entrée.

### 3.3 Exemple de création de facture

Les IDs ci-dessous illustrent un mapping C001 → 10, P001 → 5, P002 → 6. Les tests récupèrent les véritables IDs à la création.

```json
{
  "clientId": 10,
  "items": [
    {"productId": 5, "quantity": 2},
    {"productId": 6, "quantity": 3}
  ]
}
```

Résultat métier attendu pour le jeu sans taxes/remises :

- P001 : 200.00 × 2 = 400.00 MAD.
- P002 : 50.00 × 3 = 150.00 MAD.
- Total : 550.00 MAD.
- En B seulement : stock final P001=8, P002=2 après confirmation.

Ne pas envoyer un prix calculé côté test si l’API attend uniquement les IDs et quantités. Vérifier le prix provenant de la source métier.

### 3.4 Assertions communes

Pour chaque réponse, contrôler le statut exact défini, le type de contenu si une réponse avec corps est prévue, le schéma, les types, les champs métier et la persistance. Pour un rejet, contrôler également l’absence d’effets indésirables.

Les schémas d’erreur peuvent varier; relever au minimum le code métier, le message exploitable et le champ en erreur quand applicable. Le requestId/correlationId n’est obligatoire que s’il fait partie du contrat.

## 4. Exigences et critères d’acceptation

Les IDs EX ci-dessous représentent des **exigences proposées pour ce plan**, pas des exigences validées de ton application. Les résultats attendus des cas associés constituent leurs critères d’acceptation détaillés.

| ID | Exigence proposée |
|---|---|
| EX-CLI-01 | Client valide et champs contrôlés |
| EX-CLI-02 | Unicité de l’email si exigée |
| EX-CLI-03 | Consultation et liste des clients |
| EX-CLI-04 | Modification et suppression contrôlées des clients |
| EX-INV-01 | Produit valide, prix et champs contrôlés |
| EX-INV-02 | Unicité du SKU si exigée |
| EX-INV-03 | Consultation et liste des produits |
| EX-INV-04 | Modification et suppression contrôlées des produits |
| EX-BIL-01 | Création d’une facture pour un client et des produits valides |
| EX-BIL-02 | Calcul exact des lignes, du total et des devises |
| EX-BIL-03 | Rejet des références client/produit inexistantes |
| EX-BIL-04 | Lignes et quantités valides |
| EX-BIL-05 | Politique explicite pour les lignes produit répétées |
| EX-BIL-06 | Consultation des factures |
| EX-STK-01 | Stock valide et consultable — profil B |
| EX-STK-02 | Disponibilité et décrément conforme — profil B |
| EX-STK-03 | Cohérence et compensation des effets — profil B |
| EX-STK-04 | Prévention de la survente concurrente — profil B |
| EX-HIS-01 | Politique explicite de conservation historique |
| EX-INT-01 | Compatibilité des contrats interservices |
| EX-INT-02 | Gestion des pannes, réponses invalides et timeouts |
| EX-IDEM-01 | Prévention des doublons et réconciliation des opérations incertaines |
| EX-E2E-01 | Parcours réel complet et reprise après panne |
| EX-SEC-01 | Authentification et permissions si implémentées |
| EX-SEC-02 | Isolation des ressources si propriété/multitenant |
| EX-SEC-03 | Erreurs publiques sans secrets ni détails sensibles |
| EX-PERF-01 | Latence, débit et stabilité selon objectifs validés |

## 5. Environnement et jeux de données

### 5.1 Préconditions générales

- Utiliser un environnement de test isolé et autorisé pour les pannes et la charge.
- Relever les versions des trois services, des contrats et de la configuration.
- Disposer d’un moyen de créer et relire les données; utiliser des accès base en lecture seulement si nécessaires et autorisés.
- Réserver un identifiant RUN unique par exécution, dans les emails, SKU et références.
- Vérifier la disponibilité des services pour les cas nominaux.
- Pour les tests asynchrones, disposer d’un moyen de lire l’état final et la disponibilité réelle du stock.
- Définir les doublures ou points d’injection pour les cas de panne, puis toujours les restaurer.

### 5.2 Données de référence

| Alias | Données | Utilisation |
|---|---|---|
| C001 | Nom Client QA; email qa.c001.RUN@example.test | Client valide |
| C002 | Nom Client QA 2; email qa.c002.RUN@example.test | Client non référencé / concurrence |
| C999 | ID confirmé absent | Cas négatif |
| P001 | SKU QA-RUN-P001; prix 200.00 MAD; stock 10 en B | Facturation principale |
| P002 | SKU QA-RUN-P002; prix 50.00 MAD; stock 5 en B | Facture multiligne |
| P003 | SKU QA-RUN-P003; prix 30.00 MAD; stock 0 en B | Stock épuisé |
| P004 | SKU QA-RUN-P004; prix 19.99 MAD; stock 10 en B | Calcul décimal |
| P999 | ID confirmé absent | Produit inexistant |
| F001 | Facture créée pour C001, P001×2, total 400.00 MAD | Consultation et historique |
| F999 | ID confirmé absent | Facture inexistante |
| K1 | Référence/clé neuve propre à RUN | Idempotence |
| U1/U2 | Comptes synthétiques avec droits distincts | Autorisations |

Chaque cas repart de son propre état initial. Ne pas créer F001 dans un setup commun qui modifierait le stock initial des autres cas. C001, P001 et F001 sont des alias logiques, pas des IDs à coder en dur.

### 5.3 Nettoyage et isolation

Après chaque cas : conserver les preuves, restaurer les services et injections, supprimer les données de RUN ou réinitialiser le dataset dans l’environnement isolé, puis vérifier l’absence de réservations résiduelles.

Si une facture ne peut pas être supprimée, utiliser un dataset/tenant jetable ou une réinitialisation dédiée. Ne pas contourner les règles métier par une suppression directe en base sans procédure de test prévue.

Les tests de concurrence et de panne ne partagent pas leurs produits avec les cas nominaux. Ne pas nettoyer avant d’avoir vérifié l’état final.

## 6. Catalogue des scénarios

Un **scénario** décrit le parcours ou l’objectif métier. Un **cas de test** fixe un état initial, des données, une séquence d’actions et un résultat vérifiable. Une ligne de variante se lance indépendamment avec un reset entre les exécutions.

| Scénario | Objectif | Services |
|---|---|---|
| SC-CLI-01 | Enregistrer un client et valider ses données | Client |
| SC-CLI-02 | Consulter et lister les clients | Client |
| SC-CLI-03 | Modifier ou supprimer un client sans incohérence | Client / Billing |
| SC-INV-01 | Enregistrer un produit et valider ses données | Inventory |
| SC-INV-02 | Consulter et lister le catalogue | Inventory |
| SC-INV-03 | Modifier ou supprimer un produit | Inventory / Billing |
| SC-STK-01 | Consulter et ajuster un stock valide | Inventory — B |
| SC-STK-02 | Facturer seulement une quantité disponible | Billing / Inventory — B |
| SC-BIL-01 | Créer une facture simple ou multiligne | Billing / Client / Inventory |
| SC-BIL-02 | Rejeter une demande de facturation invalide | Billing |
| SC-BIL-03 | Calculer les montants et traiter les lignes | Billing / Inventory |
| SC-BIL-04 | Consulter les factures et préserver leur historique | Billing / Client / Inventory |
| SC-INT-01 | Échanger des données conformes entre services | Billing / Client / Inventory |
| SC-INT-02 | Gérer les pannes et les réponses invalides | Billing / dépendances |
| SC-INT-03 | Garantir une issue cohérente malgré retries, pannes et concurrence | Billing / Inventory |
| SC-E2E-01 | Réaliser le parcours métier nominal | Tous les services réels |
| SC-E2E-02 | Réaliser un parcours avec échec et reprise | Tous les services réels |
| SC-E2E-03 | Conserver les données historiques après modification | Tous les services réels |
| SC-SEC-01 | Limiter l’accès selon authentification et rôles | Routes protégées |
| SC-SEC-02 | Protéger les ressources et les informations exposées | Routes protégées / erreurs |
| SC-PERF-01 | Mesurer performances et stabilité sous charge | Système |

## 7. Cas de test détaillés

### Conventions d’exécution

- **P1 :** vérification critique pour le parcours et l’intégrité des données.
- **P2 :** vérification complémentaire ou opération secondaire, à reprioriser selon le produit.
- **Statut initial :** Non exécuté pour tous les cas.
- **Verdicts :** Réussi / Échoué / Bloqué / N/A / Non exécuté.
- **Préconditions communes :** section 5 + conditions spécifiques indiquées.
- **Nettoyage commun :** section 5.3; restauration obligatoire des pannes et doublures.
- **Preuves communes :** requête/réponse expurgées, IDs créés, relectures avant/après, horodatage et version. Pour stock/résilience, ajouter réservations/états et traces corrélées accessibles.
- **Asynchrone :** le premier statut accepté ne prouve pas la réussite; attendre l’état final par polling borné à Tcons. Tout dépassement de la borne validée échoue.

Les variantes regroupées sous un ID sont des exécutions distinctes, à noter par suffixe (exemple TC-BIL-06-v1 et v2). Les 65 cas décrivent donc plus de 65 exécutions possibles.

### Client Service

#### TC-CLI-01 — Créer un client valide

| Champ | Valeur |
|---|---|
| Scénario | SC-CLI-01 |
| Exigence | EX-CLI-01 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Aucun client avec l’email du jeu de données. |
| Données | name=Client QA; email=qa.client.RUN@example.test |

**Étapes :**

1. Envoyer la création du client.
2. Relire le client à partir de l’identifiant retourné.

**Résultat attendu :** Création conforme au contrat; ID non vide; nom et email restitués correctement; un seul enregistrement créé.

#### TC-CLI-02 — Refuser un email mal formé

| Champ | Valeur |
|---|---|
| Scénario | SC-CLI-01 |
| Exigence | EX-CLI-01 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Aucun client cible. |
| Données | Exécutions séparées: email=abc; abc@; @example.test |

**Étapes :**

1. Envoyer une création pour chaque valeur, en réinitialisant les données.
2. Rechercher un éventuel enregistrement correspondant.

**Résultat attendu :** Erreur de validation conforme au contrat; champ email identifié; aucun client créé.

#### TC-CLI-03 — Refuser les champs obligatoires absents

| Champ | Valeur |
|---|---|
| Scénario | SC-CLI-01 |
| Exigence | EX-CLI-01 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Liste des champs obligatoires confirmée. |
| Données | Exécutions séparées: name absent; email absent; name=null; email=null |

**Étapes :**

1. Retirer un seul champ obligatoire ou lui donner null.
2. Envoyer la création.
3. Contrôler les données persistées.

**Résultat attendu :** Chaque variante invalide est rejetée; aucun client partiel créé.

#### TC-CLI-04 — Contrôler les bornes de longueur

| Champ | Valeur |
|---|---|
| Scénario | SC-CLI-01 |
| Exigence | EX-CLI-01 |
| Priorité | P2 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Bornes Lmin/Lmax confirmées pour le nom. |
| Données | Nom de longueur Lmin−1, Lmin, Lmax, Lmax+1; email valide unique |

**Étapes :**

1. Exécuter une création indépendante pour chaque longueur.
2. Relire les créations acceptées.

**Résultat attendu :** Bornes incluses acceptées si le contrat les définit ainsi; hors bornes rejeté; aucune troncature silencieuse.

#### TC-CLI-05 — Traiter un email déjà utilisé

| Champ | Valeur |
|---|---|
| Scénario | SC-CLI-01 |
| Exigence | EX-CLI-02 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Conditionnel: unicité email |
| Préconditions spécifiques | C001 existe. |
| Données | email=C001.email |

**Étapes :**

1. Créer un autre client avec le même email.
2. Lister ou rechercher les clients portant cet email.

**Résultat attendu :** Si unicité requise: conflit métier; client initial intact; aucun doublon. Sinon marquer ce cas N/A et définir le comportement autorisé.

#### TC-CLI-06 — Consulter un client existant

| Champ | Valeur |
|---|---|
| Scénario | SC-CLI-02 |
| Exigence | EX-CLI-03 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | C001 existe. |
| Données | clientId=C001 |

**Étapes :**

1. Lire le client par son ID.
2. Comparer les champs au jeu initial.

**Résultat attendu :** Client correct restitué; aucun champ obligatoire absent; aucune mutation des données.

#### TC-CLI-07 — Consulter un client inexistant

| Champ | Valeur |
|---|---|
| Scénario | SC-CLI-02 |
| Exigence | EX-CLI-03 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | C999 est confirmé absent. |
| Données | clientId=C999 |

**Étapes :**

1. Lire C999.
2. Vérifier le code et le corps d’erreur.

**Résultat attendu :** Erreur de ressource introuvable conforme au contrat; absence de données d’un autre client.

#### TC-CLI-08 — Modifier un client

| Champ | Valeur |
|---|---|
| Scénario | SC-CLI-03 |
| Exigence | EX-CLI-04 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | C001 existe; email modifié unique. |
| Données | name=Client QA Modifié; email=qa.modified.RUN@example.test |

**Étapes :**

1. Modifier C001 selon la méthode du contrat.
2. Relire C001.

**Résultat attendu :** Champs ciblés modifiés; ID stable; champs non ciblés conservés si modification partielle.

#### TC-CLI-09 — Refuser une modification invalide

| Champ | Valeur |
|---|---|
| Scénario | SC-CLI-03 |
| Exigence | EX-CLI-04 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Capturer C001 avant modification. |
| Données | email=adresse-invalide |

**Étapes :**

1. Modifier C001 avec cet email.
2. Relire C001.

**Résultat attendu :** Validation rejetée; état initial intégralement conservé; aucune modification partielle.

#### TC-CLI-10 — Supprimer un client sans facture

| Champ | Valeur |
|---|---|
| Scénario | SC-CLI-03 |
| Exigence | EX-CLI-04 |
| Priorité | P2 |
| Niveau | API |
| Applicabilité | Conditionnel: suppression exposée |
| Préconditions spécifiques | C002 existe sans référence Billing. |
| Données | clientId=C002 |

**Étapes :**

1. Supprimer C002.
2. Relire C002.
3. Répéter la suppression si prévu au contrat.

**Résultat attendu :** Suppression physique ou désactivation conforme au contrat; comportement de relecture et de répétition documenté; pas d’impact sur C001.

#### TC-CLI-11 — Supprimer un client référencé

| Champ | Valeur |
|---|---|
| Scénario | SC-CLI-03 |
| Exigence | EX-HIS-01 |
| Priorité | P1 |
| Niveau | Intégration |
| Applicabilité | Conditionnel: suppression et factures |
| Préconditions spécifiques | Une facture F001 référence C001. |
| Données | clientId=C001; invoiceId=F001 |

**Étapes :**

1. Tenter de supprimer C001.
2. Relire F001 et son client ou son instantané.

**Résultat attendu :** Politique validée appliquée: refus, désactivation ou conservation historique; facture toujours consultable; aucune référence incohérente.

#### TC-CLI-12 — Lister et paginer les clients

| Champ | Valeur |
|---|---|
| Scénario | SC-CLI-02 |
| Exigence | EX-CLI-03 |
| Priorité | P2 |
| Niveau | API |
| Applicabilité | Conditionnel: pagination exposée |
| Préconditions spécifiques | Créer 3 clients de test distincts. |
| Données | pageSize=2; tri stable convenu |

**Étapes :**

1. Lire toutes les pages avec le tri convenu.
2. Comparer les IDs à l’ensemble créé dans un environnement isolé.

**Résultat attendu :** Chaque client apparaît une seule fois; taille et métadonnées conformes; aucun oubli ni doublon entre pages.

### Inventory Service

#### TC-INV-01 — Créer un produit valide

| Champ | Valeur |
|---|---|
| Scénario | SC-INV-01 |
| Exigence | EX-INV-01 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | SKU de test absent. |
| Données | sku=QA-RUN-PNEW; name=Produit QA; unitPrice=200.00; currency=MAD; stock=10 si supporté |

**Étapes :**

1. Créer le produit.
2. Relire par son ID.

**Résultat attendu :** ID créé; SKU, nom, prix et devise exacts; stock initial correct si exposé.

#### TC-INV-02 — Refuser les champs produit obligatoires absents

| Champ | Valeur |
|---|---|
| Scénario | SC-INV-01 |
| Exigence | EX-INV-01 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Champs obligatoires confirmés. |
| Données | Exécutions séparées: name absent; unitPrice absent; currency absente si requise |

**Étapes :**

1. Omettre un seul champ par requête.
2. Créer le produit.
3. Chercher un éventuel produit créé.

**Résultat attendu :** Rejet explicite; aucun produit incomplet persisté.

#### TC-INV-03 — Valider le prix

| Champ | Valeur |
|---|---|
| Scénario | SC-INV-01 |
| Exigence | EX-INV-01 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Politique prix nul et précision confirmée. |
| Données | Prix −0.01; 0; 0.01; valeur au-delà de la précision ou du plafond convenu |

**Étapes :**

1. Créer un produit distinct pour chaque prix.
2. Relire les valeurs acceptées.

**Résultat attendu :** Prix négatif rejeté; prix zéro accepté ou rejeté selon règle validée; décimales et dépassements traités explicitement; pas d’arrondi implicite non prévu.

#### TC-INV-04 — Traiter un SKU déjà utilisé

| Champ | Valeur |
|---|---|
| Scénario | SC-INV-01 |
| Exigence | EX-INV-02 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Conditionnel: unicité SKU |
| Préconditions spécifiques | P001 existe. |
| Données | sku=P001.sku |

**Étapes :**

1. Créer un deuxième produit de même SKU.
2. Relire P001 et rechercher le SKU.

**Résultat attendu :** Si SKU unique: conflit; aucun doublon; P001 intact. Sinon N/A et règle alternative à documenter.

#### TC-INV-05 — Consulter un produit existant

| Champ | Valeur |
|---|---|
| Scénario | SC-INV-02 |
| Exigence | EX-INV-03 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | P001 existe. |
| Données | productId=P001 |

**Étapes :**

1. Lire P001.
2. Comparer les champs au jeu initial.

**Résultat attendu :** Nom, prix, devise et stock éventuel exacts; aucune mutation.

#### TC-INV-06 — Consulter un produit inexistant

| Champ | Valeur |
|---|---|
| Scénario | SC-INV-02 |
| Exigence | EX-INV-03 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | P999 absent. |
| Données | productId=P999 |

**Étapes :**

1. Lire P999.
2. Vérifier l’erreur.

**Résultat attendu :** Ressource introuvable selon contrat; aucune donnée de P001 retournée.

#### TC-INV-07 — Modifier le prix d’un produit

| Champ | Valeur |
|---|---|
| Scénario | SC-INV-03 |
| Exigence | EX-INV-04 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | P001.unitPrice=200.00. |
| Données | unitPrice=210.00 |

**Étapes :**

1. Modifier le prix.
2. Relire P001.

**Résultat attendu :** Prix 210.00 enregistré exactement; autres champs préservés.

#### TC-INV-08 — Mettre à jour le stock

| Champ | Valeur |
|---|---|
| Scénario | SC-STK-01 |
| Exigence | EX-STK-01 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil B |
| Préconditions spécifiques | P001.stock=10; API stock disponible. |
| Données | Nouveau stock=15 ou ajustement=+5 selon contrat |

**Étapes :**

1. Appliquer l’opération exacte exposée.
2. Relire le stock.
3. Vérifier l’absence de mutation des autres produits.

**Résultat attendu :** Stock final 15; sémantique remplacement/ajustement respectée; P002 inchangé.

#### TC-INV-09 — Refuser un stock invalide

| Champ | Valeur |
|---|---|
| Scénario | SC-STK-01 |
| Exigence | EX-STK-01 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil B |
| Préconditions spécifiques | Capturer le stock de P001. |
| Données | Stock −1; valeur fractionnaire 1.5 si unités indivisibles; null |

**Étapes :**

1. Exécuter chaque variante séparément.
2. Relire le stock après chaque rejet.

**Résultat attendu :** Stock négatif rejeté; autres valeurs selon contrat; état initial préservé.

#### TC-INV-10 — Consulter un stock épuisé

| Champ | Valeur |
|---|---|
| Scénario | SC-STK-01 |
| Exigence | EX-STK-01 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil B |
| Préconditions spécifiques | P003 existe avec stock=0. |
| Données | productId=P003 |

**Étapes :**

1. Consulter le produit et son stock.

**Résultat attendu :** Produit consultable; stock 0 correctement restitué; ne pas confondre stock vide et produit inexistant.

#### TC-INV-11 — Supprimer un produit non référencé

| Champ | Valeur |
|---|---|
| Scénario | SC-INV-03 |
| Exigence | EX-INV-04 |
| Priorité | P2 |
| Niveau | API |
| Applicabilité | Conditionnel: suppression exposée |
| Préconditions spécifiques | Produit PNEW sans facture ni réservation. |
| Données | productId=PNEW |

**Étapes :**

1. Supprimer ou désactiver PNEW.
2. Relire PNEW.
3. Relire P001.

**Résultat attendu :** Politique de suppression appliquée; P001 intact; aucun stock/réservation orphelin.

#### TC-INV-12 — Lister et paginer les produits

| Champ | Valeur |
|---|---|
| Scénario | SC-INV-02 |
| Exigence | EX-INV-03 |
| Priorité | P2 |
| Niveau | API |
| Applicabilité | Conditionnel: pagination exposée |
| Préconditions spécifiques | P001, P002 et P003 existent dans un environnement isolé. |
| Données | pageSize=2; tri stable convenu |

**Étapes :**

1. Parcourir toutes les pages.
2. Comparer les IDs avec les données préparées.

**Résultat attendu :** Aucun doublon ni oubli; prix et stock éventuel cohérents.

### Billing Service

#### TC-BIL-01 — Créer une facture simple

| Champ | Valeur |
|---|---|
| Scénario | SC-BIL-01 |
| Exigence | EX-BIL-01 |
| Priorité | P1 |
| Niveau | API/Intégration |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | C001, P001 valides; stock 10 en profil B. |
| Données | clientId=C001; P001×2 |

**Étapes :**

1. Créer la facture.
2. Relire la facture.
3. Relire le stock si profil B.

**Résultat attendu :** Facture créée une seule fois; bon client et produit; quantité 2; prix unitaire 200.00; total 400.00 MAD; stock final 8 en B selon cycle validé.

#### TC-BIL-02 — Créer une facture à plusieurs lignes

| Champ | Valeur |
|---|---|
| Scénario | SC-BIL-01 |
| Exigence | EX-BIL-02 |
| Priorité | P1 |
| Niveau | API/Intégration |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | C001, P001 et P002 valides. |
| Données | P001×2; P002×3 |

**Étapes :**

1. Créer la facture.
2. Relire chaque ligne et le total.
3. Contrôler les stocks en B.

**Résultat attendu :** Lignes 400.00 et 150.00; total 550.00 MAD hors taxes/remises; en B stocks P001=8 et P002=2 après confirmation.

#### TC-BIL-03 — Refuser un client inexistant

| Champ | Valeur |
|---|---|
| Scénario | SC-BIL-02 |
| Exigence | EX-BIL-03 |
| Priorité | P1 |
| Niveau | API/Intégration |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | C999 absent; P001 valide. |
| Données | clientId=C999; P001×1 |

**Étapes :**

1. Créer la facture.
2. Chercher une facture liée au requestId de test.
3. Relire le stock.

**Résultat attendu :** Erreur métier client inexistant; aucune facture validée; aucun débit/réservation résiduel en B.

#### TC-BIL-04 — Refuser un produit inexistant

| Champ | Valeur |
|---|---|
| Scénario | SC-BIL-02 |
| Exigence | EX-BIL-03 |
| Priorité | P1 |
| Niveau | API/Intégration |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | C001 existe; P999 absent. |
| Données | clientId=C001; P999×1 |

**Étapes :**

1. Créer la facture.
2. Rechercher les effets persistés.

**Résultat attendu :** Erreur produit inexistant; aucune facture validée; aucune réservation résiduelle en B.

#### TC-BIL-05 — Refuser une facture sans ligne

| Champ | Valeur |
|---|---|
| Scénario | SC-BIL-02 |
| Exigence | EX-BIL-04 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | C001 existe. |
| Données | Exécutions séparées: items=[]; items absent; items=null |

**Étapes :**

1. Envoyer chaque variante.
2. Contrôler l’absence de facture créée.

**Résultat attendu :** Rejet de validation; pas de facture vide.

#### TC-BIL-06 — Refuser les quantités nulles ou négatives

| Champ | Valeur |
|---|---|
| Scénario | SC-BIL-02 |
| Exigence | EX-BIL-04 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | C001 et P001 existent. |
| Données | Exécutions séparées: quantity=0; −1 |

**Étapes :**

1. Créer une facture pour chaque quantité.
2. Relire stock et factures.

**Résultat attendu :** Rejet de validation; aucune facture ni mutation de stock.

#### TC-BIL-07 — Valider le type et les bornes de quantité

| Champ | Valeur |
|---|---|
| Scénario | SC-BIL-02 |
| Exigence | EX-BIL-04 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Unités indivisibles et quantité maximale Qmax confirmées; stock suffisant en B. |
| Données | quantity absente; null; 1.5; chaîne abc; Qmax; Qmax+1 |

**Étapes :**

1. Exécuter chaque variante isolément.
2. Relire les résultats acceptés.

**Résultat attendu :** Types invalides rejetés; fraction rejetée si unités indivisibles; Qmax accepté si autres règles satisfaites; Qmax+1 rejeté sans dépassement arithmétique.

#### TC-BIL-08 — Refuser une quantité supérieure au stock

| Champ | Valeur |
|---|---|
| Scénario | SC-STK-02 |
| Exigence | EX-STK-02 |
| Priorité | P1 |
| Niveau | API/Intégration |
| Applicabilité | Profil B |
| Préconditions spécifiques | P001.stock=10. |
| Données | P001×11 |

**Étapes :**

1. Créer la facture.
2. Relire stock et éventuelles réservations.

**Résultat attendu :** Erreur stock insuffisant; stock final 10; aucune facture confirmée ni réservation résiduelle.

#### TC-BIL-09 — Accepter une quantité égale au stock

| Champ | Valeur |
|---|---|
| Scénario | SC-STK-02 |
| Exigence | EX-STK-02 |
| Priorité | P1 |
| Niveau | API/Intégration |
| Applicabilité | Profil B |
| Préconditions spécifiques | P001.stock=10. |
| Données | P001×10 |

**Étapes :**

1. Créer et attendre la confirmation.
2. Relire le stock.

**Résultat attendu :** Facture confirmée; total 2000.00 MAD; stock final 0; aucune valeur négative.

#### TC-BIL-10 — Refuser la facturation d’un produit épuisé

| Champ | Valeur |
|---|---|
| Scénario | SC-STK-02 |
| Exigence | EX-STK-02 |
| Priorité | P1 |
| Niveau | API/Intégration |
| Applicabilité | Profil B |
| Préconditions spécifiques | P003.stock=0. |
| Données | P003×1 |

**Étapes :**

1. Créer la facture.
2. Relire stock et factures.

**Résultat attendu :** Refus stock insuffisant; stock reste 0; aucune facture confirmée.

#### TC-BIL-11 — Calculer les montants décimaux

| Champ | Valeur |
|---|---|
| Scénario | SC-BIL-03 |
| Exigence | EX-BIL-02 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | P004.price=19.99 MAD; stock suffisant si B. |
| Données | P004×3 |

**Étapes :**

1. Créer la facture.
2. Comparer ligne et total à un calcul décimal indépendant.

**Résultat attendu :** Total exact 59.97 MAD pour le jeu sans taxes/remises; aucune erreur binaire ou arrondi non prévu.

#### TC-BIL-12 — Traiter plusieurs lignes du même produit

| Champ | Valeur |
|---|---|
| Scénario | SC-BIL-03 |
| Exigence | EX-BIL-05 |
| Priorité | P1 |
| Niveau | API/Intégration |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | P001 stock 10 si B; politique doublons confirmée. |
| Données | P001×2 puis P001×3 dans la même requête |

**Étapes :**

1. Créer la facture.
2. Contrôler les lignes, le total et le stock en B.

**Résultat attendu :** Politique appliquée: rejet explicite ou regroupement/conservation contrôlée; si accepté, quantité totale 5, total 1000.00 et stock final 5 en B.

#### TC-BIL-13 — Consulter une facture existante

| Champ | Valeur |
|---|---|
| Scénario | SC-BIL-04 |
| Exigence | EX-BIL-06 |
| Priorité | P1 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | F001 existe, total 400.00 MAD. |
| Données | invoiceId=F001 |

**Étapes :**

1. Lire F001.
2. Comparer ID, client, lignes, total et état.

**Résultat attendu :** Facture restituée conformément à sa version persistée; aucune mutation.

#### TC-BIL-14 — Consulter une facture inexistante

| Champ | Valeur |
|---|---|
| Scénario | SC-BIL-04 |
| Exigence | EX-BIL-06 |
| Priorité | P2 |
| Niveau | API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | F999 absent. |
| Données | invoiceId=F999 |

**Étapes :**

1. Lire F999.

**Résultat attendu :** Erreur ressource introuvable conforme au contrat; aucune autre facture divulguée.

#### TC-BIL-15 — Conserver la cohérence historique du prix

| Champ | Valeur |
|---|---|
| Scénario | SC-BIL-04 |
| Exigence | EX-HIS-01 |
| Priorité | P1 |
| Niveau | Intégration |
| Applicabilité | Conditionnel: prix figé |
| Préconditions spécifiques | F001 créée pour P001×2 à 200.00. |
| Données | P001 nouveau prix=210.00 |

**Étapes :**

1. Modifier le prix dans Inventory.
2. Relire F001.
3. Créer une nouvelle facture pour P001×2.

**Résultat attendu :** Si prix figé validé: F001 reste 400.00; nouvelle facture 420.00; aucune réécriture de l’historique.

#### TC-BIL-16 — Refuser ou traiter les devises incompatibles

| Champ | Valeur |
|---|---|
| Scénario | SC-BIL-03 |
| Exigence | EX-BIL-02 |
| Priorité | P1 |
| Niveau | API/Intégration |
| Applicabilité | Conditionnel: devises exposées |
| Préconditions spécifiques | Un produit MAD et un produit EUR; règle multidevise validée. |
| Données | Facture avec une ligne MAD et une ligne EUR |

**Étapes :**

1. Tenter la création.
2. Inspecter devise et total ou l’erreur.

**Résultat attendu :** Aucune addition brute de devises; rejet si facture monodevise; si conversion supportée, taux/source/date et arrondi conformes à la règle définie.

### Intégration, résilience et cohérence

#### TC-INT-01 — Respecter le contrat Billing → Client

| Champ | Valeur |
|---|---|
| Scénario | SC-INT-01 |
| Exigence | EX-INT-01 |
| Priorité | P1 |
| Niveau | Contrat/Intégration |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Réponse Client réelle connue; environnement de contrat disponible. |
| Données | clientId=C001 |

**Étapes :**

1. Vérifier la requête émise par Billing.
2. Valider une réponse Client réelle contre le schéma convenu.
3. Créer une facture.

**Résultat attendu :** Route, méthode, ID, types et champs requis compatibles; Billing exploite le bon client.

#### TC-INT-02 — Respecter le contrat Billing → Inventory

| Champ | Valeur |
|---|---|
| Scénario | SC-INT-01 |
| Exigence | EX-INT-01 |
| Priorité | P1 |
| Niveau | Contrat/Intégration |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Réponse produit réelle connue. |
| Données | productId=P001 |

**Étapes :**

1. Vérifier l’appel de Billing.
2. Valider schéma prix/devise/stock éventuel.
3. Créer une facture.

**Résultat attendu :** Types et précision du prix compatibles; bon produit facturé; aucune interprétation erronée des champs.

#### TC-INT-03 — Gérer une réponse Client invalide

| Champ | Valeur |
|---|---|
| Scénario | SC-INT-02 |
| Exigence | EX-INT-02 |
| Priorité | P1 |
| Niveau | Intégration avec doublure |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Doublure de Client contrôlée. |
| Données | Réponses séparées: JSON mal formé; champ ID requis absent; type invalide |

**Étapes :**

1. Configurer une seule réponse invalide.
2. Créer une facture.
3. Inspecter réponse Billing et persistance.

**Résultat attendu :** Erreur d’intégration maîtrisée; pas de facture validée avec données client incomplètes; pas d’effet de stock résiduel en B.

#### TC-INT-04 — Gérer une réponse Inventory invalide

| Champ | Valeur |
|---|---|
| Scénario | SC-INT-02 |
| Exigence | EX-INT-02 |
| Priorité | P1 |
| Niveau | Intégration avec doublure |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Doublure Inventory contrôlée. |
| Données | Prix absent; prix de type objet; devise incompatible avec le contrat |

**Étapes :**

1. Configurer une variante.
2. Créer une facture.
3. Inspecter persistance et stock.

**Résultat attendu :** Aucun prix par défaut silencieux; erreur contrôlée; pas de facture incorrecte ni réservation résiduelle.

#### TC-INT-05 — Gérer Client indisponible

| Champ | Valeur |
|---|---|
| Scénario | SC-INT-02 |
| Exigence | EX-INT-02 |
| Priorité | P1 |
| Niveau | Intégration |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Simuler la panne dans un environnement isolé. |
| Données | C001; P001×1; panne avant récupération client |

**Étapes :**

1. Rendre Client inaccessible ou renvoyer l’erreur de panne convenue.
2. Créer la facture.
3. Relire les effets.
4. Restaurer Client.

**Résultat attendu :** Erreur publique prévue ou état en attente si prévu; jamais de confirmation injustifiée; aucune mutation de stock résiduelle; requête suivante réussit après retour du service.

#### TC-INT-06 — Gérer Inventory indisponible

| Champ | Valeur |
|---|---|
| Scénario | SC-INT-02 |
| Exigence | EX-INT-02 |
| Priorité | P1 |
| Niveau | Intégration |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Panne avant récupération/réservation produit. |
| Données | C001; P001×1 |

**Étapes :**

1. Rendre Inventory indisponible.
2. Créer la facture.
3. Contrôler Billing.
4. Restaurer Inventory.

**Résultat attendu :** Erreur/attente conforme au contrat; aucune facture confirmée sans données ou réservation requises; reprise correcte après restauration.

#### TC-INT-07 — Gérer un timeout d’une dépendance

| Champ | Valeur |
|---|---|
| Scénario | SC-INT-02 |
| Exigence | EX-INT-02 |
| Priorité | P1 |
| Niveau | Intégration |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Timeout Tdep et deadline globale Tglobal validés. |
| Données | Retard contrôlé > Tdep; variantes Client et Inventory |

**Étapes :**

1. Introduire le retard avant toute mutation, séparément pour chaque service.
2. Mesurer la réponse Billing.
3. Relire les effets.

**Résultat attendu :** Timeout identifié; réponse dans la borne Tglobal convenue, retries compris; pas de facture validée ni mutation résiduelle. Mutation réussie avec réponse perdue couverte par TC-INT-11.

#### TC-INT-08 — Gérer Billing indisponible à l’entrée

| Champ | Valeur |
|---|---|
| Scénario | SC-INT-02 |
| Exigence | EX-INT-02 |
| Priorité | P1 |
| Niveau | Système |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Point d’entrée direct ou gateway identifié; panne avant acceptation. |
| Données | Requête de création valide |

**Étapes :**

1. Arrêter Billing avant l’envoi.
2. Appeler le point d’entrée.
3. Inspecter dépendances.
4. Restaurer Billing et refaire la demande avec une nouvelle clé.

**Résultat attendu :** Si accès direct: erreur de connexion maîtrisée côté appelant; si gateway: erreur définie au contrat; aucun effet sur Client/Inventory; création possible après restauration.

#### TC-INT-09 — Annuler les effets après un échec de persistance

| Champ | Valeur |
|---|---|
| Scénario | SC-INT-03 |
| Exigence | EX-STK-03 |
| Priorité | P1 |
| Niveau | Intégration |
| Applicabilité | Profil B |
| Préconditions spécifiques | Mécanisme de compensation validé; P001.stock=10. |
| Données | P001×2; panne Billing après réservation mais avant confirmation persistée |

**Étapes :**

1. Injecter l’échec au point défini.
2. Créer la facture.
3. Attendre au plus Tcons.
4. Relire facture, stock et réservations.

**Résultat attendu :** Pas de facture confirmée; stock disponible restauré à 10; aucune réservation orpheline; compensation unique et observable. État en attente/échec conforme au cycle défini.

#### TC-INT-10 — Rejouer une création sans doublon

| Champ | Valeur |
|---|---|
| Scénario | SC-INT-03 |
| Exigence | EX-IDEM-01 |
| Priorité | P1 |
| Niveau | Intégration |
| Applicabilité | Conditionnel: idempotence supportée |
| Préconditions spécifiques | Clé d’idempotence K1 neuve; stock 10 si B. |
| Données | Deux requêtes identiques K1; P001×2 |

**Étapes :**

1. Envoyer la requête.
2. Renvoyer exactement la même avec K1.
3. Comparer IDs et compte des factures.
4. Relire le stock.

**Résultat attendu :** Une seule facture; même résultat ou réponse de rejeu prévue; stock débité une seule fois en B, final 8.

#### TC-INT-11 — Réconcilier une réservation dont la réponse est perdue

| Champ | Valeur |
|---|---|
| Scénario | SC-INT-03 |
| Exigence | EX-IDEM-01 |
| Priorité | P1 |
| Niveau | Intégration |
| Applicabilité | Profil B; mécanisme anti-doublon requis |
| Préconditions spécifiques | P001.stock=10; référence opération K1; injection après mutation Inventory. |
| Données | Réservation P001×2 réussie mais réponse coupée |

**Étapes :**

1. Faire réussir la réservation puis couper sa réponse.
2. Déclencher retry ou réconciliation selon conception.
3. Attendre Tcons.
4. Inspecter facture, stock et réservations.

**Résultat attendu :** Issue finale unique: facture confirmée et stock 8, ou opération annulée et stock 10; jamais double débit ni facture confirmée sans réservation; aucune réservation orpheline.

#### TC-INT-12 — Empêcher la survente concurrente

| Champ | Valeur |
|---|---|
| Scénario | SC-INT-03 |
| Exigence | EX-STK-04 |
| Priorité | P1 |
| Niveau | Intégration concurrente |
| Applicabilité | Profil B |
| Préconditions spécifiques | P001.stock=1; deux clients valides; clés distinctes. |
| Données | Deux créations simultanées P001×1 |

**Étapes :**

1. Synchroniser le départ des deux requêtes.
2. Attendre leurs états finaux.
3. Lire factures confirmées et stock.

**Résultat attendu :** Exactement une facture confirmée; autre demande refusée ou échouée selon cycle; stock final 0; aucune survente.

### Parcours métier E2E

#### TC-E2E-01 — Créer client, produit puis facture

| Champ | Valeur |
|---|---|
| Scénario | SC-E2E-01 |
| Exigence | EX-E2E-01 |
| Priorité | P1 |
| Niveau | E2E API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Tous les services réels disponibles; dataset RUN neuf. |
| Données | Nouveau client; nouveau produit 200.00 MAD, stock 10 en B; quantité 2 |

**Étapes :**

1. Créer un client et conserver son ID réel.
2. Créer le produit et conserver son ID.
3. Créer une facture avec ces IDs.
4. Relire facture et stock éventuel.

**Résultat attendu :** Chaîne complète réussie sans doublure; IDs cohérents; total 400.00; stock 8 en B après confirmation.

#### TC-E2E-02 — Facturer plusieurs produits

| Champ | Valeur |
|---|---|
| Scénario | SC-E2E-01 |
| Exigence | EX-E2E-01 |
| Priorité | P1 |
| Niveau | E2E API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | C001; P001 et P002 réels. |
| Données | P001×2; P002×3 |

**Étapes :**

1. Créer la facture via point d’entrée du système.
2. Relire la facture et les deux produits.

**Résultat attendu :** Total 550.00; aucune ligne perdue; client exact; stocks 8 et 2 en B.

#### TC-E2E-03 — Rejeter toute la facture si une ligne manque de stock

| Champ | Valeur |
|---|---|
| Scénario | SC-E2E-02 |
| Exigence | EX-STK-03 |
| Priorité | P1 |
| Niveau | E2E API |
| Applicabilité | Profil B |
| Préconditions spécifiques | P001.stock=10; P003.stock=0. |
| Données | P001×2; P003×1 |

**Étapes :**

1. Créer la facture multiligne.
2. Attendre état final.
3. Relire les deux stocks et réservations.

**Résultat attendu :** Aucune facture confirmée; stock P001=10 et P003=0 après compensation; aucun débit partiel résiduel.

#### TC-E2E-04 — Reprendre après panne d’une dépendance

| Champ | Valeur |
|---|---|
| Scénario | SC-E2E-02 |
| Exigence | EX-E2E-01 |
| Priorité | P1 |
| Niveau | E2E API |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Inventory réel; mécanisme de reprise validé. |
| Données | P001×2; panne puis restauration |

**Étapes :**

1. Arrêter Inventory avant le premier appel.
2. Tenter la création et enregistrer son état.
3. Restaurer Inventory.
4. Rejouer avec la même clé si supportée; sinon réconcilier avant un nouvel envoi.
5. Relire les résultats finaux.

**Résultat attendu :** Une seule facture finale pour l’opération; total correct; stock débité au plus une fois en B; aucune création répétée à l’aveugle.

#### TC-E2E-05 — Consulter la facture après modification des données source

| Champ | Valeur |
|---|---|
| Scénario | SC-E2E-03 |
| Exigence | EX-HIS-01 |
| Priorité | P1 |
| Niveau | E2E API |
| Applicabilité | Conditionnel: instantanés historiques |
| Préconditions spécifiques | Facture existante; conservation client et prix validée. |
| Données | Modifier C001.name et P001.price après facturation |

**Étapes :**

1. Créer F001.
2. Modifier nom du client et prix du produit.
3. Relire F001 et les ressources actuelles.

**Résultat attendu :** Facture conserve les informations historiques prévues; ressources courantes reflètent les changements; historique et état actuel distingués.

### Authentification et autorisations

#### TC-SEC-01 — Refuser un appel sans authentification

| Champ | Valeur |
|---|---|
| Scénario | SC-SEC-01 |
| Exigence | EX-SEC-01 |
| Priorité | P1 |
| Niveau | API sécurité |
| Applicabilité | Conditionnel: authentification activée |
| Préconditions spécifiques | Routes protégées identifiées. |
| Données | Aucun jeton; variantes lecture et création sur les 3 services |

**Étapes :**

1. Appeler chaque route protégée sans jeton.
2. Contrôler réponse et effets persistés.

**Résultat attendu :** Accès rejeté conformément au contrat; aucune donnée protégée retournée ni mutation réalisée.

#### TC-SEC-02 — Refuser un jeton invalide ou expiré

| Champ | Valeur |
|---|---|
| Scénario | SC-SEC-01 |
| Exigence | EX-SEC-01 |
| Priorité | P1 |
| Niveau | API sécurité |
| Applicabilité | Conditionnel: authentification activée |
| Préconditions spécifiques | Jetons de test, jamais des secrets réels. |
| Données | Jeton invalide puis expiré |

**Étapes :**

1. Appeler une route protégée avec chaque jeton.
2. Inspecter réponse et persistance.

**Résultat attendu :** Authentification rejetée; aucun effet; jeton non exposé dans les réponses ni preuves conservées.

#### TC-SEC-03 — Refuser une opération sans permission

| Champ | Valeur |
|---|---|
| Scénario | SC-SEC-01 |
| Exigence | EX-SEC-01 |
| Priorité | P1 |
| Niveau | API sécurité |
| Applicabilité | Conditionnel: rôles implémentés |
| Préconditions spécifiques | Compte autorisé en lecture uniquement. |
| Données | Jeton lecteur; requête création/modification |

**Étapes :**

1. Tenter les opérations d’écriture protégées.
2. Relire la ressource avec un compte autorisé.

**Résultat attendu :** Refus d’autorisation; données inchangées; code distinct de l’absence d’authentification selon contrat.

#### TC-SEC-04 — Protéger les factures d’un autre utilisateur

| Champ | Valeur |
|---|---|
| Scénario | SC-SEC-02 |
| Exigence | EX-SEC-02 |
| Priorité | P1 |
| Niveau | API sécurité |
| Applicabilité | Conditionnel: propriété ou multitenant |
| Préconditions spécifiques | U1 possède F001; U2 ne dispose d’aucun accès à cette facture. |
| Données | Jeton U2; invoiceId=F001 |

**Étapes :**

1. Lire F001 avec U2.
2. Tenter une autre action exposée sur F001.
3. Comparer avec lecture autorisée par U1.

**Résultat attendu :** Aucune fuite de facture ou données client; refus conforme à la politique, y compris existence masquée si prévue; F001 intacte.

#### TC-SEC-05 — Ne pas divulguer de détails techniques en erreur

| Champ | Valeur |
|---|---|
| Scénario | SC-SEC-02 |
| Exigence | EX-SEC-03 |
| Priorité | P2 |
| Niveau | API sécurité |
| Applicabilité | Profil A/B |
| Préconditions spécifiques | Erreur de validation et panne injectée de manière contrôlée. |
| Données | Payload invalide; dépendance indisponible |

**Étapes :**

1. Déclencher les deux erreurs.
2. Inspecter réponses publiques et logs accessibles au testeur autorisé.

**Résultat attendu :** Messages exploitables; absence de stacktrace, mot de passe, jeton ou détail de connexion dans la réponse publique; corrélation disponible si implémentée.

### Performance

#### TC-PERF-01 — Mesurer le temps de création nominal

| Champ | Valeur |
|---|---|
| Scénario | SC-PERF-01 |
| Exigence | EX-PERF-01 |
| Priorité | P2 |
| Niveau | Performance |
| Applicabilité | Conditionnel: SLA et charge validés |
| Préconditions spécifiques | Environnement dédié; données suffisantes; N, débit, durée et SLA approuvés. |
| Données | Créations valides; clés uniques; produits sans contention artificielle |

**Étapes :**

1. Préparer et chauffer l’environnement.
2. Exécuter le profil de charge convenu.
3. Collecter latences, taux d’erreur et effets métier.

**Résultat attendu :** p95/p99 et taux d’erreur dans les seuils convenus; totaux exacts; aucun doublon ou stock négatif; résultat bloqué tant que SLA non défini.

#### TC-PERF-02 — Mesurer la consultation et la pagination

| Champ | Valeur |
|---|---|
| Scénario | SC-PERF-01 |
| Exigence | EX-PERF-01 |
| Priorité | P2 |
| Niveau | Performance |
| Applicabilité | Conditionnel: SLA validé |
| Préconditions spécifiques | Volume V de clients, produits et factures; SLA lecture défini. |
| Données | Lectures réparties; pages de taille convenue |

**Étapes :**

1. Charger V ressources.
2. Exécuter le trafic lecture convenu.
3. Vérifier latences et contenu échantillonné.

**Résultat attendu :** Seuils de lecture respectés; pagination toujours exacte; pas de données mélangées entre requêtes.

#### TC-PERF-03 — Observer le système avec une dépendance lente

| Champ | Valeur |
|---|---|
| Scénario | SC-PERF-01 |
| Exigence | EX-PERF-01 |
| Priorité | P2 |
| Niveau | Performance/résilience |
| Applicabilité | Conditionnel: profil panne validé |
| Préconditions spécifiques | Tdep, Tglobal, concurrence N et seuils ressources définis. |
| Données | Charge convenue; Inventory ralenti au-delà de Tdep |

**Étapes :**

1. Appliquer la lenteur.
2. Exécuter la charge.
3. Mesurer réponses, ressources et reprise.
4. Restaurer Inventory.

**Résultat attendu :** Délais bornés; pas d’accumulation incontrôlée selon seuils validés; reprise conforme; aucune incohérence métier; pas de retries sans limite.

## 8. Matrice de traçabilité

Cette matrice relie chaque exigence proposée aux cas qui la couvrent. Une couverture documentaire ne prouve ni l’exécution ni la réussite.

| Exigence | Cas associés |
|---|---|
| EX-CLI-01 | TC-CLI-01, TC-CLI-02, TC-CLI-03, TC-CLI-04 |
| EX-CLI-02 | TC-CLI-05 |
| EX-CLI-03 | TC-CLI-06, TC-CLI-07, TC-CLI-12 |
| EX-CLI-04 | TC-CLI-08, TC-CLI-09, TC-CLI-10 |
| EX-INV-01 | TC-INV-01, TC-INV-02, TC-INV-03 |
| EX-INV-02 | TC-INV-04 |
| EX-INV-03 | TC-INV-05, TC-INV-06, TC-INV-12 |
| EX-INV-04 | TC-INV-07, TC-INV-11 |
| EX-BIL-01 | TC-BIL-01 |
| EX-BIL-02 | TC-BIL-02, TC-BIL-11, TC-BIL-16 |
| EX-BIL-03 | TC-BIL-03, TC-BIL-04 |
| EX-BIL-04 | TC-BIL-05, TC-BIL-06, TC-BIL-07 |
| EX-BIL-05 | TC-BIL-12 |
| EX-BIL-06 | TC-BIL-13, TC-BIL-14 |
| EX-STK-01 | TC-INV-08, TC-INV-09, TC-INV-10 |
| EX-STK-02 | TC-BIL-08, TC-BIL-09, TC-BIL-10 |
| EX-STK-03 | TC-INT-09, TC-E2E-03 |
| EX-STK-04 | TC-INT-12 |
| EX-HIS-01 | TC-CLI-11, TC-BIL-15, TC-E2E-05 |
| EX-INT-01 | TC-INT-01, TC-INT-02 |
| EX-INT-02 | TC-INT-03, TC-INT-04, TC-INT-05, TC-INT-06, TC-INT-07, TC-INT-08 |
| EX-IDEM-01 | TC-INT-10, TC-INT-11 |
| EX-E2E-01 | TC-E2E-01, TC-E2E-02, TC-E2E-04 |
| EX-SEC-01 | TC-SEC-01, TC-SEC-02, TC-SEC-03 |
| EX-SEC-02 | TC-SEC-04 |
| EX-SEC-03 | TC-SEC-05 |
| EX-PERF-01 | TC-PERF-01, TC-PERF-02, TC-PERF-03 |

## 9. Campagnes et exécution

### 9.1 Campagnes recommandées

| Campagne | Sélection proposée | Moment |
|---|---|---|
| Smoke nominal | TC-CLI-01, TC-CLI-06, TC-INV-01, TC-INV-05, TC-BIL-01, TC-BIL-13, TC-E2E-01 | Après déploiement sur environnement disponible |
| Non-régression API | Tous CLI, INV et BIL applicables | PR ou intégration selon durée |
| Contrats et résilience | Tous INT applicables | Changements des contrats ou campagnes dédiées |
| E2E | Tous E2E applicables | Environnement complet stable |
| Sécurité | Tous SEC applicables | Changements d’accès/authentification et avant livraison |
| Performance | Tous PERF dont objectifs validés | Environnement dédié; avant livraison ou changement majeur |

En profil B, le smoke doit également confirmer l’état final du stock dans TC-BIL-01. Chaque sélection reste indépendante et prépare ses données.

### 9.2 Ordre pratique

1. Valider profil, règles métier, contrats et valeurs des délais.
2. Préparer versions, environnement, comptes et dataset.
3. Exécuter le smoke; si l’environnement est inutilisable, bloquer les cas dépendants.
4. Exécuter les tests API nominaux, négatifs et limites.
5. Vérifier les contrats puis les intégrations.
6. Exécuter pannes, compensation, idempotence et concurrence dans le contexte isolé.
7. Exécuter E2E et sécurité applicables.
8. Lancer la performance lorsque ses objectifs sont validés.
9. Enregistrer les anomalies, retester les corrections et rejouer les cas impactés.
10. Produire le bilan et nettoyer l’environnement.

### 9.3 Critères d’entrée et de sortie proposés

**Entrée :** build identifié, contrats disponibles, règles attendues approuvées, données préparées, accès utilisables et environnement vérifié.

**Sortie proposée :**

- Tous les P1 applicables sont exécutés et réussis.
- Aucun défaut critique d’intégrité, de survente, de double création ou d’accès non autorisé reste ouvert.
- Tout P2 échoué ou non exécuté est identifié avec décision et risque documentés.
- Les cas bloqués ne sont pas assimilés à des réussites.
- Les seuils de performance sont respectés lorsque cette campagne fait partie de la livraison.
- Les données et services sont remis dans un état maîtrisé.

Ces critères sont une proposition à convenir avec l’équipe; ils ne prouvent pas qu’une livraison est autorisée.

### 9.4 Registre d’exécution à remplir

| Exécution | Version | Cas / variante | Profil | Statut | Résultat obtenu | Preuve | Anomalie |
|---|---|---|---|---|---|---|---|
| RUN-… | … | TC-BIL-01 | A/B | Non exécuté | À compléter | À compléter | — |
| RUN-… | … | TC-BIL-06-v1 | A/B | Non exécuté | À compléter | À compléter | — |
| RUN-… | … | TC-INT-12 | B | Non exécuté | À compléter | À compléter | — |

Une exécution « Bloqué » précise la cause (règle absente, panne environnement, accès manquant). Une exécution « N/A » précise le motif et le périmètre.

## 10. Automatisation

Le plan est indépendant du framework. Pour une suite Robot Framework avec RequestsLibrary, organiser les tests API par service, puis les intégrations et E2E. SeleniumLibrary est utile seulement si un parcours navigateur réel est ajouté.

### 10.1 Organisation proposée

| Emplacement | Contenu |
|---|---|
| tests/api/client/ | Cas CLI |
| tests/api/inventory/ | Cas INV |
| tests/api/billing/ | Cas BIL |
| tests/integration/ | Cas INT |
| tests/e2e/ | Cas E2E |
| tests/security/ | Cas SEC |
| performance/ | Scénarios de charge et résultats PERF |
| resources/ | Keywords métier et assertions partagées |
| data/ | Données synthétiques et paramètres par environnement |
| docs/ | Ce plan et les contrats validés |
| results/ | Rapports et preuves d’exécution |

### 10.2 Principes de mise en œuvre

- Conserver les IDs TC dans les noms ou tags des tests.
- Créer des keywords métier : « Créer un client valide », « Créer un produit », « Créer une facture », « Vérifier le total », « Vérifier le stock final ».
- Centraliser routes, authentification, timeouts et configuration hors des cas.
- Récupérer les IDs retournés; ne pas dépendre de l’ordre d’exécution.
- Faire des assertions sur le statut **et** les données, la persistance et les effets métier.
- Calculer l’attendu avec une arithmétique décimale et une règle d’arrondi validée, sans réutiliser la même logique métier que l’application.
- Distinguer retries de transport et nouvelle opération métier; utiliser une référence stable si le contrat la permet.
- Utiliser du polling borné pour les états asynchrones; éviter un délai fixe arbitraire.
- Restaurer les injections même lorsqu’une assertion échoue.
- Exclure les pannes et la charge des campagnes PR ordinaires si l’environnement n’est pas isolé.
- Masquer secrets et données sensibles dans les logs et rapports.

### 10.3 Tags suggérés

| Axe | Exemples |
|---|---|
| Service | client, inventory, billing |
| Niveau | api, contract, integration, e2e, security, performance |
| Criticité | P1, P2 |
| Sélection | smoke, regression |
| Profil | profile_A, profile_B, conditional |
| Comportement | positive, negative, boundary, resilience, concurrency, idempotence |

Les cas nécessitant des contrats non définis restent désactivés avec un motif explicite; ne pas les transformer en tests passant artificiellement.

## 11. Suivi des anomalies et bilan

### 11.1 Modèle de ticket

**Titre :** [Billing][TC-BIL-02] Total incorrect sur une facture multiligne  
**Environnement / versions :** à renseigner  
**Cas / variante / RUN :** à renseigner  
**Préconditions :** C001 existe; P001=200.00 MAD; P002=50.00 MAD  
**Étapes :** créer une facture avec P001×2 et P002×3; relire la facture  
**Attendu :** total 550.00 MAD hors taxes/remises selon règle validée  
**Obtenu :** valeur réellement observée, sans l’inventer  
**Preuves :** requête/réponse expurgées, ID facture, traces corrélées et relectures  
**Sévérité :** impact réel à qualifier avec l’équipe  
**Priorité :** urgence de résolution à décider  
**Reproductibilité :** nombre de reproductions / nombre d’essais  
**Nettoyage / impact données :** à préciser

La sévérité décrit l’impact; la priorité décrit l’urgence. Elles ne sont pas automatiquement identiques à P1/P2 du cas.

### 11.2 Bilan de campagne

| Mesure | Valeur à compléter |
|---|---|
| Campagne, RUN, dates et versions | … |
| Profil métier et périmètre sélectionné | … |
| Exécutions prévues, variantes comprises | … |
| Réussies | … |
| Échouées | … |
| Bloquées | … |
| N/A | … |
| Non exécutées | … |
| Défauts ouverts par sévérité | … |
| Risques et décisions | … |
| État du nettoyage | … |

Calculer le taux de réussite **Réussies / (Réussies + Échouées)** lorsqu’au moins une exécution a un verdict, et l’accompagner du nombre de cas bloqués/non exécutés. Ne pas annoncer « 100 % validé » si une partie du périmètre n’a pas été testée.

Pour la couverture exécutée, utiliser un dénominateur explicitement défini après exclusion des N/A; conserver les variantes dans le même niveau de comptage.

## 12. Checklist d’adaptation

- [ ] Profil A ou B choisi et documenté.
- [ ] Routes, méthodes, payloads, schémas et codes publics remplacés par ceux du projet.
- [ ] Champs obligatoires, longueurs, unicité et normalisation confirmés.
- [ ] Règles prix zéro, quantités, devise, taxes/remises et arrondis confirmées.
- [ ] Politique de lignes répétées confirmée.
- [ ] Conservation historique et suppression des ressources référencées définies.
- [ ] Cycle de facture et moment du débit/réservation définis en B.
- [ ] Compensation, reprise, retries et idempotence définis si nécessaires.
- [ ] Tdep, Tglobal, Tcons et éventuelles bornes de quantité renseignés.
- [ ] Authentification, rôles et isolation confirmés; cas non applicables marqués.
- [ ] Volumes, concurrence, durée et objectifs de performance fixés.
- [ ] Jeux de données, nettoyage et injection de panne opérationnels.
- [ ] Matrice EX remplacée ou reliée aux vrais tickets/exigences.
- [ ] Variantes créées comme exécutions distinctes dans l’outil de gestion.
- [ ] Campagnes, critères de sortie et responsables convenus.

**Utilisation :** importer les cas dans Jira/Xray, Squash TM ou un autre outil si disponible, ou utiliser directement ce fichier comme base de suivi et d’automatisation. Le document prépare la validation; seuls les résultats réels établissent le comportement testé.

