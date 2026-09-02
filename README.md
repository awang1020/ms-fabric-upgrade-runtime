# Migration des Runtimes Microsoft Fabric

Notebook Microsoft Fabric pour **inventorier l'ensemble d'un tenant**, mesurer l'impact d'un changement de Runtime Spark sur les notebooks et Spark Job Definitions, identifier les équipes à prévenir, puis exécuter la migration des Workspaces et des Environments via les API REST publiques Fabric.

> Le mode par défaut est un **plan sans écriture**. Aucun changement n'est envoyé tant que `DRY_RUN = False` **et** la phrase `CONFIRMATION` exacte ne sont pas renseignés.

## Notebook

[Ouvrir `Notebook_runtime_upgrade.ipynb`](./Notebook_runtime_upgrade.ipynb)

| Section | Contenu |
|---|---|
| **0** | Paramètres, garde-fous, pagination, gestion du throttling, suivi des opérations longues |
| **1.a** | Inventaire tenant-wide des Workspaces et de leur Runtime par défaut |
| **1.b** | Environment items : Runtime publié, Runtime en staging, publication en attente |
| **1.c** | Notebooks et Spark Job Definitions impactés, avec résolution du Runtime effectif |
| **1.d** | Admins et Membres à prévenir, table de notification |
| **2** | Migration des Environments puis des Workspaces, vérification après changement |

## Modèle de permissions — à lire avant d'exécuter

C'est le point le plus souvent mal compris. **Le rôle Fabric Administrator ne donne pas accès au contenu des Workspaces.** Les deux familles d'API n'ont pas la même portée :

| API | Portée | Permission requise |
|---|---|---|
| `GET /v1/admin/workspaces` | **Tenant** | Fabric Administrator, `Tenant.Read.All` |
| `GET /v1/admin/items?type=Notebook` | **Tenant** | Fabric Administrator, `Tenant.Read.All` |
| `GET /v1/admin/workspaces/{id}/users` | **Tenant** | Fabric Administrator, `Tenant.Read.All` |
| `GET /v1/workspaces/{id}/spark/settings` | **Workspace** | rôle *Viewer* minimum |
| `PATCH /v1/workspaces/{id}/spark/settings` | **Workspace** | rôle *Admin* |
| `GET`/`PATCH` `.../environments/{id}/staging/sparkcompute` | **Item** | lecture / écriture sur l'Environment |

Conséquence pratique : la **découverte** est tenant-wide, la **lecture du Runtime et la migration ne le sont pas**.

`fabric.list_workspaces()` s'appuie sur la portée utilisateur et ne retourne que les Workspaces où vous détenez un rôle. Le notebook utilise donc les Admin APIs pour l'inventaire, et **marque explicitement chaque Workspace illisible** (`AccessStatus = Forbidden`) plutôt que de le masquer silencieusement.

### Couvrir l'écriture sur tout le tenant

Trois options, du plus automatisé au plus manuel :

1. **Élévation just-in-time** — `AUTO_ELEVATE = True`. Le notebook s'attribue le rôle Workspace Admin, migre, puis le retire. Nécessite `sempy.fabric.admin` et le paramètre de tenant *Service principals can access admin APIs used for updates* si vous utilisez un SPN.
2. **Groupe de sécurité** — ajouter un groupe « Plateforme Data » comme Admin sur les Workspaces concernés. Préférable en gouvernance durable.
3. **Portail d'administration** — *Workspaces* → *Access*, pour un traitement ponctuel.

## Prérequis

| Catégorie | Exigence |
|---|---|
| Exécution | Notebook exécuté dans Microsoft Fabric, `sempy` disponible (préinstallé) |
| Inventaire tenant | Rôle **Fabric Administrator**, scope `Tenant.Read.All` |
| Lecture du Runtime | Rôle *Viewer* minimum sur chaque Workspace à inspecter |
| Migration | Rôle *Admin* sur chaque Workspace, droits d'écriture sur les Environments |
| Élévation | `sempy.fabric.admin` disponible, `ELEVATE_PRINCIPAL` renseigné |
| Validation | Tests des bibliothèques Python, wheels et JARs sur le Runtime cible |

