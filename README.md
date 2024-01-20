# OTEL + Grafana + Loki + Prometheus + Flux

## Підіймаємо кластер kind

```sh
➜ kind create cluster 
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.27.3) 🖼
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
```

### Створюємо репозиторій в поточній теці та заливаємо його на GitHub

Створюємо репозиторій на GitHub

```sh
git init
git commit --allow-empty -m "Repo init"
gh repo create --private --source=. --push

```

### Розгортаємо Flux в кластері kind

```sh
flux bootstrap github \
--token-auth \
--owner Andygol \
--repository otel \
--branch main \
--path clusters/kind \
--personal
Please enter your GitHub personal access token (PAT): 
► connecting to github.com
► cloning branch "main" from Git repository "https://github.com/Andygol/otel.git"
✔ cloned repository
► generating component manifests
✔ generated component manifests
✔ committed component manifests to "main" ("ce12a72ce447bf299b48a915765635fafedc0acf")
► pushing component manifests to "https://github.com/Andygol/otel.git"
► installing components in "flux-system" namespace
✔ installed components
✔ reconciled components
► determining if source secret "flux-system/flux-system" exists
► generating source secret
► applying source secret "flux-system/flux-system"
✔ reconciled source secret
► generating sync manifests
✔ generated sync manifests
✔ committed sync manifests to "main" ("3e65c0b5a25bc7a725ac9e0f63b4af004b8b0b87")
► pushing sync manifests to "https://github.com/Andygol/otel.git"
► applying sync manifests
✔ reconciled sync configuration
◎ waiting for GitRepository "flux-system/flux-system" to be reconciled
✔ GitRepository reconciled successfully
◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
✔ Kustomization reconciled successfully
► confirming components are healthy
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
```

Отримуємо результат розгортання Flux в кластері kind

```sh
git pull origin main
```

## Встановлення Cert-Manager

Створимо теку для Cert-Manager

```sh
mkdir -p clusters/kind/cert-manager
```

Згенеруємо маніфести для розгортання Cert-Manager в кластері kind згідно з інструкцією <https://cert-manager.io/docs/installation/continuous-deployment-and-gitops/>

### HelmRepository

```sh
flux create source helm cert-manager \
--url https://charts.jetstack.io \
--namespace cert-manager \
--export > clusters/kind/cert-manager/cert-manager-helmrepository.yaml
```

### HelmRelease

```sh
cat <<EOF > clusters/kind/cert-manager/values.yaml
# values.yaml
installCRDs: true
EOF
```

values.yaml не додаємо в репозиторій, оскільки він міститься в HelmRelease

```sh
flux create helmrelease cert-manager \
--namespace cert-manager \
--source HelmRepository/cert-manager.cert-manager \
--chart cert-manager \
--chart-version 1.13.x \
--values ./clusters/kind/cert-manager/values.yaml \
--export > clusters/kind/cert-manager/cert-manager-helmrelease.yaml
```

### Namespace

```sh
kubectl create namespace cert-manager \
--dry-run=client \
-o yaml > clusters/kind/cert-manager/cert-manager-namespace.yaml 
```

Збережемо зміни в репозиторії

```sh
git add .
git commit -m "Add cert-manager HelmRelease, HelmRepository, namespace, and values.yaml

This commit adds the necessary YAML files for cert-manager, including the HelmRelease, HelmRepository, namespace, and values.yaml."
git push origin main
```

## Monitoring

Створення теки для моніторингу

```sh
mkdir -p clusters/kind/monitoring
```

namespace Monitoring

```sh
kubectl create namespace monitoring \
--dry-run=client \
-o yaml > clusters/kind/monitoring/monitoring-namespace.yaml
```

## Встановлення OpenTelemetry Operator

### Створення HelmRepository для OpenTelemetry

```sh
mkdir -p clusters/kind/monitoring/opentelemetry
```

```sh
flux create source helm opentelemetry \
  --url https://open-telemetry.github.io/opentelemetry-helm-charts \
  --namespace monitoring \
  --export > clusters/kind/monitoring/opentelemetry/opentelemetry-helmrepository.yaml
```

### Створення HelmRelease для OpenTelemetry

```sh
cat <<EOF > clusters/kind/monitoring/opentelemetry/otel-operator-values.yaml
admissionWebhooks.certManager.enabled: false
admissionWebhooks.certManager.autoGenerateCert.enabled: true
manager.featureGates: operator.autoinstrumentation.go
EOF
```

