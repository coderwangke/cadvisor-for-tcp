# cadvisor-for-tcp

## cadvisor部署

* `--disable_metrics` 指定要禁止cadvisor生成的指标，避免数据量过大，消耗prom和cadvisor太多的资源。有些metrics是必须的(比如cpu和memory)无法禁止，通常我们把不需要的，并且可以禁止的指标都禁止掉。此例只不禁止tcp相关指标(如container_network_tcp_usage_total)，用于监控pod连接数
* 镜像同步到ccr，cadvisor目前在gcr.io/google_containers/cadvisor更新，用latest tag也就是0.34.0的镜像。
* `--store_container_labels=false` 指示 cadvisor 不生成容器相关的 label，比如环境变量(container_env_开头的label)，会有大量没有必要的数据，消耗过多的资源。`--whitelisted_container_labels` 指定容器 label 的白名单，在 `--store_container_labels=false` 的情况下，在白名单内的容器 label 可以保留。

## Prometheus 抓取配置
``` yaml
scrape_configs:
- job_name: cadvisor-tcp
  honor_timestamps: true
  metrics_path: /metrics
  scheme: http
  kubernetes_sd_configs:
  - role: endpoints
    namespaces:
      names:
      - cadvisor
  relabel_configs:
  - source_labels: [__meta_kubernetes_service_label_app]
    separator: ;
    regex: cadvisor
    replacement: $1
    action: keep
  - source_labels: [__meta_kubernetes_endpoint_port_name]
    separator: ;
    regex: http
    replacement: $1
    action: keep
  - source_labels: [__meta_kubernetes_endpoint_node_name]
    separator: ;
    regex: (.*)
    target_label: node
    replacement: ${1}
    action: replace
  metric_relabel_configs:
  - source_labels: [__name__]
    separator: ;
    regex: container_network_tcp_usage_total
    replacement: $1
    action: keep
  - source_labels: [container_label_io_kubernetes_pod_name]
    separator: ;
    regex: (.+)
    target_label: pod
    replacement: $1
    action: replace
  - source_labels: [container_label_io_kubernetes_pod_namespace]
    separator: ;
    regex: (.+)
    target_label: namespace
    replacement: $1
    action: replace

```
* 只保留需要的指标，此例是 `container_network_tcp_usage_total`，如需保留多个，用"|"隔开，如：`container_network_tcp_usage_total|container_network_udp_usage_total`。
* 将 `container_label_io_kubernetes_pod_name` 替换成 `pod`，`container_label_io_kubernetes_pod_namespace` 替换成 `namespace`，以便跟 kube-state-metrics 的 mixin_pod_workload 指标做关联查询。