## Paramètres

```python
TARGET_RUNTIME  = "1.3"        # ou "2.0"
SOURCE_RUNTIMES = {"1.2"}      # runtimes considérés comme à migrer

USE_ADMIN_API    = True        # inventaire tenant-wide
INCLUDE_PERSONAL = False       # inclure les "My workspaces"

DRY_RUN           = True       # aucun appel d'écriture
CONFIRMATION      = ""         # doit valoir exactement f"UPGRADE TO {TARGET_RUNTIME}"
TARGET_WORKSPACES = []         # [] = tous les workspaces détectés en SOURCE_RUNTIMES

AUTO_ELEVATE           = False
ELEVATE_PRINCIPAL      = None      # UPN (User) ou objectId (App / Group)
ELEVATE_PRINCIPAL_TYPE = "User"
REVOKE_AFTER_MIGRATION = True

RESOLVE_CONTACTS    = True     # équipes à prévenir
DEEP_SCAN_NOTEBOOKS = False    # Environment réellement attaché à chaque notebook

MAX_ADMIN_CALLS = 180          # garde-fou sous la limite de 200 appels/heure
```

## Utilisation

1. Importez le notebook dans un Workspace Fabric de développement.
2. Exécutez la section **0**, puis les sections **1.a** à **1.d**.
3. Analysez trois sorties clés :
   - `workspace_runtime_df` — répartition des Runtimes et **angles morts** d'accès ;
   - `impacted_items_df` — notebooks et Spark Job Definitions concernés ;
   - `notification_df` — destinataires par Workspace, et Workspaces orphelins.
4. Prévenez les équipes **avant** toute écriture.
5. Restreignez le périmètre à un Workspace de développement :

```python
TARGET_WORKSPACES = ["Fabric-DEV"]
```

6. Activez les deux garde-fous :

```python
DRY_RUN      = False
CONFIRMATION = "UPGRADE TO 1.3"   # doit correspondre à TARGET_RUNTIME
```

7. Vérifiez `operations_df` : seules les lignes `Succeeded` valent succès. Testez les traitements représentatifs, puis élargissez à UAT et PROD une vague à la fois.

### Résolution du Runtime effectif

Un Environment attaché à un item **écrase** le Runtime par défaut du Workspace. Le notebook applique cette priorité :

| Impact | Signification |
|---|---|
| `Impacte` | Le Runtime effectif est dans `SOURCE_RUNTIMES` : l'item changera de Runtime |
| `A verifier` | Workspace à jour, mais contenant un Environment encore en Runtime source |
| `Indetermine` | Workspace non lisible : impact inconnu tant que l'accès n'est pas obtenu |
| `Non impacte` | Déjà sur un Runtime hors périmètre |

L'association notebook → Environment vit dans la **définition** du notebook et non dans la liste des items : elle n'est résolue que si `DEEP_SCAN_NOTEBOOKS = True`, au prix d'un appel supplémentaire par notebook.

## Choisir la cible

Runtime 1.2 est **hors support depuis le 31 mars 2026**. Il ne reçoit plus de correctifs, de mises à jour de sécurité ni de remédiations de vulnérabilités, et la désactivation progressive de ses jobs a été annoncée.

| Composant | Runtime 1.2 | Runtime 1.3 | Runtime 2.0 |
|---|---|---|---|
| Statut | Hors support | GA jusqu'au 30/09/2026, puis LTS | GA |
| Fin de support | 31 mars 2026 | mars 2027 | 31 août 2028 |
| Apache Spark | 3.4.1 | 3.5 | 4.1 |
| Java | 11 | 11 | 21 |
| Scala | 2.12.17 | 2.12.17 | 2.13 |
| Python | 3.10 | 3.11 | 3.13 |
| Delta Lake | 2.4 | 3.2 | 4.2 |

