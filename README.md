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

## Monitoring

Створення теки для моніторингу

```sh
mkdir -p clusters/kind/monitoring
```

Створення namespace Monitoring

```sh
kubectl create namespace monitoring \
--dry-run=client \
-o yaml > clusters/kind/monitoring/monitoring-namespace.yaml
```

## Встановлення Cert-Manager

Створимо теку для Cert-Manager

```sh
mkdir -p clusters/kind/monitoring/cert-manager
```

Згенеруємо маніфести для розгортання Cert-Manager в кластері kind згідно з інструкцією <https://cert-manager.io/docs/installation/continuous-deployment-and-gitops/>

### HelmRepository

```sh
flux create source helm cert-manager \
--url https://charts.jetstack.io \
--namespace monitoring \
--export > clusters/kind/monitoring/cert-manager/cert-manager-helmrepository.yaml
```

### HelmRelease

```sh
cat <<EOF > clusters/kind/monitoring/cert-manager/values.yaml
# values.yaml
installCRDs: true
EOF
```

⚠️ values.yaml не додаємо в репозиторій, оскільки він міститься в HelmRelease

```sh
flux create helmrelease cert-manager \
--namespace monitoring \
--source HelmRepository/cert-manager.monitoring \
--chart cert-manager \
--chart-version 1.13.x \
--values ./clusters/kind/monitoring/cert-manager/values.yaml \
--export > clusters/kind/monitoring/cert-manager/cert-manager-helmrelease.yaml
```

Збережемо зміни в репозиторії

```sh
git add .
git commit -m "Add cert-manager HelmRelease, HelmRepository, namespace, and values.yaml

This commit adds the necessary YAML files for cert-manager, including the HelmRelease, HelmRepository, namespace, and values.yaml."
git push origin main
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

⚠️ otel-operator-values.yaml не додаємо в репозиторій, оскільки він міститься в HelmRelease

```sh
flux create helmrelease opentelemetry-operator \
  --chart opentelemetry-operator \
  --source HelmRepository/opentelemetry.monitoring \
  --namespace monitoring \
  --values clusters/kind/monitoring/opentelemetry/otel-operator-values.yaml \
  --export > clusters/kind/monitoring/opentelemetry/opentelemetry-operator-helmrelease.yaml 
```

Після створення OpenTelemetry Operator створимо маніфест opentelemetry-collector.yaml

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

### Створення інструментарію для OpenTelemetry

(🤔 потреба в створенні інструментарію досліджується, можливо цей маніфест не потрібний)

```sh
cat <<EOF > clusters/kind/monitoring/opentelemetry/opentelemetry-instrumentation.yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: opentelemetry-instrumentation
  namespace: monitoring
spec:
  exporter:
    endpoint: http://opentelemetry-collector:4317
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "1"
  go:
    env:
      # Required if endpoint is set to 4317.
      # Go autoinstrumentation uses http/proto by default
      # so data must be sent to 4318 instead of 4317.
      - name: OTEL_EXPORTER_OTLP_ENDPOINT
        value: http://opentelemetry-collector:4318
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
  name: fluentbit-fluent-bit
  namespace: monitoring
  labels:
    k8s-app: fluent-bit
