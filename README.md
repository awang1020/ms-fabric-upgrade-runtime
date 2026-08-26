# Migration des Runtimes Microsoft Fabric

Notebook Microsoft Fabric pour inventorier les Workspaces et les Environments
encore en Runtime 1.2, puis préparer ou exécuter leur migration vers Runtime
1.3 ou Runtime 2.0 avec les API REST publiques Fabric.

> Le mode par défaut est un **dry run**. Aucun changement n'est envoyé tant
> qu'une liste explicite de Workspaces, le mode d'exécution et la phrase de
> confirmation ne sont pas renseignés.

## Notebook

[Ouvrir `Notebook_runtime_upgrade.ipynb`](./Notebook_runtime_upgrade.ipynb)

Le notebook couvre :

- l'inventaire du Runtime par défaut de chaque Workspace accessible ;
- l'inventaire du Runtime effectif des Environment items ;
- la sélection explicite des Workspaces à migrer ;
- la mise à jour du Runtime Workspace ;
- la mise à jour et la publication des Environments ;
- le suivi de la publication asynchrone ;
- la vérification du Runtime effectif après chaque changement ;
- le retry des réponses HTTP 429 et la pagination des Environments.

Il refuse de publier un Environment qui contient déjà des changements compute
non publiés. Ces changements doivent être revus manuellement avant la migration.

## Choisir la cible

Runtime 1.2 est hors support depuis le 31 mars 2026. Il ne reçoit plus de
correctifs, de mises à jour de sécurité ni de remédiations de vulnérabilités.

| Composant | Runtime 1.3 | Runtime 2.0 |
|---|---|---|
| Statut | GA, transition LTS annoncée | GA, version la plus récente |
| Apache Spark | 3.5.5 | 4.1 |
| Java | 11 | 21 |
| Scala | 2.12.17 | 2.13.16 |
| Python | 3.11 | 3.13 |
| Delta Lake | 3.2 | 4.2 |

- **Runtime 1.3** : utilisez-le comme palier si les changements Spark 4,
	Python 3.13, Java 21 ou Scala 2.13 bloquent une migration immédiate.
- **Runtime 2.0** : cible durable recommandée après validation des notebooks,
	Spark Job Definitions, bibliothèques et tables Delta partagées.

Runtime 2.0 est encore opt-in au moment de la publication de ce dépôt. Microsoft
prévoit d'en faire la valeur par défaut des nouveaux Workspaces et Environments
à partir de fin septembre 2026.

## Prérequis

- Exécuter le notebook dans Microsoft Fabric avec `sempy` disponible.
- Disposer d'un accès en lecture aux Workspaces à inventorier.
- Disposer du rôle Admin sur les Workspaces à modifier.
- Disposer des droits d'écriture sur les Environment items à publier.
- Tester les bibliothèques Python, wheels et JARs sur le Runtime cible.

## Utilisation

1. Importez le notebook dans un Workspace Fabric de développement.
2. Exécutez les cellules d'inventaire.
3. Dans la cellule de migration, choisissez la cible et gardez le dry run :

```python
TARGET_RUNTIME = "2.0"  # Ou "1.3"
TARGET_WORKSPACES = []
EXECUTE_MIGRATION = False
CONFIRMATION = ""
```

4. Analysez les lignes `Planned`, `Skipped` et `Failed`.
5. Ajoutez d'abord un seul Workspace de développement :

```python
TARGET_WORKSPACES = ["Fabric-DEV"]
```

6. Pour exécuter réellement la migration, activez les deux garde-fous :

```python
EXECUTE_MIGRATION = True
CONFIRMATION = "UPGRADE TO 2.0"  # Doit correspondre a TARGET_RUNTIME
```

7. Vérifiez les lignes `Verified`, puis testez les notebooks et Spark Job
	 Definitions représentatifs avant d'ajouter UAT et PROD à l'allowlist.

## Ce que le notebook ne valide pas

Le changement de Runtime ne prouve pas la compatibilité applicative. Le notebook
ne remplace pas les tests de non-régression sur :

- les bibliothèques Python et les wheels compilées ;
- les JARs et leurs versions Scala, Spark ou Java ;
- les changements de comportement Spark SQL ;
- les fonctionnalités Delta Lake 4.x, notamment sur des tables consommées par
	plusieurs workloads Fabric ;
- les performances et la consommation de Capacity Units de vos traitements.

La règle terrain : DEV, puis UAT, puis PROD. Une vague à la fois, avec une
validation fonctionnelle entre chaque vague.

## Documentation officielle

- [Fabric Runtime 1.2 (EOS)](https://learn.microsoft.com/fabric/data-engineering/runtime-1-2)
- [Apache Spark runtimes in Fabric](https://learn.microsoft.com/fabric/data-engineering/runtime)
- [Fabric Runtime 1.3](https://learn.microsoft.com/fabric/data-engineering/runtime-1-3)
- [Fabric Runtime 2.0](https://learn.microsoft.com/fabric/data-engineering/runtime-2-0)
- [Lifecycle of Apache Spark runtimes](https://learn.microsoft.com/fabric/data-engineering/lifecycle)
- [Manage Fabric Environments through public APIs](https://learn.microsoft.com/fabric/data-engineering/environment-public-api)
