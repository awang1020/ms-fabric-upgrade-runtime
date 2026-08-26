# ms-fabric-upgrade-runtime

Notebook guidance for upgrading Microsoft Fabric Spark workloads from Runtime 1.2 to Runtime 2.0.

## Notebook

Open [`notebooks/upgrade_fabric_runtime_1_2_to_2_0.ipynb`](notebooks/upgrade_fabric_runtime_1_2_to_2_0.ipynb) in Microsoft Fabric and run it as part of the upgrade process:

1. Run the baseline and dependency inventory cells while the notebook or environment still uses Runtime 1.2.
2. In Fabric, change the notebook or attached environment runtime to Runtime 2.0 and restart the Spark session.
3. Rerun the baseline, dependency inventory, and smoke-test cells to validate the upgrade.
4. Optionally set a temporary Lakehouse path in the notebook to validate Delta read/write behavior.

The notebook does not change the runtime by itself because Fabric runtime selection is managed in notebook or environment settings.