data:
  custom_parsers.conf: |
    [PARSER]
        Name docker_no_time
        Format json
        Time_Keep Off
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
  fluent-bit.conf: |
    [SERVICE]
        Daemon Off
        Flush 1
        Log_Level info
        Parsers_File /fluent-bit/etc/parsers.conf
        Parsers_File /fluent-bit/etc/conf/custom_parsers.conf
        HTTP_Server On
        HTTP_Listen 0.0.0.0
        HTTP_Port 2020
        Health_Check On

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        # Exclude_Path      /var/log/containers/*_kube-system_*.log,/var/log/containers/*_logging_*.log,/var/log/containers/*_ingress-nginx_*.log,/var/log/containers/*_kube-node-lease_*.log,/var/log/containers/*_kube-public_*.log,/var/log/containers/*_cert-manager_*.log,/var/log/containers/*_prometheus-operator_*.log
        multiline.parser  docker, cri
        Refresh_Interval  10
        Ignore_Older      6h
        Docker_Mode       On
        Tag_Regex         var.log.containers.(?<pod_name>[a-z0-9](?:[-a-z0-9]*[a-z0-9])?(?:\\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*)_(?<namespace_name>[^_]+)_(?<container_name>.+)-(?<docker_id>[a-z0-9]{64})\.log$
        Tag               kube.${kube_namespace}.${kube_pod}.${kube_container}.${kube_id}.${kube_name}.${kube_format}

    [INPUT]
        Name systemd
        Tag host.*
        Systemd_Filter _SYSTEMD_UNIT=kubelet.service
        Read_From_Tail On

    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Merge_Log_Key log_processed
        Keep_Log Off
        K8S-Logging.Parser On
        K8S-Logging.Exclude On

    [OUTPUT]
        Name            opentelemetry
        Match           *
        Host            opentelemetry-collector
        Port            3030
        metrics_uri     /v1/metrics
        logs_uri        /v1/logs
        Log_response_payload True
        tls             off
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

ConfigMap для Prometheus, який містить конфігурацію для збору метрик з OpenTelemetry Collector створюємо з базового маніфесту

```sh
cat <<EOF > clusters/kind/monitoring/prometheus/prometheus-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-server
  namespace: monitoring
  labels:
    k8s-app: prometheus
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
      scrape_timeout: 10s

    rule_files:
    - /etc/config/recording_rules.yml
    - /etc/config/alerting_rules.yml
    - /etc/config/rules
    - /etc/config/alerts

    scrape_configs:
    - job_name: otel_collector
      scrape_interval: 5s
      static_configs:
        - targets: ['collector:8889']

    - job_name: prometheus
      static_configs:
      - targets: [ 'localhost:9090' ]

    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      job_name: kubernetes-apiservers
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: default;kubernetes;https
        source_labels:
        - __meta_kubernetes_namespace
        - __meta_kubernetes_service_name
        - __meta_kubernetes_endpoint_port_name
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true

    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      job_name: kubernetes-nodes
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - replacement: kubernetes.default.svc:443
        target_label: __address__
      - regex: (.+)
        replacement: /api/v1/nodes/$1/proxy/metrics
        source_labels:
        - __meta_kubernetes_node_name
        target_label: __metrics_path__
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true

    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      job_name: kubernetes-nodes-cadvisor
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - replacement: kubernetes.default.svc:443
        target_label: __address__
      - regex: (.+)
        replacement: /api/v1/nodes/$1/proxy/metrics/cadvisor
        source_labels:
        - __meta_kubernetes_node_name
        target_label: __metrics_path__
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true

    - honor_labels: true
      job_name: kubernetes-service-endpoints
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape
      - action: drop
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (.+?)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_kubernetes_service_annotation_prometheus_io_port
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_service_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_service_name
        target_label: service
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: node

    - honor_labels: true
      job_name: kubernetes-service-endpoints-slow
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (.+?)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_kubernetes_service_annotation_prometheus_io_port
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_service_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_service_name
        target_label: service
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: node
      scrape_interval: 5m
      scrape_timeout: 30s

    - honor_labels: true
      job_name: prometheus-pushgateway
      kubernetes_sd_configs:
      - role: service
      relabel_configs:
      - action: keep
        regex: pushgateway
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_probe

    - honor_labels: true
      job_name: kubernetes-services
      kubernetes_sd_configs:
      - role: service
      metrics_path: /probe
      params:
        module:
        - http_2xx
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_probe
      - source_labels:
        - __address__
        target_label: __param_target
      - replacement: blackbox
        target_label: __address__
      - source_labels:
        - __param_target
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - source_labels:
        - __meta_kubernetes_service_name
        target_label: service

    - honor_labels: true
      job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape
      - action: drop
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (\d+);(([A-Fa-f0-9]{1,4}::?){1,7}[A-Fa-f0-9]{1,4})
        replacement: '[$2]:$1'
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: replace
        regex: (\d+);((([0-9]+?)(\.|$)){4})
        replacement: $2:$1
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_name
        target_label: pod
      - action: drop
        regex: Pending|Succeeded|Failed|Completed
        source_labels:
        - __meta_kubernetes_pod_phase
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: node

    - honor_labels: true
      job_name: kubernetes-pods-slow
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - action: keep
        regex: true
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape_slow
      - action: replace
        regex: (https?)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scheme
        target_label: __scheme__
      - action: replace
        regex: (.+)
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: (\d+);(([A-Fa-f0-9]{1,4}::?){1,7}[A-Fa-f0-9]{1,4})
        replacement: '[$2]:$1'
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: replace
        regex: (\d+);((([0-9]+?)(\.|$)){4})
        replacement: $2:$1
        source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_annotation_prometheus_io_param_(.+)
        replacement: __param_$1
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - action: replace
        source_labels:
        - __meta_kubernetes_namespace
        target_label: namespace
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_name
        target_label: pod
      - action: drop
        regex: Pending|Succeeded|Failed|Completed
        source_labels:
        - __meta_kubernetes_pod_phase
      - action: replace
        source_labels:
        - __meta_kubernetes_pod_node_name
        target_label: node
      scrape_interval: 5m
      scrape_timeout: 30s

    alerting:
      alertmanagers:
      - kubernetes_sd_configs:
          - role: pod
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace]
          regex: monitoring
          action: keep
        - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_instance]
          regex: prometheus
          action: keep
        - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
          regex: alertmanager
          action: keep
        - source_labels: [__meta_kubernetes_pod_container_port_number]
          regex: "9093"
          action: keep

EOF
```

## Grafana + Loki

Створимо теки для Grafana + Loki

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
memberlist:
  service:
    publishNotReadyAddresses: true
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

ConfigMap для Loki створюємо з базового маніфесту

```sh
cat <<EOF > clusters/kind/monitoring/grafana/loki/loki-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki
  namespace: monitoring
data:
  config.yaml: |
    auth_enabled: false
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

## Sealed Secrets

```sh
mkdir -p clusters/kind/sealed-secrets
```

### Створення HelmRepository для Sealed Secrets

```sh
flux create source helm sealed-secrets \
  --url https://bitnami-labs.github.io/sealed-secrets \
  --export > clusters/kind/sealed-secrets/sealed-secrets-helmrepository.yaml
```

### Створення HelmRelease для Sealed Secrets

```sh
flux create helmrelease sealed-secrets \
  --chart sealed-secrets \
  --source HelmRepository/sealed-secrets \
  --target-namespace flux-system \
  --release-name sealed-secrets-controller \
  --crds CreateReplace \
  --chart-version ">=1.15.0-0" \
  --export > clusters/kind/sealed-secrets/sealed-secrets-helmrelease.yaml
```

### Встановлення Sealed Secrets

```sh
brew install kubeseal
```

### Отримання ключа для Sealed Secrets

```sh
kubeseal --fetch-cert \
--controller-name=sealed-secrets-controller \
--controller-namespace=flux-system \
> clusters/kind/sealed-secrets/sealed-secrets-cert.pem
```

## Kbot

```sh
mkdir -p clusters/kind/kbot
```

### Створення namespace для Kbot

kbot-namspace.yaml

```sh
cat <<EOF > clusters/kind/kbot/kbot-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kbot
EOF
```

### Створення секрету

```sh
read -s TELE_TOKEN
kubectl -n kbot create secret generic kbot \
--dry-run=client \
--from-literal=token=$TELE_TOKEN \
-o yaml > clusters/kind/sealed-secrets/secret.yaml
```

### Шифрування секрету

```sh
kubeseal --format=yaml \
--cert=clusters/kind/sealed-secrets/sealed-secrets-cert.pem \
< clusters/kind/sealed-secrets/secret.yaml > clusters/kind/sealed-secrets/secret-sealed.yaml
rm clusters/kind/sealed-secrets/secret.yaml
```

### Створюємо маніфесту для розгортання Kbot

```sh
cat <<EOF > clusters/kind/kbot/kbot-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kbot
  namespace: kbot
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kbot
  template:
    metadata:
      labels:
        app: kbot
      annotations:
        # instrumentation.opentelemetry.io/inject-go: "monitoring/opentelemetry-instrumentation"
        # instrumentation.opentelemetry.io/otel-go-auto-target-exe: "/kbot"
        sidecar.opentelemetry.io/inject: "monitoring/opentelemetry-collector"
    spec:
      containers:
      - name: kbot
        image: denvasyliev/kbot:v1.0.0-otel
        securityContext:
          capabilities:
            add:
              - SYS_PTRACE
          privileged: true
          runAsUser: 0
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 200m
            memory: 256Mi
        env:
        - name: TELE_TOKEN
          valueFrom:
            secretKeyRef:
              name: kbot
              key: token
        - name: METRICS_HOST
          value: opentelemetry-collector.monitoring.svc.cluster.local:4318
EOF
```

Всі маніфести мають бути збережені в репозиторій після чого Flux автоматично розгорне їх в кластері kind

## Дашборад

### Відкриваємо порти для доступу до дашбордів

```sh
kubectl port-forward service/grafana 3000:80 -n monitoring 
```

або

```sh
kubectl port-forward deployments/grafana 3000:3000 -n monitoring
```

Переходимо за посиланням <http://localhost:3000> для роботи з дашбордом Grafana.
