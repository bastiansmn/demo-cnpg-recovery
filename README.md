# proto-cnpg-multi-region

Prototype de basculement d'un cluster CloudNativePG (CNPG) entre deux régions via Helm, en utilisant la réplication S3 comme mécanisme de continuité.

> ⚠️ Les valeurs Helm données dans l'exemple fonctionnent pour le chart https://github.com/this-is-tobi/helm-charts/tree/main/charts/cnpg-cluster.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Région R1 (primaire)          │  Région R2 (secondaire)        │
│                                │                                 │
│  ┌──────────────────────┐      │  ┌──────────────────────┐      │
│  │  CNPG Cluster        │      │  │  CNPG Cluster        │      │
│  │  mode: primary       │      │  │  mode: recovery      │      │
│  │  backup: enabled     │      │  │  enabled: false      │      │
│  └──────────┬───────────┘      │  └──────────┬───────────┘      │
│             │ WAL + basebackups│             │ restore           │
│             ▼                  │             ▼                   │
│  ┌──────────────────────┐      │  ┌──────────────────────┐      │
│  │  S3 R1               │─────▶│  │  S3 R2               │      │
│  │  (source)            │ répl │  │  (réplique)          │      │
│  └──────────────────────┘      │  └──────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

- **R1** : cluster CNPG en mode `primary`, qui écrit en continu ses WAL et ses basebackups vers le bucket S3 de R1.
- **S3 réplication** : le bucket S3 de R1 est configuré avec une réplication automatique vers R2. Les WAL et les basebackups sont ainsi disponibles sur R2 sans intervention.
- **R2** : cluster CNPG en mode `recovery`, **désactivé par défaut** (`enabled: false`). En cas de basculement, il est activé et restaure depuis le bucket S3 de R2.

## Structure des fichiers

| Fichier | Rôle |
|---|---|
| `values.yaml` | Valeurs communes (credentials S3, ressources, nom de la DB, secrets Vault…) |
| `values-r1.yaml` | Surcharge R1 : mode `primary`, backup activé, endpoint S3 R1 |
| `values-r2.yaml` | Surcharge R2 : mode `recovery`, cluster désactivé, endpoint S3 R2 |

## Déploiement initial (état nominal)

### R1 — cluster primaire actif

`values-r1.yaml` configure :
- `cnpg.enabled: true` → le cluster est créé
- `cnpg.mode: primary` → cluster en mode primaire
- `cnpg.backup.enabled: true` → backups et WAL vers S3 R1
- `cnpg.backup.endpointURL` / `cnpg.recovery.endpointURL` → S3 R1

### R2 — cluster secondaire désactivé

`values-r2.yaml` configure :
- `cnpg.enabled: false` → **aucune ressource CNPG créée**, le chart Helm ne déploie rien (condition sur `cnpg.enabled`)
- `cnpg.mode: recovery` → prêt à restaurer dès l'activation
- `cnpg.backup.enabled: false` → pas de backup depuis R2
- `cnpg.recovery.endpointURL` → S3 R2 (qui contient la réplique)

> Les secrets Vault sont tout de même synchronisés sur R2 via le `VaultStaticSecret`, même quand CNPG est désactivé, ce qui évite un délai au moment du basculement.

---

## Procédure de basculement R1 → R2

### Pré-requis

- La réplication S3 R1 → R2 est opérationnelle et à jour.
- Les secrets (S3, appuser, superuser) sont disponibles dans Vault sur R2.
- L'opérateur CNPG est installé sur le cluster Kubernetes de R2.

### Étape 1 — Arrêter le cluster R1

L'objectif est d'éviter toute écriture pendant la bascule pour garantir la cohérence des données.

**Option A — Via Helm (désactivation du cluster)**

Modifier `values-r1.yaml` :

```yaml
# values-r1.yaml
cnpg:
  enabled: false   # <-- était true
  mode: recovery # <-- pas obligatoire mais permet de minimiser les changements futurs en cas de rebascule.
  backup:
    enabled: false  # <-- était true
    endpointURL: https://sdid-248f-n8n.s3obj.ecs.objstore.r1.pi2.minint.fr
  recovery:
    endpointURL: https://sdid-248f-n8n.s3obj.ecs.objstore.r1.pi2.minint.fr
```

