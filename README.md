# Migration des Runtimes Microsoft Fabric

Notebook Fabric pour **inventorier un tenant entier**, mesurer l'impact d'un changement de Runtime Spark, prévenir les équipes concernées, migrer, puis produire un compte rendu.

Runtime 1.2 est **hors support depuis le 31 mars 2026**. Ce notebook automatise le passage vers un Runtime supporté, y compris sur les Workspaces où vous n'avez aucun rôle.

> Par défaut le notebook ne modifie rien. Deux garde-fous cumulatifs sont nécessaires pour écrire.

## Notebook

[Ouvrir `Fabric_Runtime_Migration.ipynb`](./Fabric_Runtime_Migration.ipynb)

| Section | Contenu | Écrit ? |
|---|---|---|
| **0** | Paramètres, couche HTTP, pagination, throttling, suivi des opérations longues | Non |
| **1** | Inventaire tenant-wide, capacités et éligibilité, Runtime par défaut | Non |
| **2** | Environments, Notebooks, SJD et répartition par Runtime | Non |
| **3** | Segmentation en vagues A et B, classement des Workspaces inaccessibles | Non |
| **4** | Admins et Membres à prévenir, liste de diffusion | Non |
| **5** | Migration en 4 phases, élévation just-in-time | **Oui** |
| **6** | Compte rendu de synthèse | Non |

## Modèle de permissions

Le rôle **Fabric Administrator ne donne pas accès au contenu des Workspaces**. Les API sont asymétriques : la découverte est tenant-wide, la lecture du Runtime et la migration ne le sont pas.

| API | Portée | Permission |
|---|---|---|
| `GET /v1/admin/workspaces` | **Tenant** | Fabric Administrator, `Tenant.Read.All` |
| `GET /v1/admin/items` | **Tenant** | Fabric Administrator, `Tenant.Read.All` |
| `GET /v1/admin/capacities` | **Tenant** | Fabric Administrator, `Tenant.Read.All` |
| `GET /v1/admin/workspaces/{id}/users` | **Tenant** | Fabric Administrator, `Tenant.Read.All` |
| `GET /v1/workspaces/{id}/spark/settings` | **Workspace** | rôle *Viewer* minimum |
| `PATCH /v1/workspaces/{id}/spark/settings` | **Workspace** | rôle *Admin* |
| `.../environments/{id}/staging/sparkcompute` | **Item** | écriture sur l'Environment |

Conséquence pratique : **l'analyse d'impact fonctionne sur 100 % du tenant sans aucun rôle Workspace.** Le décompte des Notebooks, la capacité, le SKU et les Admins viennent tous des Admin APIs. Seule la valeur du Runtime exige un rôle.

`fabric.list_workspaces()` s'appuie sur la portée utilisateur : il ne retourne que vos Workspaces. Le notebook utilise les Admin APIs à la place.

## Prédire l'éligibilité sans accès

Le `capacityId` est renvoyé pour **tous** les Workspaces, y compris ceux en `Forbidden`. Croisé avec le SKU de la capacité, il indique si un Workspace peut exécuter Spark — donc si demander un accès en vaut la peine.

| Famille de SKU | Spark | Élévation utile ? |
|---|---|---|
| `F2` … `F8192`, `FT1`, `FTL64` | Oui | Oui |
| `P1` … `P5` | Oui | Oui |
| `Trial` | Oui | Oui |
| `A1` … `A8` | **Non** | Non — Power BI uniquement |
| `EM1` … `EM3` | **Non** | Non |
| Aucune capacité (Pro, PPU) | **Non** | Non |

La classification se fait par **préfixe** et non par famille extraite en regex : les SKU Fabric déclinent de nombreux suffixes, et `FTL64` serait sinon lu comme une famille `FTL` inconnue.

Un SKU non reconnu produit `CapacitySkuUnknown`, une catégorie distincte de « incapable ». L'élévation y est **tentée** : mieux vaut un privilège inutile qu'un Workspace laissé sur un Runtime non supporté.

## Prérequis

