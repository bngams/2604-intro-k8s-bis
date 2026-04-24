# Observabilité sur Kubernetes — Logs & Métriques avec Loki, Promtail, Prometheus et Grafana

Prérequis — on sait déployer et synchroniser des applications sur Kubernetes via HELM. On va maintenant introduire l'**observabilité** : collecter les logs et les métriques du cluster pour les visualiser dans Grafana.

> **Prérequis :** minikube démarré, Helm 3+ installé, ArgoCD optionnel.

---

# PHASE 0 — Pourquoi l'observabilité ?

## Le problème sans observabilité

Un cluster Kubernetes qui tourne, c'est bien. Mais sans observabilité :

- **Comment savoir qu'un pod crashe en boucle ?** `kubectl get pods` toutes les 5 minutes ?
- **Pourquoi cette requête a échoué il y a 2h ?** Les logs sont partis avec le pod redémarré
- **Le cluster est-il en train de saturer la mémoire ?** Aucune idée sans métriques
- **Quelle application consomme le plus de CPU ?** Mystère

## La réponse : la stack PLG + Prometheus

/!\ Promtail deprecated et remplacé par Alloy

```
Logs  ──► Promtail ──► Loki  ──► Grafana
                                    ▲
Métriques ──► Prometheus ───────────┘
```

| Outil | Rôle |
|---|---|
| **Promtail** | Agent de collecte de logs — tourne sur chaque nœud (DaemonSet), lit les logs des pods /!\ Promtail deprecated et remplacé par Alloy |
| **Loki** | Stockage et indexation des logs (comme Elasticsearch, mais léger) |
| **Prometheus** | Collecte et stockage des métriques (CPU, RAM, requêtes HTTP...) |
| **Grafana** | Interface de visualisation — dashboards, alertes, exploration |

---

# PHASE 1 — Préparer l'environnement

## Etape 1 — Vérifier les prérequis

```bash
# Minikube doit tourner
minikube status

# Helm doit être installé
helm version
# version.BuildInfo{Version:"v3.x.x"...}
```

Si Helm n'est pas installé :
```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Etape 2 — Créer le namespace d'observabilité

On regroupe tous les outils dans un namespace dédié `obs` :

```bash
kubectl create namespace obs
```

---

# PHASE 2 — Déployer Loki (stockage de logs)

**Loki** est le système de stockage de logs. Il ne collecte pas les logs lui-même — c'est le rôle de Promtail — il les reçoit, les indexe et les expose pour Grafana.

On utilise le mode **SingleBinary** : un seul pod qui fait tout (adapté au lab, pas à la production).

## Etape 3 — Ajouter le repo Helm Grafana

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

## Etape 4 — Créer le fichier de configuration Loki

Créer le fichier `obs/loki-values.yaml` (ou utiliser celui fourni) :

```yaml
loki:
  # Mode sans authentification — pas besoin de X-Scope-OrgID ni de tenant_id
  auth_enabled: false
  commonConfig:
    replication_factor: 1
  schemaConfig:
    configs:
      - from: "2024-04-01"
        store: tsdb
        object_store: s3
        schema: v13
        index:
          prefix: loki_index_
          period: 24h
  pattern_ingester:
      enabled: true
  limits_config:
    allow_structured_metadata: true
    volume_enabled: true

minio:
  enabled: true

# Désactiver les caches (inutiles en lab)
chunksCache:
  enabled: false
resultsCache:
  enabled: false

deploymentMode: SingleBinary
singleBinary:
  replicas: 1

# Désactiver tous les autres modes de déploiement
backend:
  replicas: 0
read:
  replicas: 0
write:
  replicas: 0
```

> **`auth_enabled: false`** — par défaut, Loki fonctionne en mode multi-tenant et exige un header `X-Scope-OrgID` sur chaque requête. En lab, on désactive cette authentification pour simplifier la configuration de Grafana et Promtail.

## Etape 5 — Installer Loki

```bash
helm install my-loki grafana/loki \
  -n obs \
  -f obs/loki-values.yaml \
  --version 6.53.0