Cela suspend les pods sans supprimer le cluster, ce qui permet un retour en arrière plus rapide si nécessaire.

> Dans les deux cas, attendre que le dernier WAL soit bien répliqué sur S3 R2 avant de continuer.

### Étape 2 — Activer le cluster R2 en mode recovery

Modifier `values-r2.yaml` :

```yaml
# values-r2.yaml
cnpg:
  enabled: true    # <-- était false
  mode: recovery
  backup:
    enabled: false # <-- IMPORTANT : Désactiver les backup pendant le mode recovery
    endpointURL: https://sdid-248f-n8n.s3obj.ecs.objstore.r2.pi2.minint.fr
  recovery:
    endpointURL: https://sdid-248f-n8n.s3obj.ecs.objstore.r2.pi2.minint.fr
```

CNPG va créer un cluster en mode `recovery` : il va lire le dernier basebackup disponible sur le S3 R2 puis rejouer les WAL jusqu'au point le plus récent.

### Étape 3 — Suivre la restauration

```bash
# Surveiller l'état du cluster
kubectl get cluster cnpg-cluster -n <namespace-r2> -w

# Consulter les logs du pod primaire en cours de restauration
kubectl logs -l cnpg.io/instanceRole=primary -n <namespace-r2> -f
```

Le cluster passera par les phases suivantes :
1. `Setting up primary` — restauration du basebackup
2. `Applying WAL` — rejeu des WAL
3. `Cluster in healthy state` — cluster opérationnel

### Étape 4 — Repasser R2 en mode primaire avec backups

Une fois la restauration terminée, si R2 doit devenir le nouveau primaire durable (avec ses propres backups), modifier `values-r2.yaml` :

```yaml
# values-r2.yaml
cnpg:
  enabled: true
  mode: primary    # <-- était recovery
  backup:
    enabled: true  # <-- activer les backups depuis R2
    endpointURL: https://sdid-248f-n8n.s3obj.ecs.objstore.r2.pi2.minint.fr
  recovery:
    endpointURL: https://sdid-248f-n8n.s3obj.ecs.objstore.r2.pi2.minint.fr
```

> **Attention** : lors du passage de `recovery` à `primary`, CNPG supprime et recrée le cluster. C'est un comportement attendu — prévoir une courte indisponibilité.

---

## Récapitulatif des états par région

| État | R1 `values-r1.yaml` | R2 `values-r2.yaml` |
|---|---|---|
| **Nominal** | `enabled: true`, `mode: primary`, `backup: true` | `enabled: false`, `mode: recovery` |
| **Bascule en cours** | `enabled: false` | `enabled: true`, `mode: recovery` |
| **R2 nouveau primaire** | `enabled: false` | `enabled: true`, `mode: primary`, `backup: true` |

---

## Retour arrière (R2 → R1)

Si R1 redevient disponible et que l'on souhaite revenir dessus :

1. S'assurer que R1 est vide / arrêté (plus aucune écriture).
2. Si R2 a des backups actifs, vérifier que les WAL de R2 sont répliqués vers R1 (ou configurer la réplication dans l'autre sens).
3. Modifier `values-r1.yaml` pour activer R1 en mode `recovery` pointant vers son propre S3.
4. Une fois R1 restauré et opérationnel, l'arrêter et le repasser en `mode: primary` avec backup.
5. Désactiver R2.

---

## Notes

- Le chart utilise [`cnpg-cluster`](https://this-is-tobi.github.io/helm-charts) en version `1.6.0` comme dépendance.
- La condition `cnpg.enabled` dans `Chart.yaml` permet d'activer/désactiver le déploiement du cluster sans supprimer les autres ressources (ex: secrets Vault).
- Les secrets S3, appuser et superuser sont synchronisés via HashiCorp Vault (VaultStaticSecret) depuis le mount `democnpgrecovery`.
