# Diff à réaliser pour le switch

## Étape 1 :

Désactivation du cluster sur R1

```diff
diff --git a/values-r1.yaml b/values-r1.yaml
index 255c8ed..412af95 100644
--- a/values-r1.yaml
+++ b/values-r1.yaml
@@ -1,8 +1,8 @@
 cnpg:
-  enabled: true
-  mode: primary
+  enabled: false
+  mode: recovery
   backup:
-    enabled: true
+    enabled: false
```

## Étape 2 :

Allumage de R2 en mode recovery

> ⚠️ Attention les backups doivent être désactivés sur R2.

```diff
diff --git a/values-r2.yaml b/values-r2.yaml
index ae09424..4580c71 100644
--- a/values-r2.yaml
+++ b/values-r2.yaml
@@ -1,8 +1,11 @@
 cnpg:
-  enabled: false
+  enabled: true
   mode: recovery
   backup:
     enabled: false
     endpointURL: https://sdid-248f-n8n.s3obj.ecs.objstore.r2.pi2.minint.fr
   recovery:
     endpointURL: https://sdid-248f-n8n.s3obj.ecs.objstore.r2.pi2.minint.fr
+    extraArgs:
+      recoveryTarget:
+        targetTime: "2026-03-23 08:55:00+00" # spécifier la date pour le recovery
```

## Étape 3 :

Passage de R2 en mode primary et réactivation des backups.

```diff
diff --git a/values-r2.yaml b/values-r2.yaml
index 4580c71..b80ac29 100644
--- a/values-r2.yaml
+++ b/values-r2.yaml
@@ -1,11 +1,8 @@
 cnpg:
   enabled: true
-  mode: recovery
+  mode: primary
   backup:
-    enabled: false
+    enabled: true
     endpointURL: https://sdid-248f-n8n.s3obj.ecs.objstore.r2.pi2.minint.fr
   recovery:
     endpointURL: https://sdid-248f-n8n.s3obj.ecs.objstore.r2.pi2.minint.fr
-    extraArgs:
-      recoveryTarget:
-        targetTime: "2026-03-23 08:55:00+00" # spécifier la date pour le recovery
```