| Catégorie | Exigence |
|---|---|
| Exécution | Notebook exécuté dans Microsoft Fabric, `sempy` disponible (préinstallé) |
| Inventaire | Rôle **Fabric Administrator**, scope `Tenant.Read.All` |
| Lecture du Runtime | Rôle *Viewer* minimum par Workspace |
| Migration | Rôle *Admin* par Workspace, écriture sur les Environments |
| Élévation JIT | `sempy.fabric.admin` disponible, `ELEVATE_PRINCIPAL` renseigné |

## Paramètres

```python
# --- Cible
TARGET_RUNTIME  = "2.0"
SOURCE_RUNTIMES = {"1.1", "1.2", "1.3"}

# --- Périmètre
INCLUDE_PERSONAL  = False   # inclure les "My workspaces"
TARGET_WORKSPACES = []      # [] = tout le tenant, sinon une liste de noms
MIGRATION_WAVE    = "A"     # "A" | "B" | "ALL"

# --- Garde-fous
DRY_RUN         = True      # True = plan seul
CONFIRMATION    = ""        # doit valoir f"UPGRADE TO {TARGET_RUNTIME}"
BLOCK_DOWNGRADE = True      # refuse de rétrograder un Runtime plus récent

# --- Élévation just-in-time
AUTO_ELEVATE           = False
ELEVATE_PRINCIPAL      = None     # UPN pour un User, objectId pour App / Group
ELEVATE_PRINCIPAL_TYPE = "User"   # "User" | "App" | "Group"
REVOKE_AFTER_MIGRATION = True
ELEVATION_WAIT_S       = 30

# --- Options
RESOLVE_CONTACTS    = True
NOTIFY_ROLES        = ["Admin", "Member"]
DEEP_SCAN_NOTEBOOKS = False
MAX_ADMIN_CALLS     = 180
```

La section 0 **échoue immédiatement** si `ELEVATE_PRINCIPAL_TYPE` n'est pas `User`, `App` ou `Group`, ou si `AUTO_ELEVATE = True` sans `ELEVATE_PRINCIPAL`. L'erreur la plus fréquente est d'intervertir les deux paramètres.

---

# Scénarios d'utilisation

## Scénario 1 — Assessment complet, sans rien modifier

```python
DRY_RUN        = True
CONFIRMATION   = ""
AUTO_ELEVATE   = False
MIGRATION_WAVE = "A"
```

Exécutez les sections **0 → 6**. Sorties utiles :

| Objet | Contenu |
|---|---|
| `profile_df` | Un Workspace par ligne : Runtime, capacité, SKU, éligibilité, vague, accès |
| `spark_summary_df` | Objets Spark par type et par état vis-à-vis de la cible |
| `wave_a_df` | Migrables directement — aucun objet Spark |
| `wave_b_df` | À faire tester — contiennent du Spark |
| `role_worth_it_df` | Rôle manquant sur une capacité éligible |
| `notification_df` | Liste de diffusion pour la vague B |
| `summary_df` | Tableau de synthèse à archiver |

## Scénario 2 — Migrer les Workspaces sans charge Spark

Ces Workspaces ne contiennent ni Notebook, ni Spark Job Definition, ni Environment — typiquement des rapports Power BI. Changer leur Runtime est une opération de configuration sans effet fonctionnel.

Commencez par **un seul** Workspace :

```python
TARGET_WORKSPACES = ["nom-du-pilote"]
MIGRATION_WAVE    = "A"
DRY_RUN           = False
CONFIRMATION      = "UPGRADE TO 2.0"
```

Puis élargissez :

```python
TARGET_WORKSPACES = []
MIGRATION_WAVE    = "A"
RESOLVE_CONTACTS  = False    # la liste de diffusion est déjà établie
```

> `TARGET_WORKSPACES` filtre **dès la section 1**. Après l'avoir modifié, réexécutez à partir de la section 1, pas seulement la section 5.

## Scénario 3 — Migrer les Workspaces avec charge Spark

Uniquement après retour des équipes prévenues via `notification_df`.