otel-operator-values.yaml не додаємо в репозиторій, оскільки він міститься в HelmRelease

```sh
flux create helmrelease opentelemetry-operator \
  --chart opentelemetry-operator \
  --source HelmRepository/opentelemetry.monitoring \
  --namespace monitoring \
  --values clusters/kind/monitoring/opentelemetry/otel-operator-values.yaml \
  --export > clusters/kind/monitoring/opentelemetry/opentelemetry-operator-helmrelease.yaml 
```

opentelemetry-collector.yaml

```sh
cat <<EOF > clusters/kind/monitoring/opentelemetry/opentelemetry-collector.yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: opentelemetry
  namespace: monitoring
spec:
  mode: daemonset
  hostNetwork: true
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
            endpoint: "0.0.0.0:3030"

    exporters:
      logging:
      loki:
        endpoint: http://loki:3100/loki/api/v1/push
      prometheus:
        endpoint: "0.0.0.0:8889"

    service:
      pipelines:
        logs:
          receivers: [otlp]
          exporters: [loki]
        traces:
          receivers: [otlp]
          exporters: [logging]
        metrics:
          receivers: [otlp]
          exporters: [logging,prometheus]
EOF
```

## Fluent Bit

### Створення HelmRepository для Fluent Bit

```sh
mkdir -p clusters/kind/monitoring/fluentbit
```

```sh
flux create source helm fluentbit \
  --url https://fluent.github.io/helm-charts \
  --namespace monitoring \
  --export > clusters/kind/monitoring/fluentbit/fluentbit-helmrepository.yaml
```

### Створення HelmRelease для Fluent Bit

```sh
flux create helmrelease fluentbit \
  --chart fluent-bit \
  --source HelmRepository/fluentbit.monitoring \
  --namespace monitoring \
  --export > clusters/kind/monitoring/fluentbit/fluentbit-helmrelease.yaml
```

### Створення ConfigMap для Fluent Bit

```sh
cat <<EOF > clusters/kind/monitoring/fluentbit/fluentbit-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit
  namespace: monitoring
  labels:
    k8s-app: fluent-bit
data:
  fluent-bit.conf: |
    [SERVICE]
            Flush         1
            Log_Level     info
            # Log_Level     error
            # Allowed values are: off, error, warn, info, debug and trace. 
            #If 'debug' is set, it will include error, warning, info and debug.
      
      inputs: |
       [INPUT]
           Name              tail
           Tag               kube.*
           Path              /var/log/containers/*.log
           multiline.parser  docker, cri
           Refresh_Interval  10
           Ignore_Older      6h
           Docker_Mode       On
           Tag_Regex         (?<kube_namespace>[^_]+)_(?<kube_pod>[^_]+)_(?<kube_container>[^_]+)_(?<kube_id>[^_]+)_(?<kube_name>.*)\.(?<kube_format>log|log\.gz|txt|json|out)$
           Tag               kube.${kube_namespace}.${kube_pod}.${kube_container}.${kube_id}.${kube_name}.${kube_format}
      
      filters: |
        [FILTER]
            Name                kubernetes
            Match               kube.*
            Merge_Log           On
            Merge_Log_Key       log_processed
            
      outputs: |
        [OUTPUT]
            Name            opentelemetry
            Match           *
            Host            collector
            Port            3030
            metrics_uri     /v1/metrics
            logs_uri        /v1/logs
            trace_uri       /v1/traces
            Log_response_payload True
            tls             On
            tls_verify      On
            add_labels      app fluent-bit
            add_labels      color blue
EOF
```

## Prometheus

```sh
mkdir -p clusters/kind/monitoring/prometheus
```

### Створення HelmRepository для Prometheus

```sh
flux create source helm prometheus \
  --url https://prometheus-community.github.io/helm-charts \
  --namespace monitoring \
  --export > clusters/kind/monitoring/prometheus/prometheus-helmrepository.yaml
```

### Створення HelmRelease для Prometheus

```sh
flux create helmrelease prometheus \
  --chart prometheus \
  --source HelmRepository/prometheus.monitoring \
  --namespace monitoring \
  --export > clusters/kind/monitoring/prometheus/prometheus-helmrelease.yaml
```

ConfigMap для Prometheus

