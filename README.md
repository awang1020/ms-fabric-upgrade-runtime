# Migrer le runtime Spark de Microsoft Fabric de 1.3 vers 2.0

Commencez par le [notebook optimisé](./Fabric_Runtime_Migration_Optimisee.ipynb) : il inventorie les espaces de travail **sans rien modifier**, puis permet de migrer un lot explicite et de vérifier le résultat. Il change le **runtime Spark par défaut de l’espace**, pas le runtime d’un environnement Fabric attaché à un notebook ou à un job.

[Démarrage rapide](#démarrage-rapide) · [Migration](#migrer-un-lot) · [Retour arrière](#revenir-en-arrière) · [Dépannage](#dépannage)

## Démarrage rapide

1. Sur GitHub, téléchargez le fichier [`Fabric_Runtime_Migration_Optimisee.ipynb`](./Fabric_Runtime_Migration_Optimisee.ipynb) via **Download raw file**. Dans un espace de test Fabric sur une capacité active, sélectionnez **Importer → Notebook → Depuis cet ordinateur**.
2. Ouvrez le notebook Python importé et exécutez toutes les cellules sans modifier les paramètres. Avec `MODE = 'INVENTAIRE'`, le notebook n’effectue que des lectures REST et ne crée aucun fichier.
3. Dans le tableau de sortie, relevez les GUID de la colonne `workspace_id` et choisissez **un seul espace hors production** pour le premier essai.

Par défaut, `CIBLES = []` inventorie les espaces visibles par l’identité d’exécution. Pour découvrir aussi les espaces du tenant auxquels cette identité n’a pas accès, configurez `INVENTAIRE_TENANT = True`. Cette option exige le rôle **Administrateur Fabric** et l’autorisation déléguée `Tenant.Read.All` ou `Tenant.ReadWrite.All`. Elle permet de découvrir un espace, pas de lire ses paramètres Spark sans rôle approprié.

| Résultat de l’inventaire | Que faire ? |
|---|---|
| `CANDIDAT`, vague A | Aucun type d’élément hors de la liste des rapports, modèles sémantiques, tableaux de bord et rapports paginés n’a été trouvé. Utilisez cet espace pour un premier essai, après vérification des dépendances en aval. |
| `VALIDATION_DEPENDANCES_REQUISE`, vague B | Un autre type d’élément est présent, par exemple un notebook, un environnement ou un Lakehouse. Testez les dépendances sur 2.0 avant d’ajouter le GUID à `VALIDES`. L’espace peut alors devenir `CANDIDAT` tout en restant en vague B. |
| `DEJA_EN_2_0` ou `HORS_RUNTIME_SOURCE` | Rien à migrer avec cette version : elle passe uniquement de 1.3 à 2.0. |
| `ENVIRONNEMENT_A_TRAITER` | Un environnement par défaut est attaché. Validez et traitez cet environnement séparément. |

`CANDIDAT` indique une éligibilité à la bascule, pas une garantie de compatibilité.

## Migrer un lot

L’identité qui exécute le notebook doit être **Admin de chaque espace cible** et pouvoir écrire dans le Lakehouse de journalisation. La capacité de chaque cible à modifier doit être confirmée active. Si vous n’êtes pas Admin de l’espace, faites exécuter le notebook par une identité approuvée qui dispose de ce rôle. Le rôle d’Administrateur Fabric du tenant ne le remplace pas, et le notebook optimisé n’accorde aucun droit.

Créez un dossier de journalisation protégé sous `Files` dans un Lakehouse. Dans la cellule de paramètres du notebook, remplacez les valeurs entre chevrons :

```python
MODE = 'MIGRATION'
CONFIRMATION = 'MIGRER 1.3 VERS 2.0'
TENANT_ATTENDU = '<guid-du-tenant>'
CIBLES = ['<guid-de-l-espace-pilote>']
VALIDES = []  # Pour la vague B, ajoutez ici chaque GUID validé.
EXCLUS = []   # GUID des espaces à ne jamais traiter.
JOURNAL = 'abfss://<guid-espace-journal>@onelake.dfs.fabric.microsoft.com/<guid-lakehouse>/Files/audit'
```

Le GUID de l’espace figure dans son URL Fabric, après `/groups/`. Le GUID du Lakehouse figure dans son URL, après `/lakehouses/`. Pour `TENANT_ATTENDU`, utilisez l’ID du tenant affiché dans Microsoft Entra.

Exécutez toutes les cellules. Le notebook vérifie le lot avant d’envoyer le premier changement, sauvegarde les paramètres initiaux, modifie uniquement le champ `runtimeVersion`, puis relit la configuration pour vérifier que les autres réglages n’ont pas changé. Un espace déjà en 2.0 n’est pas réécrit.

Après l’exécution, notez le GUID `Run` affiché et vérifiez les fichiers `*-avant.json` et `*-apres.json` dans le dossier OneLake de ce run. Démarrez une **nouvelle session Spark** et testez les notebooks, jobs, pipelines et consommateurs SQL/Power BI/Direct Lake concernés avant d’élargir le lot. Procédez de DEV à UAT puis PROD, une vague à la fois. Remettez ensuite le notebook en mode `INVENTAIRE` et effacez `CONFIRMATION`.

## Revenir en arrière

Utilisez le **même dossier `JOURNAL`**, le GUID du run de migration d’origine et uniquement les GUID des espaces à restaurer :

```python
MODE = 'RESTAURATION'
CONFIRMATION = 'RESTAURER 2.0 VERS 1.3'
TENANT_ATTENDU = '<guid-du-tenant>'
CIBLES = ['<guid-de-l-espace-à-restaurer>']
RUN_A_RESTAURER = '<guid-du-run-de-migration>'
```

La restauration exige un journal de migration vérifié pour chaque cible. Elle est bloquée si la configuration courante diffère de l’état enregistré après la migration. Elle reste possible tant que Fabric accepte le runtime 1.3. Elle ne restaure ni les artefacts ni les protocoles des tables Delta.

## Dépannage

| Symptôme | Action |
|---|---|
| `ACCES_SPARK_REQUIS`, `DELEGUER_A_UN_ADMIN` ou HTTP 403 | Vérifiez l’identité d’exécution et ses rôles. La lecture des paramètres Spark requiert au moins le rôle Lecteur ; la modification requiert Admin sur l’espace. |
| `CAPACITE_INACTIVE`, capacité non confirmée active ou `HORS_PERIMETRE` | Vérifiez l’affectation et l’état de la capacité. Le notebook ne réactive pas les capacités et exclut les espaces personnels et ceux sans capacité. |
| `NON_VISIBLE` ou `LECTURE_INCOMPLETE` | Vérifiez le GUID, le périmètre d’inventaire et le détail de l’erreur. N’interprétez pas une lecture incomplète comme une absence de dépendances. |
| HTTP 429 | Attendez avant de relancer. Le notebook réessaie certaines lectures, mais ne rejoue pas automatiquement une écriture dont le résultat est incertain. |
| Fichier `*-avant.json` sans `*-apres.json` | Relisez les paramètres Spark et examinez le journal **avant** toute relance : l’écriture peut avoir abouti malgré l’erreur. Si la vague s’est interrompue, les cibles précédentes peuvent déjà être migrées. |

Le notebook ne republie pas les environnements, ne modifie pas les données et ne teste pas la compatibilité des bibliothèques. Évitez les vagues concurrentes et les changements manuels sur les mêmes cibles pendant une exécution : l’API ne fournit pas de contrôle de concurrence documenté.

## Partager une copie du notebook

Avant de partager une copie exécutée, retirez les sorties et les métadonnées de visualisation mises en cache : elles peuvent contenir des noms et des identifiants d’espaces.

## Documentation Microsoft

- [Guide de migration de Fabric Runtime 1.3 vers 2.0](https://learn.microsoft.com/fabric/data-engineering/runtime-2-0-migration-guide) et [cycle de vie des runtimes](https://learn.microsoft.com/fabric/data-engineering/lifecycle)
- [Lire les paramètres Spark](https://learn.microsoft.com/rest/api/fabric/spark/workspace-settings/get-spark-settings) et [les modifier](https://learn.microsoft.com/rest/api/fabric/spark/workspace-settings/update-spark-settings)
- [Inventorier les espaces du tenant](https://learn.microsoft.com/rest/api/fabric/admin/workspaces/list-workspaces) et [gérer les environnements Fabric](https://learn.microsoft.com/fabric/data-engineering/environment-public-api)