```python
TARGET_WORKSPACES = ["equipe-1", "equipe-2"]
MIGRATION_WAVE    = "B"
DRY_RUN           = False
CONFIRMATION      = "UPGRADE TO 2.0"
```

C'est ici que les Environments sont republiés. **Attendez-vous à des échecs** : Microsoft indique que les bibliothèques sont migrées automatiquement mais que les **JAR** ont une probabilité significative de ne plus fonctionner, et que la publication échoue en cas de conflit. Le notebook les remonte dans `failures` sans s'interrompre.

## Scénario 4 — Migrer des Workspaces où vous n'avez aucun rôle

```python
MIGRATION_WAVE         = "ALL"
TARGET_WORKSPACES      = ["un-seul-workspace"]
DRY_RUN                = False
CONFIRMATION           = "UPGRADE TO 2.0"
AUTO_ELEVATE           = True
ELEVATE_PRINCIPAL      = "prenom.nom@contoso.com"
ELEVATE_PRINCIPAL_TYPE = "User"
REVOKE_AFTER_MIGRATION = True
```

Déroulé de la section 5 :

| Phase | Action |
|---|---|
| 0 | Attribution du rôle *Admin*, attente de propagation, **re-lecture** du Runtime **et des Environments** devenus visibles |
| 1 | Environments : mise à jour puis publication suivie jusqu'à l'état terminal |
| 2 | Workspaces : mise à jour du Runtime, puis relecture de contrôle |
| 3 | Retrait des rôles accordés, dans un `finally` |

Points d'attention :

- **L'élévation ne cible que les capacités éligibles.** Un `Forbidden` sur un SKU A/EM, sans capacité ou sur capacité en pause n'est jamais élevé : le rôle n'y changerait rien.
- **`MIGRATION_WAVE` filtre aussi l'élévation.** Un Workspace bloqué contenant des Notebooks est classé vague B : avec `MIGRATION_WAVE = "A"`, il ne sera pas élevé. Utilisez `"ALL"` en cas de doute.
- **Les Environments sont relus après élévation.** Sans cela, le Runtime du Workspace serait migré mais son Environment resterait en arrière — et comme l'Environment écrase le Runtime du Workspace, la migration serait silencieusement incomplète.
- **Seuls les rôles accordés par le notebook sont retirés.** Un accès légitime préexistant n'est jamais révoqué.
- **La propagation n'est pas garantie en 30 s.** Si des `ReadAfterElevation` échouent, montez `ELEVATION_WAIT_S` à 60 ou 120, ou redémarrez la session pour forcer un nouveau jeton.
- **Un crash du kernel laisse les rôles en place.** Gardez les lots courts et contrôlez le portail après une interruption anormale.
- Chaque attribution et retrait est **journalisé dans l'audit Fabric**.

Vérification après exécution :

```python
operations_df[operations_df["Operation"].isin(["Elevate", "Revoke"])]
```

Le nombre d'`Elevate` réussis doit égaler celui de `Revoke` réussis.

## Scénario 5 — Workspaces bloqués par la capacité

Ces cas **ne se résolvent pas par un rôle**. Le notebook les classe séparément et les exclut de l'élévation.

| Catégorie | `errorCode` observé | Action |
|---|---|---|
| `CapacityNotActive` | `CapacityNotActive` | Reprendre la capacité dans le portail Azure, puis relancer |
| `WorkspaceHasNoCapacityAssigned` | `WorkspaceHasNoCapacityAssigned` | Hors périmètre **tant qu'aucune capacité n'est assignée** |
| `SkuWithoutSpark` | — | Hors périmètre : A, EM ou PPU ne portent que Power BI |
| `CapacitySkuUnknown` | — | Élévation tentée, vérifier la capacité manuellement |

Un Workspace sans capacité Fabric n'a pas de Runtime Spark. Présentez-le comme « non applicable en l'état » et non comme « terminé » : si une capacité lui est assignée plus tard, il redevient concerné.

## Scénario 6 — Revenir en arrière

`snapshots_df` contient la configuration de chaque objet **avant** modification.

```python
snapshots_df.to_csv("/lakehouse/default/Files/snapshots.csv", index=False)
```

