# Projet 4 : auditer un environnement de données

Audit de la base de données décisionnelle (OLAP) de **SuperSmartMarket**, une chaîne française de supermarchés. Le but est de trouver pourquoi le chiffre d'affaires affiché dans Power BI change après coup.

## Contexte

SuperSmartMarket utilise des outils Microsoft, un entrepôt de données OLAP et Power BI pour suivre son chiffre d'affaires. Le service commercial a remarqué que l'historique du CA n'était pas stable. Par exemple :

- le **14 août 2024**, le CA du jour affiché était de **275 186,59 €** ;
- le **16 août**, après le jour férié du 15, le CA du 14 août affiché était passé à **284 243,88 €**.

La base n'avait jamais été documentée. La mission comporte trois volets :

1. documenter la base de données (dictionnaire des données et schéma relationnel) ;
2. trouver la source de l'écart ;
3. proposer un plan d'action pour corriger le problème à la source.

## Contenu du dossier

| Élément | Description |
| --- | --- |
| `df_Vente_detail.csv` | Table de faits : une ligne par article vendu (client, employé, EAN, date d'achat, ticket) |
| `df_Produits.csv` | Dimension produits : EAN, catégorie, rayon, libellé, prix |
| `df_Clients.csv` | Dimension clients : identifiant, date d'inscription |
| `df_Employe.csv` | Dimension employés : identifiant, nom, date d'arrivée, adresse e-mail |
| `df_Calendrier.csv` | Dimension calendrier : jour, mois, trimestre, année |
| `Logs.csv` | Journal des opérations sur la base (INSERT, UPDATE, DELETE) : 207 489 lignes |
| `Schéma+architecture+.jpg` | Schéma de l'architecture des données de l'entreprise |
| `Auditez_un_environnement_de_données_SIESSE_Thomas/` | Livrables du projet |
| `Auditez_un_environnement_de_données_SIESSE_Thomas.zip` | Archive des livrables, prête à être déposée |
| `SUD_Immo/`, `Course-implementez-BDD-SQL-master/`, `dataset_quiz_p1.xlsx` | Supports et exercices de cours (SQL), hors livrables |

Les CSV utilisent le point-virgule comme séparateur. Les dates sont au format numéro de série Excel : `45518` correspond au 14 août 2024.

### Livrables

- **`SIESSE_Thomas_1_PRESENTATION_042026.pptx`** : support de soutenance. Il présente la mission, l'architecture, le schéma relationnel, le dictionnaire des données, le prototype, les résultats SQL, l'analyse des logs et les recommandations.
- **`SIESSE_Thomas_2_audit_042026.pdf`** : rapport d'audit complet. Il contient le résumé exécutif, les constats, l'analyse, les recommandations, le plan d'action et un glossaire.

## Modèle de données

La base suit un **schéma en étoile**, adapté à l'analyse décisionnelle :

- la table de faits **Vente_detail** enregistre chaque transaction ;
- elle est reliée à 4 dimensions : **Produit**, **Clients**, **Employé** et **Calendrier**.

Ce modèle permet d'analyser le CA selon quatre axes : qui a acheté, quoi, quand, et quel employé a réalisé la vente.

## Démarche

1. Import des données dans **PostgreSQL** en local, sous forme de POC, pour ne pas toucher à la production.
2. Requêtes SQL de contrôle : CA total du 14 août, top 10 des clients, CA encaissé par employé.
3. Analyse des logs : 207 489 opérations, dont 206 905 INSERT, 582 UPDATE et 2 DELETE.

## Constat

Le dernier chargement des logs d'une journée a lieu **le lendemain**. La colonne `date` des logs contient la date d'enregistrement, pas la date de la vente. Les ventes du 14 août chargées le 15 apparaissent donc avec un jour de décalage dans Power BI :

| Logs enregistrés le | CA |
| --- | --- |
| 14 août (`45518`) | 275 186,59 € |
| 15 août (`45519`) | 9 057,29 € |
| **Total** | **284 243,88 €** |

Le total correspond exactement au CA affiché le 16 août pour la journée du 14.

```sql
SELECT SUM(p.prix) AS chiffre_affaires_total
FROM logs l
JOIN produit p ON p.ean = l.detail::NUMERIC
WHERE l.champs = 'EAN' AND l.date = '45518';
```

## Recommandations

| Action | Échéance | Responsable |
| --- | --- | --- |
| Enregistrer le dernier lot de logs à la fin de la journée (planification Power BI) | Immédiat | Équipe BI |
| Horodater les ventes et passer les dates au format `TIMESTAMP` ISO 8601 | Court terme | Développeur / data engineer |
| Mettre en place un CDC (Change Data Capture) pour alimenter le cube OLAP en quasi temps réel | Moyen terme | Data engineer |
| Développer une API et une interface métier (tableau de bord, KPI en temps réel) | Long terme | Développeur fullstack |

Le prototype proposé :
- migre les dates en `TIMESTAMP` ;
- sépare la date de transaction de la date d'import ;
- ajoute des contraintes de validation (`CHECK`, `NOT NULL`) et des index ;
- améliore le trigger de log pour qu'il horodate précisément chaque opération.

## Outils

PostgreSQL, SQL, Power BI, Excel.

## Auteur

Thomas SIESSE, avril 2026.
