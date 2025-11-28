# Kubernetes Prometheus Auto-discovery Configuration Guide

## I. Background

In a Kubernetes (K8s) cluster, resources such as Pods and Services frequently change due to scheduling, scaling, fault restarts, etc., making it difficult to fix their IPs, names, and other attributes. The traditional method of statically configuring scraping targets requires manual maintenance of scraping addresses, which is not only inefficient but also prone to scraping failures due to resource changes.
The Prometheus auto-discovery mechanism, combined with the Datakit collector, implements dynamic identification of scraping targets and metric pulling through the method of "K8s resource annotations + collector rule configuration". It can adapt to the dynamic characteristics of K8s clusters without manual intervention, greatly simplifying the monitoring configuration process.

## II. Implementation Principle

The core of Prometheus auto-discovery is "K8s event listening + dynamic configuration generation". Datakit inherits this principle and optimizes label remapping in combination with its own collector configuration specifications, achieving the effect of enabling and auto-discovering services in Datakit Daemonset.
Auto-discovery process:
1. Event listening registration: The Datakit collector registers an event notification mechanism with the K8s API Server to real-time monitor the creation, update, and deletion events of resources such as Pods and Services.
2. Resource filtering and judgment: When a resource changes, the collector determines whether the resource needs to be scraped based on preset configurations (namespace, label selector, etc.).
3. Scraping address construction: For eligible resources, the collector extracts resource attributes (such as Pod IP, port/path configured in annotations) through placeholders in the configuration to automatically construct a metric scraping address (e.g., http://PodIP:Port/Path).
4. Metric scraping and parsing: The collector accesses the constructed address, pulls Prometheus-format metrics, parses them, adds custom labels (such as Pod name and namespace), and reports them to the observation platform.
5. Resource change adaptation: If a resource is updated (e.g., annotation modification) or deleted, the collector responds in real-time to update the scraping configuration or terminate scraping.

## III. Introduction to Two Auto-discovery Mechanisms

Datakit auto-discovery has two effective mechanisms. The first is the `<Prometheus Annotation Configuration>` mode based on **"enable_*" tag group + workload annotations**. In this mode, when configuring the auto-discovery function of the Datakit `KubernetesPrometheus` collector, you only need to modify the corresponding item in the following switches to `True` for the type of resource you need to discover:
```toml
enable_discovery_of_prometheus_pod_annotations = false
enable_discovery_of_prometheus_service_annotations = false
enable_discovery_of_prometheus_pod_monitors = false
enable_discovery_of_prometheus_service_monitors = false
```
After completing this modification, you do not need to continue with the subsequent collector instance configuration. Service auto-discovery can be achieved by directly adding annotations to the workload. The purpose of this is to be compatible with the Prometheus ecosystem. For example, in some environments, customers have already configured auto-discovery annotations for Prometheus Exporters and hope that Datakit can be directly compatible with this configuration to achieve auto-discovery without modifying the workload. It can also simplify the configuration operation of your new workloads, such as eliminating the need to maintain a lengthy datakit collector configuration table. However, the disadvantage is that the configuration content supported by annotations is limited, and it is only applicable to scenarios where metrics are obtained through simple HTTP. In addition, in this mode, the annotations related to auto-discovery in the workload are mandatory. If no annotations are configured, the auto-discovery function may fail to take effect.

The second is to **disable the above flags** (set to `false` or not configure them, whose default value is `false`), and use the configuration content in the **collector instance configuration block** to guide Datakit for auto-discovery (`<Collector Instance Configuration>`). In this mechanism, the collector instance will determine which services need to be auto-discovered, where to obtain data, and which labels need to be added based on the parameter values set in the configuration file.
The advantage of this configuration is that the continuous integration team does not need to maintain annotations for the workload, and only needs to deploy pods using standard delivery templates. All configuration management work is concentrated on the datakit collector. Although the configuration file is relatively complex, it concentrates all configuration information. It is suitable for scenarios where centralized management of auto-discovery rules is required.

These two methods need to be chosen based on your own needs. If both the "enable_*" flags and the instance configuration block are enabled at the same time, it may lead to duplicate data collection, causing confusion in data usage and billing. Therefore, you need to first confirm which method to use for auto-discovery based on the initial state of your own environment and configuration management requirements, and then perform the corresponding configuration.

## IV. How to Enable Datakit Kubernetes Auto-discovery

### 4.1 Before We Start

Regardless of which auto-discovery mechanism we choose, we need to do some preparation work before starting the configuration to proceed with the subsequent configuration.

#### 4.1.1 Disable the Default Collector

After you complete the installation of Datakit DaemonSet, if you use the `kubectl describe` command to check the Datakit DaemonSet and retrieve the `ENV_DEFAULT_ENABLED_INPUTS` keyword, you will see content similar to the following:

```
- name: ENV_DEFAULT_ENABLED_INPUTS
  value: statsd,dk,cpu,disk,diskio,mem,swap,system,hostobject,net,host_processes,container,kubernetesprometheus,ddtrace,profile
```

If the Datakit version you are using has `"kubernetesprometheus"` set in this environment variable, please delete it. This preset collector flag will generate a set of default auto-discovery rules, which will overwrite the auto-discovery rules you customized and modified. Therefore, before we start, check this variable and delete "kubernetesprometheus" from it.

#### 4.1.2 Understand Datakit Daemonset Configuration Methods

Since subsequent operations require configuring Datakit's KubernetesPrometheus, it is necessary to briefly understand how these configuration files take effect. The steps to add a custom scraping configuration to enable a collector in Kubernetes are similar for all Datakit collectors: first, go to the official website to find the configuration file format of the collector, copy the format to your local editor, modify its content, inject the modified content into the cluster in the form of a configmap, and finally mount this configmap to the specified directory of datakit in the form of `volumeMount`. The relative path of the directory is usually described above the configuration file example in the document you are viewing. The absolute path starts with `/usr/local/datakit`.

#### Brief Introduction to Configuration Methods

Taking the configuration related to auto-discovery as an example, you first need to search for `KubernetesPrometheus` in the official documentation. In the [Example](https://docs.truewatch.com/integrations/kubernetesprometheus/#example) section of the opened document, you will see the mount location of the configuration file and the required file name:

```yaml
volumeMounts:
- mountPath: /usr/local/datakit/conf.d/kubernetesprometheus/kubernetesprometheus.conf
  name: datakit-conf
  subPath: kubernetesprometheus.conf
  readOnly: true
```

And a configMap example corresponding to the above mount, where the content we need to modify is after `"kubernetesprometheus.conf: |-"`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: datakit-conf
  namespace: datakit
data:
  kubernetesprometheus.conf: |-
    [inputs.kubernetesprometheus]
      # Enable NodeLocal mode to distribute auto-discovery tasks to each pod and avoid concentrated pressure
      node_local = true
      # Global scraping interval
      scrape_interval = "30s"

      # Whether to retain the original Prometheus metric names without truncation
      keep_exist_metric_name = true

      # Whether to use the timestamps of the metrics themselves
      honor_timestamps = true
     
      .....
```

All subsequent configurations are modifications to the content in kubernetesprometheus.conf. After completing the modification or configuration, overwrite the modified content to the configMap, and then execute volumeMount to make the configuration take effect.

#### 4.1.3 Understand the Structure of the kubernetesprometheus Configuration File
If you check the various configuration examples in [Example](https://docs.truewatch.com/integrations/kubernetesprometheus/#example), you will find that the structure of kubernetesprometheus is very complex. To help you better understand this configuration file, we will introduce it in two parts:

##### Collector Main Configuration
First, the collector main configuration structure, marked by [inputs.kubernetesprometheus] as the start of the main configuration. A main configuration structure includes:

```toml
[inputs.kubernetesprometheus]
  ## Collector main configuration block
  ...

  [inputs.kubernetesprometheus.global_tags]
  ## Collector global tag block
  ...

  [[inputs.kubernetesprometheus.instances]]
  ### Collector instance configuration block 1
  ...
  [[inputs.kubernetesprometheus.instances]]
  ### Collector instance configuration block 2
  ...
  [[inputs.kubernetesprometheus.instances]]
  ### Collector instance configuration block 3
  ...
```
Among them, the main configuration block is responsible for configuring the basic attributes of kubernetesprometheus, such as setting the collector's working mode, scraping interval, and metric data processing method. It also includes the switch group for enabling the `<Prometheus Annotation Configuration>` mode. The configuration of this main configuration block takes effect globally for the kubernetesprometheus collector. That is, for all subsequent individually configured collector instances, the values in the main configuration will affect the behavior of the collector instances.

[inputs.kubernetesprometheus.global_tags] is used to add some global tags to the collected data, such as the name of the current cluster, the affiliated project team, the availability zone where the cluster is located, and other public attribute tags. When these tags need to be globally attached to all collector instances, they can be set here.

For the Prometheus Annotation Configuration mode, using the collector main configuration and global tag block can meet the configuration requirements. However, please note that for each Datakit runtime, there should be only one `[inputs.kubernetesprometheus]` mark. If multiple `[inputs.kubernetesprometheus]` are used, the `KubernetesPrometheus` collector will fail to start, and only the sub-configuration in the first loaded `[inputs.kubernetesprometheus]` will take effect. The rest of the configurations will not take effect.

The detailed content and description of the collector main configuration block and global tag block are as follows:

```toml
[inputs.kubernetesprometheus]
  # true: Enable NodeLocal mode, which will make the Daemonset distribute auto-discovery tasks to each pod, avoiding concentrated pressure on a certain Datakit pod when multiple auto-discovery scraping instances for workloads are enabled. Set to false to disable this function
  node_local             = true   

  # Specify the scraping interval for all collector instances enabled in subsequent configurations. The default is 30 seconds, which can be modified
  scrape_interval        = "30s"

  # Whether to retain the original metric names. Since Datakit will truncate Prometheus metric names to adapt to TrueWatch metric storage requirements, if you want to retain the Prometheus metric names to follow your historical usage habits, set this flag to True
  keep_exist_metric_name = true   

  # Whether to enable predefined Pod Annotations configuration
  enable_discovery_of_prometheus_pod_annotations     = false

  # Whether to enable predefined Service Annotations configuration
  enable_discovery_of_prometheus_service_annotations = false

  # Whether to enable Prometheus PodMonitors CRD function
  enable_discovery_of_prometheus_pod_monitors        = false 
  
  # Whether to enable Prometheus ServiceMonitors CRD function
  enable_discovery_of_prometheus_service_monitors    = false 

  # Configure global tags for the collector instances you will enable later. The key-value pairs of tags filled in here will be added to all data collected by all collectors below
  [inputs.kubernetesprometheus.global_tags]
    instance = "__kubernetes_mate_instance"
    host     = "__kubernetes_mate_host"
    # .... You can continue to add other tags
```

If you adopt the `<Prometheus Annotation Configuration>` mode for auto-discovery, you only need to focus on the use of the main configuration block and global tag block. If you adopt the `<Collector Instance Configuration>` mode, in addition to the above two configurations, you also need to proceed with the following collector instance configuration.

##### Collector Instance Configuration

The structure of a collector instance is as follows:

```
[[inputs.kubernetesprometheus.instances]]
  ## Instance-level main configuration block

  [inputs.kubernetesprometheus.instances.http_headers]
  ## Instance-level request header configuration block

  [inputs.kubernetesprometheus.instances.custom]
  ## Instance-level custom tag block

  [inputs.kubernetesprometheus.instances.auth]
  ## Instance-level authentication resource configuration block
```

This structure includes the public configuration of each collector instance, namely the main configuration block. It also includes other customized configurations for each instance, such as request header configuration, local custom tags, and data access authentication-related content. This information usually varies depending on the auto-discovery target, so a separate configuration area is required to set it. And this configuration will not be shared among different collector instances.

In addition, unlike the [inputs.kubernetesprometheus] mark, multiple collector instance configurations can be enabled, each saving different configurations to correspond to different auto-discovery objects. For example, if you need to implement auto-discovery for Nginx, Redis, and MySQL instances at the same time, you can directly copy multiple copies of [[inputs.kubernetesprometheus.instances]]:

```toml
[inputs.kubernetesprometheus]
  ...

  [[inputs.kubernetesprometheus.instances]]
    ## Write Nginx auto-discovery rules here

  [[inputs.kubernetesprometheus.instances]]
    ## Write Redis auto-discovery rules here
    ...
  [[inputs.kubernetesprometheus.instances]]
    ## Write MySQL auto-discovery rules here
    ...
```

Please note that the order of `[[inputs.kubernetesprometheus.instances]]` is not important. It is only used to distinguish different collection rules and notify `kubernetesprometheus` of the number of corresponding collection instances that need to be enabled.

##### Detailed Explanation of Collector Instance Configuration

If you check the [Configuration Description](https://docs.truewatch.com/integrations/kubernetesprometheus/#input-config-added) section of the official website, you can see a complete configuration template for a collector instance:

```toml
[[inputs.kubernetesprometheus.instances]]
  role       = "pod"
  namespaces = ["middleware"]
  selector   = "app=nginx"

  scrape     = "true"
  scheme     = "http"
  port       = "__kubernetes_pod_container_nginx_port_metrics_number"
  path       = "/metrics"
  params     = ""
  
  [inputs.kubernetesprometheus.instances.http_headers]
      "Authorization" = "Bearer XXXXX"
      "X-testing-key" = "value"

  [inputs.kubernetesprometheus.instances.custom]
    measurement        = "pod-nginx"
    job_as_measurement = false
    [inputs.kubernetesprometheus.instances.custom.tags]
      instance         = "__kubernetes_mate_instance"
      host             = "__kubernetes_mate_host"
      pod_name         = "__kubernetes_pod_name"
      pod_namespace    = "__kubernetes_pod_namespace"

  [inputs.kubernetesprometheus.instances.auth]
    bearer_token_file      = "/var/run/secrets/kubernetes.io/serviceaccount/token"
    [inputs.kubernetesprometheus.instances.auth.tls_config]
      insecure_skip_verify = false
      ca_certs = ["/opt/nginx/ca.crt"]
      cert     = "/opt/nginx/peer.crt"
      cert_key = "/opt/nginx/peer.key"
```

Now, based on this template, we will introduce each block in the collector instance configuration.

##### Instance-level Main Configuration Block

It is used to inform the current configured collection instance of the basic information for auto-discovery. The field description is as follows:

```toml
[[inputs.kubernetesprometheus.instances]]
  # Used to inform the current collector instance of the type of workload for which auto-discovery needs to be performed. The allowed values are Node, Pod, Service, Endpoint
  role       = "pod"
  
  # Used to inform the current collector instance of the namespace of the workload for which auto-discovery needs to be performed. Leaving it empty means no need to monitor and discover by namespace
  namespaces = []
  
  # Used to inform the current collector instance of the workload with which tags need to be auto-discovered. For example, if your label is "app:nginx", set app=nginx to match this workload. Similarly, if the label is "abc:def", set abc=def as the matching condition.
  selector   = ""

  # Used to specify the workload auto-discovery strategy for the current collector instance. If configured as true here, regardless of the switch mark configured in the annotations of the workload later, this workload will be automatically discovered and collected
  scrape     = "true"
  
  # Used to mark the communication protocol for the collector instance to communicate with the workload and obtain metric data after auto-discovery. If configured as https here, the content of the instance-level authentication resource configuration block below needs to be configured at the same time
  scheme     = "http"
  
  # Used to inform the current collector instance of the port through which to obtain the metric data of the corresponding workload
  port       = "__kubernetes_pod_container_nginx_port_metrics_number"
  
  # Used to inform the current collector instance of the Path through which to read Prometheus metric data. The default value is /metrics, but for workloads that do not have built-in Prometheus collection and need to introduce Exporter Sidecar, this path may change. Please modify the value of this path according to the actual requirements of your workload.
  path       = "/metrics"
  
  # HTTP access parameters. This field allows you to add additional parameters to the metric acquisition path composed of port/path. If the Exporter you use or the built-in Prometheus metric interface of the workload allows carrying parameters, you can fill in the HTTP parameters here.
  params     = ""
```

##### Instance-level Request Header Configuration Block

If the workload targeted by the current configured collector instance requires carrying specified request headers when accessing the Prometheus metric API, the content of the request headers can be specified through this configuration block. If no specific request headers are needed, this configuration can be ignored.

##### Instance-level Custom Tag Block

It is used to configure customized tags, measurement names, and other content for the current collector instance, which will affect the storage and display of the collected data in the TrueWatch workspace. The field description is as follows:
```toml
[inputs.kubernetesprometheus.instances.custom]
  # Used to inform the current collector of the measurement name under which the data collected from the target workload configured above is stored. For example, if configured as pod-nginx here, it means that when the user views the data in the TrueWatch workspace, all metrics will be seen in the measurement named pod-nginx. You can specify any name for the measurement, but please ensure that it does not conflict with existing measurements, as this will cause data from different sources to be mixed together
  measurement        = "pod-nginx"
  
  # Controls whether to use the job label value in the collected metric data as the name of the measurement. When JobAsMeasurement is true, the measurement name directly uses the job label value in the data (for example, if job="node-exporter", the measurement name is node-exporter)
  # When JobAsMeasurement is false (default value), the measurement name is determined by the following rules:
  # 1. Priority is given to the name specified by the measurement configuration item.
  # 2. If measurement is not configured, split the metric field name by the first underscore _, and take the first field after splitting as the measurement name.
  job_as_measurement = false

  [inputs.kubernetesprometheus.instances.custom.tags]
    # Used to configure instance-level user-defined data tags for the current collector. Unlike global tags, the tags set here only take effect in the current collector instance.
    instance         = "__kubernetes_mate_instance"
    host             = "__kubernetes_mate_host"
    pod_name         = "__kubernetes_pod_name"
    pod_namespace    = "__kubernetes_pod_namespace"
    # You can continue to add other custom tags
```

##### Instance-level Authentication Resource Configuration Block

If the workload you want to auto-discover requires the collector to obtain monitoring data through the https protocol, this block must be configured. This configuration block will contain authentication and security-related information:
```toml
[inputs.kubernetesprometheus.instances.auth]
    # If bearer token authentication is used for https access, configure the token file path here, and the token needs to be mounted to your datakit container at the same time.
    # Please fill in the path according to the actual path where you mount the token. The path here is an example path
    bearer_token_file      = "/var/run/secrets/kubernetes.io/serviceaccount/token"
  
  # The following are the configuration of CA certificate and cert_key path, and the switch configuration of whether to skip SSL verification. Please replace the relevant certificate paths with the actual mount paths in datakit
  [inputs.kubernetesprometheus.instances.auth.tls_config]
    insecure_skip_verify = false
    ca_certs = ["/opt/nginx/ca.crt"]
    cert     = "/opt/nginx/peer.crt"
    cert_key = "/opt/nginx/peer.key"
```

Above, we have introduced the complete structure of the KubernetesPrometheus collector configuration document. Next, let's introduce the configuration steps according to the different modes you choose.

### 4.2 Prometheus Annotation Discovery Mode
Operation steps: Enable the Datakit KubernetesPrometheus collector and inject the configuration -> Add annotations to the workload.

As mentioned earlier, when adopting the Prometheus annotation discovery mechanism, you only need to make a few modifications to the global configuration of the collector to enable the datakit Kubernetesprometheus collector, and then add annotations to the workload to make auto-discovery take effect:

#### 4.2.1 Enable the Datakit KubernetesPrometheus Collector
Prepare a collector configuration for datakit and modify its content to complete the collector activation. A simplest example for configuration modification is as follows:

```toml
[inputs.kubernetesprometheus]
  enable_discovery_of_prometheus_pod_annotations = true
```

Combined with the preparation knowledge we introduced earlier, you should be able to see that this configuration only declares the activation of kubernetesprometheus and turns on the pod auto-discovery switch.

Add this configuration to the ConfigMap:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: datakit-conf
  namespace: datakit
data:
  kubernetesprometheus.conf: |-
    [inputs.kubernetesprometheus]
      enable_discovery_of_prometheus_pod_annotations = true
```
And mount it to the specified directory of datakit to complete the activation and configuration modification of the Datakit collector.

Of course, you can also continue to add to this main configuration to meet more complex configuration requirements. For example: if you need to specify the scraping cycle as 15 seconds and add a tag named "name=bob" to the metric data obtained after auto-discovery, you can set it like this:

```toml
[inputs.kubernetesprometheus]

  # Corresponding to the requirement of 15s scraping cycle interval
  scrape_interval = "15s"

  enable_discovery_of_prometheus_pod_annotations = true

  [inputs.kubernetesprometheus.global_tags]
    name = "bob"
```

Or on this basis, require Datakit to retain the original Prometheus measurement name:

```toml
[inputs.kubernetesprometheus]

  # Corresponding to the requirement of 15s scraping cycle interval
  scrape_interval = "15s"

  # Require Datakit to retain the original Prometheus measurement name  
  keep_exist_metric_name = true

  enable_discovery_of_prometheus_pod_annotations = true

  [inputs.kubernetesprometheus.global_tags]
    name = "bob"
```
After completing the modification, update the content to the configMap and complete the volumeMount for Datakit, and the new configuration will take effect. Please refer to the content of the collector main configuration in section 4.1.3 to modify or add/delete this part.

#### 4.2.2 Add Annotations to the Workload

If you view the example in the [Example](https://docs.truewatch.com/integrations/kubernetesprometheus/#auto-discovery-metrics-with-prometheus-example) section of the official documentation, you can see the workload configuration we use for the example:
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: ns-testing
  annotations:
    prometheus.io/scrape: "true"      # Enable metric collection
    prometheus.io/port: "80"          # Metrics port
    prometheus.io/scheme: "http"      # Protocol (optional)
    prometheus.io/path: "/metrics"    # Metrics path (optional)
    prometheus.io/param_measurement: "nginx_metrics"  # Measurement name (optional)
    prometheus.io/param_tags: "service_type=web,version=1.0"  # Custom tags (optional)
spec:
  selector:
    app.kubernetes.io/name: proxy
  ports:
  - name: name-of-service-port
    protocol: TCP
    port: 8080
    targetPort: http-web-svc
```

Please note the `annotations` field. Here, the fixed `key` value `"prometheus.io/*"` is used to inform the collector how to obtain the configuration information of the current workload during the auto-discovery process. These are all the tags currently supported by datakit auto-discovery. Users cannot add auto-discovery configuration items through custom methods. Please only use these 6 annotations to enable auto-discovery. The explanation of the fields is as follows:

| Annotation | Required | Default Value | Description |
| :--- | :--- | :--- | :--- |
| `prometheus.io/scrape` | Yes | - | Flag to configure whether auto-discovery is needed from the workload side |
| `prometheus.io/port` | Yes | - | The port number for metric collection of the current workload. This port number must exist in the port definition of the Pod Template, that is, this port is first a port accessible by the pod |
| `prometheus.io/scheme` | No | `http` | The protocol type for metric collection of the current workload: `http` or `https` |
| `prometheus.io/path` | No | `/metrics` | The metric endpoint path of the current workload, filled in according to the actual exposed metric path of the workload |
| `prometheus.io/param_measurement` | No | Auto-generated | Custom measurement name. This is an optional configuration item that allows workload responsible personnel to specify a preferred measurement name |
| `prometheus.io/param_tags` | No | - | Custom tags, string, content format: `tag1=value1,tag2=value2`. This is an optional configuration item that allows workload responsible personnel to specify additional data tags for metrics. |

Please note that for enabling service auto-discovery through the annotation method, in principle, it is necessary to fully configure the values of all the above annotation `key`s to ensure that the auto-discovery function can normally collect the metric data of the workload.

After editing these annotations and restarting the service, you can complete the activation of the auto-discovery function.

### 4.3 Collector Instance Configuration Mode
Operation steps: Enable the Datakit KubernetesPrometheus collector and inject the main configuration -> Add collector instance configuration according to the actual situation of the target object.

In this mode, which objects need to be auto-discovered and how to collect their metrics are specified through the collector instance configuration instead of reading the prometheus.io annotation fields. Therefore, in addition to the collector main configuration block and global tag block, we need to continue to configure each block of the collector instance to complete the activation of the auto-discovery function. The steps for enabling the main configuration will not be repeated here. Let's continue to introduce the incremental configuration part based on the main configuration in the previous chapter.

```toml
[inputs.kubernetesprometheus]

  # Corresponding to the requirement of 15s scraping cycle interval
  scrape_interval = "15s"

  # Require Datakit to retain the original Prometheus measurement name  
  keep_exist_metric_name = true

  enable_discovery_of_prometheus_pod_annotations = true

  [inputs.kubernetesprometheus.global_tags]
    name = "bob"
```

#### 4.3.1 Add Collector Instance Configuration

"According to the actual situation of the target object" in the operation steps means that before starting the collector instance configuration, we need to first understand the situation of the target workload for auto-discovery. For example, how do I identify the objects that need to be discovered, through which port to collect the metric data of the target object, what the Path of the collection API is, whether https access is required, whether the authentication certificate has been obtained, and whether it is necessary to add custom tags for it or read the K8s attribute fields of the object as additional data tags. When sufficient information is collected, we configure this information into the [[inputs.kubernetesprometheus.instances]] block to complete the collector instance configuration.

To better explain the above process, we assume a collection target scenario:
1. The workload to be monitored is Mysql, the auto-discovery target type is Service, the namespace for which discovery is expected is "my-demo", it needs to match target objects carrying the `app:prod_db` label, its query port is 19090, and the DBA responsible for maintenance has modified its default metric reporting path to `/db/metric`, using the HTTP protocol for access;
2. The collected data is reported to the measurement named "mysql_svc_metrics", and the service_name is added as an additional tag to the data for query and retrieval;

Based on this scenario, we copy the [[inputs.kubernetesprometheus.instances]] block and edit its content:

```toml
[[inputs.kubernetesprometheus.instances]]
    # Specify the auto-discovery load type as service
    role = "service"
    
    # Specify the namespace for auto-discovery
    namespaces = ["my-demo"]

    # The required matching application label is prod_db
    selector = "app=prod_db"

    # true means enabling collection
    scrape = "true"
    
    # Use HTTP protocol for collection
    scheme = "http"
    
    # Specify the metric query port as 19090
    port = 19090

    # Specify the metric query path
    path = "/db/metrics"
    
    # Custom measurement and tags
    [inputs.kubernetesprometheus.instances.custom]
      # Specify the measurement name for the collected metrics
      measurement = "mysql_svc_metrics"

      [inputs.kubernetesprometheus.instances.custom.tags]
        # Add the service name as a tag to the collected metric data. Here, placeholders are used to extract the value of service_name from the target object.
        service_name = "__kubernetes_service_name"
```

Integrate the edited collector instance configuration block with the main configuration block mentioned earlier to get the complete configuration:

```toml
[inputs.kubernetesprometheus]

  # Corresponding to the requirement of 15s scraping cycle interval
  scrape_interval = "15s"

  # Require Datakit to retain the original Prometheus measurement name  
  keep_exist_metric_name = true

  enable_discovery_of_prometheus_pod_annotations = true

  [inputs.kubernetesprometheus.global_tags]
    name = "bob"

  [[inputs.kubernetesprometheus.instances]]
    # Specify the auto-discovery load type as service
    role = "service"
    
    # Specify the namespace for auto-discovery
    namespaces = ["my-demo"]

    # The required matching application label is prod_db
    selector = "app=prod_db"

    # true means enabling collection
    scrape = "true"
    
    # Use HTTP protocol for collection
    scheme = "http"
    
    # Specify the metric query port as 19090
    port = 19090

    # Specify the metric query path
    path = "/db/metrics"
    
    # Custom measurement and tags
    [inputs.kubernetesprometheus.instances.custom]
      # Specify the measurement name for the collected metrics
      measurement = "mysql_svc_metrics"

      [inputs.kubernetesprometheus.instances.custom.tags]
        # Add the service name as a tag to the collected metric data. Here, placeholders are used to extract the value of service_name from the target object.
        service_name = "__kubernetes_service_name"
```

Mount this complete configuration block to the Datakit Daemonset through configMap. Subsequently, all objects from the my-demo namespace that meet the label and workload type requirements will be automatically discovered and their metric data will be obtained.

According to the above introduction, you should already be able to complete the activation and configuration of the Datakit auto-discovery mechanism. Please enable and use it in your environment.

#### 4.4 Introduction to Datakit Placeholders

In the above configuration examples, you should have noticed configurations similar to the following:

```toml
pod_namespace    = "__kubernetes_pod_namespace"
```

This specific assignment placeholder is a combination of placeholders set by datakit to facilitate obtaining workload metadata information in the configuration. In the official documentation of TrueWatch, there are detailed explanations of the available placeholder names and their corresponding value sources. You can obtain various metadata information of the workload for the collector through these placeholders, and set the values corresponding to these metadata information as tags or configuration item parameters in the configuration file. For the list of placeholders corresponding to all configuration objects currently supported by Datakit and their value introductions, please refer to the following links:

*   [Global Placeholders](https://docs.truewatch.com/integrations/kubernetesprometheus/#placeholders-global)
*   [Node](https://docs.truewatch.com/integrations/kubernetesprometheus/#placeholders-node)
*   [Pod](https://docs.truewatch.com/integrations/kubernetesprometheus/#placeholders-pod)
*   [Service](https://docs.truewatch.com/integrations/kubernetesprometheus/#placeholders-service)
*   [Endpoints](https://docs.truewatch.com/integrations/kubernetesprometheus/#placeholders-endpoints)

##### Explanation of the Placeholder Name Construction Method

In the above documentation, you may see placeholder examples in the following format:
```
__kubernetes_pod_container_%s_port_%s_number	
```

This means that before using the placeholder to obtain the corresponding workload information in the datakit configuration file, you need to first replace `%s` with the correct string with the help of the workload yaml. The method to obtain the correct content of the string is relatively simple. You need to check the value description of the current placeholder in the documentation:

```
.spec.containers[*].ports[*].containerPort ("name" equal "%s")
```

Then check the relevant fields of the corresponding workload template in your yaml. For pods, we need to check the fields in pod.template. Take the following yaml configuration as an example:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prom-server
  namespace: testing
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: prom
  replicas: 1
  template:
    metadata:
      labels:
        app.kubernetes.io/name: prom
    spec:
      containers:
      - name: testbox
        image: pubrepo.somewebsight.com/demo/busy-box:latest
        imagePullPolicy: IfNotPresent
        env:
        - name: ENV_PORT
          value: "30001"
        - name: ENV_NAME
          value: "promhttp"
        ports:
        - name: metrics
          containerPort: 30001
```
In this example, the `name` of `spec.containers` in the template is `testbox`, which means the first placeholder string is `testbox`. Then check that the `name` of the `port` is `metrics`, so the second placeholder string corresponds to `metrics`. That is to say, the full-format placeholder actually converts the JsonPath of the current resource in the Kubernetes yaml into snake_case naming:

`.spec.container['testbox'].ports[metric]`

Will be converted to

`__kubernetes_pod_container_testbox_port_metrics_number`

And obtain the value of the corresponding port.

Since this placeholder conversion involves the internal logic implementation of Datakit, if you encounter conversion problems during use, please contact our engineers to obtain placeholder examples to help you better understand the creation rules.

In addition, due to historical version issues, some datakit versions may have differences in name format support. If the name you configure contains a hyphen "-", please first configure it according to the actual name and enable the datakit collector. If you find that the collector's value is incorrect, please convert it to an underscore "_" according to Prometheus naming conventions and retry the collection.