Exportez-le avant de fermer la session : il n'est pas persisté.

> Le retour arrière n'est pas une stratégie durable. Runtime 1.2 étant hors support et ses jobs progressivement désactivés, y revenir ne fait que différer le problème.

---

# Comprendre les sorties

## Statuts d'accès

Le notebook classe les échecs sur l'`errorCode` Fabric, plus stable que le code HTTP.

| `AccessStatus` | `errorCode` | Cause | Action |
|---|---|---|---|
| `Ok` | — | Runtime lisible | Aucune |
| `Forbidden` | `InsufficientPrivileges` | Aucun rôle sur le Workspace | **Seul cas** que `AUTO_ELEVATE` résout |
| `NoFabricCapacity` | `WorkspaceHasNoCapacityAssigned` | Aucune capacité assignée | Hors périmètre en l'état |
| `CapacityUnavailable` | `CapacityNotActive` | Capacité en pause | Reprendre puis relancer |
| `Throttled` | — | Quota dépassé | Attendre puis relancer |

Aucun Workspace n'est écarté silencieusement : chacun porte sa cause et son action.

## Répartition de la charge Spark

La section 2 croise chaque objet Spark avec son Runtime effectif :

```
Etat                Deja sur la cible  A migrer  Autre runtime  Inconnu  Total
Environment                         1         1              0        0      2
Notebook                            1         2              0        1      4
SparkJobDefinition                  0         1              0        0      1
TOTAL                               2         4              0        1      7
```

Quatre états, et non deux :

- **Autre runtime** — hors `SOURCE_RUNTIMES` et hors cible : l'objet ne sera pas touché.
- **Inconnu** — objets des Workspaces illisibles. On sait qu'ils existent, pas sur quel Runtime.

Les noyer dans « à migrer » fausserait le chiffrage.

## Segmentation en vagues

| Vague | Définition | Risque |
|---|---|---|
| **A** | Aucun Notebook, SJD ni Environment | Négligeable |
| **B** | Au moins un objet Spark | Réel, validation requise |

Un Lakehouse sans Notebook reste en vague A : le Runtime appliqué est celui du Workspace **qui exécute** le code, pas celui qui héberge les données.

## Runtime effectif d'un item

Un Environment attaché **écrase** le Runtime par défaut du Workspace. Cette association vit dans la **définition** du Notebook, pas dans la liste des items : la résoudre exige un appel par Notebook, activé par `DEEP_SCAN_NOTEBOOKS`.

Le scan est restreint aux Notebooks dont le Workspace **contient au moins un Environment** — ailleurs, il ne peut rien révéler.

Il n'est **pas nécessaire à la migration** : Workspaces et Environments sont migrés séparément. Il n'affine que le reporting par item. Deux cas justifient de l'activer :

- vous présentez au client un décompte Notebook par Notebook ;
- vous soupçonnez un Notebook attaché à un Environment situé dans un **autre** Workspace, potentiellement hors périmètre.

## Lire le compte rendu

Le rapport distingue **objets** et **opérations** :

| Objet | Opérations |
|---|---|
| Workspace | `UpdateRuntime` |
| Environment | `UpdateRuntime` puis `Publish` |

30 Workspaces et 2 Environments produisent 34 opérations.

---

# Garde-fous

- **Double confirmation** : `DRY_RUN = False` **et** `CONFIRMATION` exacte. La chaîne contient le Runtime cible, ce qui bloque une migration vers une cible modifiée sans revalidation.
- **`BLOCK_DOWNGRADE`** : un Workspace sur un Runtime plus récent que la cible est exclu et signalé.
- **Élévation ciblée** : uniquement les `Forbidden` dont la capacité peut porter Spark.
- **Aucune erreur silencieuse** : chaque objet reçoit un statut explicite avec son `errorCode`.
- **Un `202 Accepted` ne vaut pas succès** : chaque publication est suivie via `/v1/operations/{id}`, puis relue.
- **Payload minimal** : seul `runtimeVersion` est envoyé, le `PATCH` fusionne et préserve pool et propriétés Spark.
- **Snapshots** avant chaque modification.
- **Throttling** : respect de `Retry-After` et garde-fou `MAX_ADMIN_CALLS`.