```sh
cat <<EOF > clusters/kind/monitoring/prometheus/prometheus-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    k8s-app: prometheus
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s 
      evaluation_interval: 15s

    scrape_configs:
      - job_name: otel_collector
        scrape_interval: 5s
        static_configs:
          - targets: ['collector:8889']

      - job_name: 'prometheus'
        static_configs:
          - targets: [ 'localhost:9090' ]
EOF
```

## Grafana + Loki

```sh
mkdir -p clusters/kind/monitoring/grafana/{grafana,loki}
```

### Створення HelmRepository для Grafana + Loki

```sh
flux create source helm grafana \
  --url https://grafana.github.io/helm-charts \
  --namespace monitoring \
  --export > clusters/kind/monitoring/grafana/grafana-helmrepository.yaml
```

### Створення HelmRelease для Loki

```sh
cat <<EOF > clusters/kind/monitoring/grafana/loki/loki-values.yaml
# values.yaml
loki:
  commonConfig:
    replication_factor: 1
  storage:
    type: 'filesystem'
singleBinary:
  replicas: 1
EOF
```

loki-values.yaml не додаємо в репозиторій, оскільки він міститься в HelmRelease

```sh
flux create helmrelease loki \
  --chart loki \
  --source HelmRepository/grafana.monitoring \
  --namespace monitoring \
  --values clusters/kind/monitoring/grafana/loki/loki-values.yaml \
  --export > clusters/kind/monitoring/grafana/loki/loki-helmrelease.yaml
```

ConfigMap для Loki

```sh
cat <<EOF > clusters/kind/monitoring/grafana/loki/loki-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki
  namespace: monitoring
data:
  config.yaml: |
    auth_enabled: true
    common:
      compactor_address: 'loki'
      path_prefix: /var/loki
      replication_factor: 1
      storage:
        filesystem:
          chunks_directory: /var/loki/chunks
          rules_directory: /var/loki/rules
    frontend:
      scheduler_address: ""
    frontend_worker:
      scheduler_address: ""
    index_gateway:
      mode: ring
    limits_config:
      max_cache_freshness_per_query: 10m
      reject_old_samples: true
      reject_old_samples_max_age: 168h
      split_queries_by_interval: 15m
    memberlist:
      join_members:
      - loki-memberlist
    query_range:
      align_queries_with_step: true
      results_cache:
        cache:
          embedded_cache:
            max_size_mb: 100
            enabled: true
    ruler:
    #   alertmanager_url: http://alertmanager.monitoring.svc:9093
      alertmanager_url: http://localhost:9093
      storage:
        type: local
    runtime_config:
      file: /etc/loki/runtime-config/runtime-config.yaml
    schema_config:
      configs:
      - from: "2022-01-11"
        index:
          period: 24h
          prefix: loki_index_
        object_store: filesystem
        schema: v12
        store: boltdb-shipper
    server:
      grpc_listen_port: 9095
      http_listen_port: 3100
    storage_config:
      hedging:
        at: 250ms
        max_per_second: 20
        up_to: 3
    tracing:
      enabled: false
    analytics:
      reporting_enabled: false
EOF
```

### Створення HelmRelease для Grafana

```sh
cat <<EOF > clusters/kind/monitoring/grafana/grafana/grafana-values.yaml
# values.yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Loki
      type: loki
      access: proxy
      url: http://loki:3100
      editable: true
      version: 1
      basicAuth: false
      isDefault: false
    - name: Prometheus
      type: prometheus
      uid: prometheus
      access: proxy
      # url: http://prometheus:9090
      url: http://prometheus-server.monitoring.svc
      jsonData:
        httpMethod: GET
      editable: true
      isDefault: true
      basicAuth: false
      orgId: 1
      version: 1
env:
  GF_AUTH_ANONYMOUS_ENABLED: "true"
  GF_AUTH_ANONYMOUS_ORG_ROLE: Admin
  GF_AUTH_DISABLE_LOGIN_FORM: "true"
  GF_FEATURE_TOGGLES_ENABLE: "traceqlEditor"
  GF_SERVER_PORT: "3000"
EOF
```

grafana-values.yaml не додаємо в репозиторій, оскільки він міститься в HelmRelease

```sh
flux create helmrelease grafana \
  --chart grafana \
  --source HelmRepository/grafana.monitoring \
  --namespace monitoring \
  --values clusters/kind/monitoring/grafana/grafana/grafana-values.yaml \
  --export > clusters/kind/monitoring/grafana/grafana/grafana-helmrelease.yaml
```