- **Runtime 1.3** — palier **transitoire**. Le saut depuis 1.2 est faible (Spark 3.4 → 3.5, Python 3.10 → 3.11), donc peu risqué, mais le support s'arrête en **mars 2027** : une seconde migration est à prévoir.
- **Runtime 2.0** — cible durable, supportée jusqu'en août 2028. Le saut est plus large (Spark 4.1, Python 3.13, Java 21, Scala 2.13, Delta Lake 4.2) et exige une validation applicative sérieuse.

Runtime 2.0 reste opt-in : Microsoft prévoit d'en faire la valeur par défaut des nouveaux Workspaces et Environments à partir de fin septembre 2026.

## Garde-fous intégrés

- **Double confirmation** avant toute écriture.
- **Aucune erreur silencieuse** : chaque Workspace et chaque Environment reçoit un statut explicite, y compris `Forbidden`, `NotFound` et `Exception`.
- **Un `202 Accepted` ne vaut pas succès** : chaque publication d'Environment est suivie jusqu'à son état terminal via `/v1/operations/{id}`, puis la configuration publiée est relue.
- **Payload minimal** : seul `runtimeVersion` est envoyé. Le PATCH fusionne, ce qui préserve le pool et les propriétés Spark existantes.
- **Snapshots** de la configuration avant chaque modification, conservés dans `snapshots_df`.
- **Throttling** : respect de `Retry-After` sur HTTP 429 et garde-fou `MAX_ADMIN_CALLS`.

## Limites connues

- Les Admin APIs sont limitées à **200 requêtes par heure**. `RESOLVE_CONTACTS` consomme un appel par Workspace impacté, d'où la restriction aux seuls Workspaces concernés.
- `GET /v1/admin/items` est en **préversion**.
- Le rollback n'est pas une stratégie durable : Runtime 1.2 étant hors support, y revenir ne fait que déplacer le problème.
- La résolution notebook → Environment est *best effort* : elle lit la définition du notebook et nécessite un accès au Workspace.

## Ce que le notebook ne valide pas

Changer de Runtime ne prouve pas la compatibilité applicative. Ce notebook ne remplace pas les tests de non-régression sur :

- les bibliothèques Python et les wheels compilées ;
- les JARs et leurs versions Scala, Spark ou Java ;
- les changements de comportement Spark SQL ;
- les fonctionnalités Delta Lake 4.x, notamment sur des tables consommées par plusieurs workloads Fabric ;
- les performances et la consommation de Capacity Units.

La règle terrain : DEV, puis UAT, puis PROD. Une vague à la fois, avec une validation fonctionnelle entre chaque vague.

## Confidentialité

Les sorties de ce notebook contiennent des **noms de Workspaces, des identifiants et des adresses de messagerie**. Videz les sorties avant tout partage ou commit :

```
Notebook → Edit → Clear all outputs
```

Sur un notebook Fabric, pensez également à la clé `metadata.synapse_widget`, qui met en cache les lignes des DataFrames affichés.

## Documentation officielle

- [Fabric Runtime 1.2 (EOS)](https://learn.microsoft.com/fabric/data-engineering/runtime-1-2)
- [Fabric Runtime 1.3](https://learn.microsoft.com/fabric/data-engineering/runtime-1-3)
- [Fabric Runtime 2.0](https://learn.microsoft.com/fabric/data-engineering/runtime-2-0)
- [Apache Spark runtimes in Fabric](https://learn.microsoft.com/fabric/data-engineering/runtime)
- [Lifecycle of Apache Spark runtimes](https://learn.microsoft.com/fabric/data-engineering/lifecycle)
- [Manage Fabric Environments through public APIs](https://learn.microsoft.com/fabric/data-engineering/environment-public-api)
- [Workspaces - List Workspaces (Admin)](https://learn.microsoft.com/rest/api/fabric/admin/workspaces/list-workspaces)
- [Items - List Items (Admin)](https://learn.microsoft.com/rest/api/fabric/admin/items/list-items)
- [Workspace Settings - Update Spark Settings](https://learn.microsoft.com/rest/api/fabric/spark/workspace-settings/update-spark-settings)
- [Long running operations](https://learn.microsoft.com/rest/api/fabric/articles/long-running-operation)