> Pensez à remettre `CONFIRMATION = ""` après une exécution : la valeur est persistée dans le notebook enregistré.

# Choisir la cible

| Composant | Runtime 1.2 | Runtime 1.3 | Runtime 2.0 |
|---|---|---|---|
| Statut | Hors support | GA jusqu'au 30/09/2026 | GA |
| Fin de support | 31 mars 2026 | mars 2027 | 31 août 2028 |
| Apache Spark | 3.4.1 | 3.5 | 4.1 |
| Java | 11 | 11 | 21 |
| Scala | 2.12.17 | 2.12.17 | 2.13 |
| Python | 3.10 | 3.11 | 3.13 |
| Delta Lake | 2.4 | 3.2 | 4.2 |

**Runtime 1.3** est un palier transitoire : saut faible depuis 1.2, mais support s'arrêtant en mars 2027. **Runtime 2.0** est la cible durable, au prix d'une validation applicative plus sérieuse.

Runtime 2.0 devient le défaut des nouveaux Workspaces et Environments fin septembre 2026.

# Limites connues

- Admin APIs limitées à **200 requêtes par heure**. `RESOLVE_CONTACTS` consomme un appel par Workspace de la vague B — passez-le à `False` sur les réexécutions.
- `GET /v1/admin/items` et `List Workspace Access Details` sont en **préversion**.
- Après passage en 2.0, certains Environments renvoient `LibraryManagementError: An upgrade to the base Spark Python environment has been detected`. Correctif : retirer toutes les bibliothèques, publier, les réajouter, publier à nouveau.

# Ce que le notebook ne valide pas

Changer de Runtime ne prouve pas la compatibilité applicative. Testez :

- les bibliothèques Python et les wheels compilées ;
- les JAR et leurs versions Scala, Java, Spark ;
- les changements de comportement Spark SQL ;
- les fonctionnalités Delta Lake 4.x sur des tables consommées par plusieurs workloads Fabric ;
- les performances et la consommation de Capacity Units.

La règle terrain : DEV, puis UAT, puis PROD. Une vague à la fois.

# Confidentialité

Les sorties contiennent des **noms de Workspaces, identifiants et adresses de messagerie**. Videz-les avant tout partage :

```
Notebook → Edit → Clear all outputs
```

Sur un notebook Fabric, pensez aussi à `metadata.synapse_widget`, qui met en cache les lignes des DataFrames affichés et **survit à l'effacement des sorties**.

# Documentation officielle

- [Fabric Runtime 1.2 (EOS)](https://learn.microsoft.com/fabric/data-engineering/runtime-1-2)
- [Fabric Runtime 1.3](https://learn.microsoft.com/fabric/data-engineering/runtime-1-3)
- [Fabric Runtime 2.0](https://learn.microsoft.com/fabric/data-engineering/runtime-2-0)
- [Lifecycle of Apache Spark runtimes](https://learn.microsoft.com/fabric/data-engineering/lifecycle)
- [Understand Microsoft Fabric licenses and capacity](https://learn.microsoft.com/fabric/enterprise/licenses)
- [Microsoft Fabric features by SKU](https://learn.microsoft.com/fabric/enterprise/fabric-features)
- [Manage Fabric Environments through public APIs](https://learn.microsoft.com/fabric/data-engineering/environment-public-api)
- [Workspaces - List Workspaces (Admin)](https://learn.microsoft.com/rest/api/fabric/admin/workspaces/list-workspaces)
- [Workspaces - List Workspace Access Details](https://learn.microsoft.com/rest/api/fabric/admin/workspaces/list-workspace-access-details)
- [Items - List Items (Admin)](https://learn.microsoft.com/rest/api/fabric/admin/items/list-items)
- [Workspace Settings - Update Spark Settings](https://learn.microsoft.com/rest/api/fabric/spark/workspace-settings/update-spark-settings)
- [Long running operations](https://learn.microsoft.com/rest/api/fabric/articles/long-running-operation)