```

Attendre que les pods soient `Running` (peut prendre 2-3 minutes) :

```bash
watch kubectl get pods -n obs
```

On doit voir les pods `my-loki-*` et `my-loki-minio-*` en état `Running`.

**URL interne de Loki** (utilisée par Promtail et Grafana) :
```
http://my-loki-gateway.obs.svc.cluster.local
```

---

# PHASE 3 — Déployer Grafana (visualisation)

**Grafana** est l'interface de visualisation. Il se connecte à Loki et Prometheus comme sources de données et permet de créer des dashboards.

## Etape 6 — Installer Grafana

```bash
helm repo add grafana-community https://grafana-community.github.io/helm-charts/
helm repo update
helm install my-grafana grafana-community/grafana -n obs
```

Après l'installation, Helm affiche la commande pour récupérer le mot de passe admin. Elle ressemble à :

```bash
kubectl get secret --namespace obs my-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

## Etape 7 — Accéder à l'UI Grafana

```bash
# Accès via port-forward
kubectl port-forward svc/my-grafana 3000:80 -n obs
```

Ouvrir [http://localhost:3000](http://localhost:3000) :
- **Username** : `admin`
- **Password** : le mot de passe récupéré ci-dessus

## Etape 8 — Configurer la datasource Loki

Dans Grafana : **Connections > Data Sources > Add data source > Loki**

| Champ | Valeur |
|---|---|
| **URL** | `http://my-loki-gateway.obs.svc.cluster.local` |

> Pas de header `X-Scope-OrgID` nécessaire — on a désactivé l'authentification Loki à l'étape 4.

Cliquer **Save & Test** → le message `Data source connected and labels found` doit apparaître.

---

# PHASE 4 — Déployer Promtail (collecte de logs)

**Promtail** est l'agent de collecte de logs. Il tourne en **DaemonSet** (un pod par nœud Kubernetes), lit les fichiers de logs des containers depuis `/var/log/pods/`, et les envoie à Loki.

## Etape 9 — Déployer Promtail

Le manifest `obs/promtail.yml` contient 5 ressources Kubernetes :
- `DaemonSet` — le pod Promtail sur chaque nœud
- `ConfigMap` — la configuration Promtail (URL Loki, règles de relabeling)
- `ClusterRole` — permissions pour lire pods/nodes/services
- `ServiceAccount` — compte de service
- `ClusterRoleBinding` — liaison rôle ↔ compte de service

```bash
kubectl apply -f obs/promtail.yml -n obs
```

Vérifier que le DaemonSet est actif :

```bash
kubectl get daemonset -n obs
# NAME                 DESIRED   CURRENT   READY
# promtail-daemonset   1         1         1
```

> **Pourquoi pas le chart Helm Promtail ?** Le manifest manuel permet de comprendre exactement ce qui est déployé. Pour la production, le chart Helm officiel (`grafana/promtail`) est recommandé.

## Etape 10 — Vérifier que les logs arrivent dans Loki

Dans Grafana : **Explore** (icône boussole) → sélectionner la datasource **Loki**

Cliquer **Log browser** → les labels `namespace`, `pod`, `container` devraient apparaître.

Exemple de requête LogQL pour voir les logs du namespace `argocd` :

```logql
{namespace="argocd"}
```

Ou filtrer par container :

```logql
{namespace="bgd", container="bgd"}
```

---

# PHASE 5 — Déployer Prometheus (métriques)

**Prometheus** collecte les métriques exposées par les pods Kubernetes (CPU, RAM, requêtes HTTP, etc.) et les stocke en time-series. Le chart Helm inclut automatiquement `kube-state-metrics` et `node-exporter`.

## Etape 11 — Ajouter le repo Helm Prometheus

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

## Etape 12 — Créer le fichier de configuration Prometheus

Créer le fichier `obs/prometheus-values.yaml` (ou utiliser celui fourni) :

```yaml
server:
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 100m
      memory: 256Mi
  retention: "7d"
  persistentVolume:
    enabled: true
    size: 8Gi

# Composants désactivés pour le lab
alertmanager:
  enabled: false
pushgateway:
  enabled: false

# Métriques Kubernetes — garder activés
kubeStateMetrics:
  enabled: true
nodeExporter:
  enabled: true
```

## Etape 13 — Installer Prometheus

```bash
helm install my-prometheus prometheus-community/prometheus \
  -n obs \
  -f obs/prometheus-values.yaml
```

Vérifier les pods :

```bash
kubectl get pods -n obs | grep prometheus
```

**URL interne de Prometheus** :
```
http://my-prometheus-server.obs.svc.cluster.local
```

## Etape 14 — Configurer la datasource Prometheus dans Grafana

Dans Grafana : **Connections > Data Sources > Add data source > Prometheus**

| Champ | Valeur |
|---|---|
| **URL** | `http://my-prometheus-server.obs.svc.cluster.local` |

Cliquer **Save & Test** → `Data source is working`.

---

# PHASE 6 — Importer les dashboards

## Etape 15 — Dashboard Loki : visualiser les logs Kubernetes

Dans Grafana : **Dashboards > New > Import**

| Champ | Valeur |
|---|---|
| **Dashboard ID** | `15141` |
| **Datasource** | Loki |

Ce dashboard ([Kubernetes Service Logs](https://grafana.com/grafana/dashboards/15141-kubernetes-service-logs/)) affiche les logs de tous les services Kubernetes, filtrables par namespace et pod.

## Etape 16 — Dashboard Prometheus : métriques du cluster

Dans Grafana : **Dashboards > New > Import**

| Champ | Valeur |
|---|---|
| **Dashboard ID** | `15661` |
| **Datasource** | Prometheus |

Ce dashboard ([K8s Dashboard EN](https://grafana.com/grafana/dashboards/15661-k8s-dashboard-en-20250125/)) affiche CPU, RAM, réseau, stockage par node et par pod.

---

# PHASE 7 — Explorer l'observabilité en action

## Etape 17 — Générer de l'activité et observer les logs

Si l'app `bgd` du TP6 est déployée, générer des requêtes :

```bash
# Lancer quelques requêtes sur bgd
kubectl port-forward svc/bgd 8080:8080 -n bgd &
for i in {1..20}; do curl -s http://localhost:8080 > /dev/null; done
```

Dans Grafana Explore avec Loki :

```logql
{namespace="bgd"} |= "GET"
```

## Etape 18 — Observer les métriques CPU/RAM

Dans le dashboard Prometheus importé, observer :
- **CPU usage** par pod
- **Memory usage** par namespace
- **Pod restarts** — indicateur clé d'instabilité

Pour simuler une charge :

```bash
# Scaler bgd à 3 replicas et observer l'impact sur les métriques
kubectl scale deploy/bgd --replicas=3 -n bgd
# Puis revenir à 1
kubectl scale deploy/bgd --replicas=1 -n bgd
```

---

## Recap — Stack d'observabilité déployée

```mermaid
graph TD
  Pods["Pods Kubernetes\n(tous namespaces)"] -->|logs /var/log/pods| PT["Promtail\n(DaemonSet)"]
  PT -->|push logs| Loki["Loki\n(stockage logs)"]
  Pods -->|expose /metrics| Prom["Prometheus\n(scrape métriques)"]
  Loki -->|datasource| Grafana["Grafana\n(dashboards)"]
  Prom -->|datasource| Grafana

  style Loki fill:#f59e0b,color:#000
  style Prom fill:#ef4444,color:#fff
  style Grafana fill:#1a1a2e,color:#fff
  style PT fill:#3b82f6,color:#fff
```

| Composant | Namespace | Accès |
|---|---|---|
| **Loki** | `obs` | `http://my-loki-gateway.obs.svc.cluster.local` |
| **Grafana** | `obs` | `kubectl port-forward svc/my-grafana 3000:80 -n obs` |
| **Prometheus** | `obs` | `http://my-prometheus-server.obs.svc.cluster.local` |
| **Promtail** | `obs` | DaemonSet — pas d'UI |

---

# PHASE 8 — Introduction à OpenTelemetry

## Le problème avec la stack actuelle

On a déployé **3 outils différents** pour l'observabilité :
- **Promtail** → collecte de logs
- **Prometheus** → collecte de métriques
- **(manquant)** → collecte de traces (traçage distribué)

Chaque outil a :
- Sa propre configuration
- Son propre format de données
- Son propre agent à déployer

**Problème** : si demain on veut changer de backend (ex: passer de Loki à Elasticsearch), il faut reconfigurer Promtail. Idem pour Prometheus.

## La réponse : OpenTelemetry (OTel)

**OpenTelemetry** est un standard CNCF qui unifie la collecte des **3 piliers de l'observabilité** :

```mermaid
graph TD
    OTel["OTel Collector<br/>(agent unique)"]
    OTel -->|Logs| L[Loki]
    OTel -->|Metrics| P[Prometheus]
    OTel -->|Traces| T[Tempo]
    L --> G[Grafana]
    P --> G
    T --> G

    style OTel fill:#f59e0b,color:#000
    style L fill:#f59e0b,color:#000
    style P fill:#ef4444,color:#fff
    style T fill:#8b5cf6,color:#fff
    style G fill:#1a1a2e,color:#fff
```

### Les 3 piliers de l'observabilité

| Pilier | Question à laquelle il répond | Exemple |
|---|---|---|
| **Logs** | Qu'est-ce qui s'est passé ? | `ERROR: Connection to database failed` |
| **Metrics** | Combien de fois ? Quelle tendance ? | CPU à 80%, 500 req/s |
| **Traces** | Quel est le chemin d'une requête ? | API → Service A → DB (120ms) |

> **Aujourd'hui**, on a logs + metrics. **OpenTelemetry** ajoute les traces et unifie les 3.

## Avantages d'OpenTelemetry

| Avantage | Explication |
|---|---|
| **Un seul agent** | OTel Collector remplace Promtail, node-exporter, etc. |
| **Vendor-agnostic** | Change de backend (Loki → Elasticsearch) sans toucher aux apps |
| **Standard industriel** | CNCF graduated project — supporté par tous les vendors |
| **Traces distribuées** | Suivre une requête à travers 10 microservices |
| **Corrélation** | Lier logs + metrics + traces d'une même requête |

## Architecture : avant / après

### Avant (stack actuelle)

```mermaid
graph LR
    Pods["Pods K8s"] -->|logs| Promtail
    Pods -->|metrics| Prometheus
    Promtail --> Loki
    Loki --> Grafana
    Prometheus --> Grafana

    style Promtail fill:#3b82f6,color:#fff
    style Loki fill:#f59e0b,color:#000
    style Prometheus fill:#ef4444,color:#fff
    style Grafana fill:#1a1a2e,color:#fff
```

### Après (avec OpenTelemetry)

```mermaid
graph LR
    Pods["Pods K8s"] -->|logs + metrics + traces| OTel["OTel Collector"]
    OTel --> Loki
    OTel --> Prometheus
    OTel --> Tempo
    Loki --> Grafana
    Prometheus --> Grafana
    Tempo --> Grafana

    style OTel fill:#f59e0b,color:#000
    style Loki fill:#f59e0b,color:#000
    style Prometheus fill:#ef4444,color:#fff
    style Tempo fill:#8b5cf6,color:#fff
    style Grafana fill:#1a1a2e,color:#fff
```

> **Note** : on peut utiliser OTel en **complément** (garder Prometheus + Loki) ou en **remplacement** complet.

## Concepts clés

### OTel Collector

L'**OTel Collector** est un proxy qui :
1. **Reçoit** les données (logs, metrics, traces) depuis les apps
2. **Transforme** les données (filtrage, enrichissement)
3. **Exporte** vers les backends (Loki, Prometheus, Tempo, Jaeger...)

Déployé en **DaemonSet** (comme Promtail) ou en **Deployment** centralisé.

### Instrumentation

Les applications peuvent envoyer des données à OTel de 2 façons :

| Méthode | Description | Exemple |
|---|---|---|
| **Auto-instrumentation** | Injection automatique sans modifier le code | OTel Operator injecte un agent Java |
| **Manuel (SDK)** | Ajouter le SDK OTel dans le code | `from opentelemetry import trace` |

### Tempo (backend de traces)

**Grafana Tempo** est le backend de stockage de traces (équivalent de Loki pour les logs).

Une **trace** = le parcours complet d'une requête à travers plusieurs services.

Exemple : requête HTTP → API Gateway → Service Auth → Base de données
- Durée totale : 250ms
- Auth a pris 180ms → **bottleneck identifié**

## OTel Collector vs Grafana Alloy

**Question** : pourquoi utiliser OTel Collector alors que Grafana propose **Alloy** ?

> **Alloy** = successeur de Promtail/Grafana Agent, construit sur les standards OTel mais avec des optimisations Grafana.

### Lequel choisir ?

| Cas d'usage | Recommandation |
|---|---|
| **Apprendre le standard** | OTel Collector |
| **Stack 100% Grafana** | Alloy (plus simple) |
| **Multi-vendor** (aujourd'hui Grafana, demain Datadog) | OTel Collector |
| **Production Grafana Cloud** | Alloy (recommandé par Grafana) |

**Dans ce TP**, on utilise **OTel Collector** pour comprendre le standard industriel.

---

# PHASE 9 — Déployer Tempo et OpenTelemetry Collector

Dans cette phase, on va ajouter le support des **traces** à notre stack d'observabilité.

**Plan** :
1. Déployer **Tempo** (backend de stockage de traces)
2. Déployer **OpenTelemetry Collector** (collecteur de traces)
3. Configurer **Grafana** pour visualiser les traces
4. Tester avec une application instrumentée

---

## Etape 19 — Déployer Tempo (backend de traces)

**Tempo** stocke les traces distribuées. On utilise le mode **monolithic** (un seul pod — adapté au lab).

### Créer le fichier de configuration Tempo

Créer le fichier `obs/tempo-values.yaml` :

```yaml
tempo:
  # Pas d'authentification multi-tenant
  multitenancyEnabled: false

  # Stockage en local (pas de S3 en lab)
  storage:
    trace:
      backend: local
      local:
        path: /var/tempo/traces

# Mode monolithique — un seul pod
deployment:
  mode: monolithic
  replicas: 1

# Persistance des traces (8Gi suffit pour le lab)
persistentVolume:
  enabled: true
  size: 8Gi

# Désactiver les composants distribués
distributor:
  replicas: 0
ingester:
  replicas: 0
querier:
  replicas: 0
compactor:
  replicas: 0
```

### Installer Tempo

```bash
helm install my-tempo grafana/tempo \
  -n obs \
  -f obs/tempo-values.yaml \
  --version 1.10.1
```

Attendre que le pod soit `Running` :

```bash
kubectl get pods -n obs | grep tempo
```

**URL interne de Tempo** :
```
http://my-tempo.obs.svc.cluster.local:3100
```

---

## Etape 20 — Déployer OpenTelemetry Collector

L'**OTel Collector** va recevoir les traces depuis les applications et les envoyer à Tempo.

### Installer le cert-manager (prérequis pour OTel Operator)

L'OTel Operator a besoin de cert-manager pour gérer les certificats TLS des webhooks :

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
```

Attendre que cert-manager soit prêt :

```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=120s
```

### Installer OpenTelemetry Operator

L'**Operator** permet de déployer et gérer des OTel Collectors via des ressources Kubernetes.

```bash
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/download/v0.97.1/opentelemetry-operator.yaml
```

Vérifier le déploiement :

```bash
kubectl get pods -n opentelemetry-operator-system
```

### Créer la configuration de l'OTel Collector

Créer le fichier `obs/otel-collector.yaml` :

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel
  namespace: obs
spec:
  mode: deployment
  replicas: 1

  config: |
    # RECEIVERS — comment les données arrivent
    receivers:
      # OTLP = OpenTelemetry Protocol (standard)
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317  # Port gRPC
          http:
            endpoint: 0.0.0.0:4318  # Port HTTP

    # PROCESSORS — transformations des données
    processors:
      # Ajout des attributs K8s (namespace, pod, node...)
      k8sattributes:
        passthrough: false
        extract:
          metadata:
            - k8s.namespace.name
            - k8s.pod.name
            - k8s.pod.uid
            - k8s.deployment.name
            - k8s.node.name

      # Batch pour améliorer les performances
      batch:
        timeout: 10s
        send_batch_size: 1024

    # EXPORTERS — où envoyer les données
    exporters:
      # Export vers Tempo
      otlp/tempo:
        endpoint: http://my-tempo.obs.svc.cluster.local:4317
        tls:
          insecure: true

      # Export de debug (logs dans la console du collector)
      debug:
        verbosity: detailed

    # SERVICE — pipeline de traitement
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [k8sattributes, batch]
          exporters: [otlp/tempo, debug]
```

### Déployer l'OTel Collector

```bash
kubectl apply -f obs/otel-collector.yaml
```

Vérifier le déploiement :

```bash
kubectl get pods -n obs | grep otel
kubectl logs -n obs -l app.kubernetes.io/component=opentelemetry-collector
```

**URL interne de l'OTel Collector** :
```
# gRPC (port 4317)
http://otel-collector.obs.svc.cluster.local:4317

# HTTP (port 4318)
http://otel-collector.obs.svc.cluster.local:4318
```

---

## Etape 21 — Configurer la datasource Tempo dans Grafana

Dans Grafana : **Connections > Data Sources > Add data source > Tempo**

| Champ | Valeur |
|---|---|
| **URL** | `http://my-tempo.obs.svc.cluster.local:3100` |

Cliquer **Save & Test** → `Data source is working`.

---

## Etape 22 — Tester avec une application instrumentée

Pour générer des traces, on va déployer une application de démonstration instrumentée avec OpenTelemetry.

### Déployer l'app de démo OpenTelemetry

Créer le fichier `obs/demo-app.yaml` :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: otel-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tracegen
  namespace: otel-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tracegen
  template:
    metadata:
      labels:
        app: tracegen
    spec:
      containers:
      - name: tracegen
        image: otel/opentelemetry-collector-contrib:0.97.0
        args:
          - tracegen
          - --otlp-endpoint=otel-collector.obs.svc.cluster.local:4317
          - --otlp-insecure
          - --duration=5m
          - --rate=1
          - --service-name=demo-app
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

Cette application génère 1 trace par seconde pendant 5 minutes.

```bash
kubectl apply -f obs/demo-app.yaml
```

### Vérifier que les traces arrivent

Attendre 30 secondes, puis dans Grafana : **Explore** → sélectionner **Tempo**

Dans **Query type**, sélectionner **Search** :
- **Service Name** : `demo-app`

Cliquer **Run query** → les traces doivent apparaître.

Cliquer sur une trace pour voir le détail :
- **Spans** — les étapes de la requête
- **Duration** — temps de chaque étape
- **Attributes** — métadonnées (namespace, pod...)

---

## Etape 23 — Explorer une trace

Dans Grafana Explore avec Tempo, sélectionner une trace.

**Informations visibles** :
- **Service Map** — graphe des services appelés
- **Trace timeline** — durée de chaque span
- **Span details** — attributs, logs, événements

**Corrélation logs ↔ traces** :
Si l'application injecte le `trace_id` dans ses logs, Grafana peut afficher les logs liés à une trace.

---

## Recap — Stack d'observabilité complète

```mermaid
graph TD
  Apps["Applications K8s"] -->|OTLP traces| OTel["OTel Collector"]
  Pods["Pods K8s"] -->|logs| Promtail
  Pods -->|metrics| Prom["Prometheus"]

  OTel -->|traces| Tempo
  Promtail --> Loki

  Tempo --> Grafana
  Loki --> Grafana
  Prom --> Grafana

  style OTel fill:#f59e0b,color:#000
  style Tempo fill:#8b5cf6,color:#fff
  style Loki fill:#f59e0b,color:#000
  style Prom fill:#ef4444,color:#fff
  style Grafana fill:#1a1a2e,color:#fff
  style Promtail fill:#3b82f6,color:#fff
```

| Composant | Namespace | Port | Rôle |
|---|---|---|---|
| **Tempo** | `obs` | 3100 | Stockage de traces |
| **OTel Collector** | `obs` | 4317 (gRPC), 4318 (HTTP) | Collecte de traces |
| **Loki** | `obs` | 3100 | Stockage de logs |
| **Prometheus** | `obs` | 9090 | Stockage de métriques |
| **Grafana** | `obs` | 3000 | Visualisation |

---

## Pour aller plus loin

| Sujet | Ressource |
|---|---|
| Loki — documentation officielle | [grafana.com/docs/loki/latest](https://grafana.com/docs/loki/latest/) |
| Promtail — guide d'installation | [grafana.com/docs/loki/latest/send-data/promtail](https://grafana.com/docs/loki/latest/send-data/promtail/installation/) |
| Prometheus — documentation | [prometheus.io/docs](https://prometheus.io/docs/introduction/overview/) |
| LogQL — langage de requête Loki | [grafana.com/docs/loki/latest/query](https://grafana.com/docs/loki/latest/query/) |
| PromQL — langage de requête Prometheus | [prometheus.io/docs/prometheus/latest/querying](https://prometheus.io/docs/prometheus/latest/querying/basics/) |
| Grafana Dashboards | [grafana.com/grafana/dashboards](https://grafana.com/grafana/dashboards/) |
| OpenTelemetry — documentation | [opentelemetry.io/docs](https://opentelemetry.io/docs/) |
| Tempo — documentation | [grafana.com/docs/tempo/latest](https://grafana.com/docs/tempo/latest/) |
| OTel Collector — configuration | [opentelemetry.io/docs/collector](https://opentelemetry.io/docs/collector/configuration/) |
