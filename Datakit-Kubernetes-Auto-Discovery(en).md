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

## III. How to Enable Datakit Kubernetes Auto-discovery
The general configuration for Datakit to enable a collector is to first enable the corresponding collector and perform necessary parameter configuration for it, ensuring that the collector knows which objects' which addresses to access and how to access them to obtain necessary monitoring data. For the auto-discovery scenario discussed here, we need to enable Kubernetes auto-discovery in the following steps.

### 3.1 Disable the Default KubernetesPrometheus Collector
After you complete the installation of Datakit DaemonSet, if you use the `kubectl describe` command to check the Datakit DaemonSet and retrieve the `ENV_DEFAULT_ENABLED_INPUTS` keyword, you will see content similar to the following:

```
- name: ENV_DEFAULT_ENABLED_INPUTS
  value: statsd,dk,cpu,disk,diskio,mem,swap,system,hostobject,net,host_processes,container,kubernetesprometheus,ddtrace,profile
```

If the Datakit version you are using has `"kubernetesprometheus"` set in this environment variable, please delete it. This default collector flag will generate a set of default auto-discovery rules, which will overwrite the custom auto-discovery rules you configure later. Therefore, before we start, check this variable and delete the collector.

### 3.2 Kubernetes Service Auto-discovery Based on Custom Configuration
The steps to add a custom scraping configuration to enable a collector in Kubernetes are similar for all Datakit collectors: first, go to the official website to find the configuration file format of the collector, copy the format to your local editor, modify its content, inject the modified content into the cluster in the form of a configmap, and finally mount this configmap to the specified directory of datakit in the form of `volumeMount`. The relative path of the directory is usually described above the configuration file example in the document you are viewing. The absolute path starts with `/usr/local/datakit`.

#### 3.2.1 Determine the Mount Path
In this example, you first need to search for `KubernetesPrometheus` in the official documentation. In the [Example](https://docs.truewatch.com/integrations/kubernetesprometheus/#example) section of the opened document, you will see the mount location of the configuration file and the required file name:

```yaml
volumeMounts:
- mountPath: /usr/local/datakit/conf.d/kubernetesprometheus/kubernetesprometheus.conf
  name: datakit-conf
  subPath: kubernetesprometheus.conf
  readOnly: true
```

#### 3.2.2 Obtain the Configuration File Template
A complete `KubernetesPrometheus` configuration template is as follows. The example shows the format of the `configMap` mount content, and the content of the configuration template is after `"kubernetesprometheus.conf: |-"`:

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

      # Auto-discovery mode switches (mutually exclusive with instance configuration blocks)
      enable_discovery_of_prometheus_pod_annotations = false
      enable_discovery_of_prometheus_service_annotations = false
      enable_discovery_of_prometheus_pod_monitors = false
      enable_discovery_of_prometheus_service_monitors = false

      # Global label configuration, all scraping instances will append these labels
      [inputs.kubernetesprometheus.global_tags]
        instance = "__kubernetes_mate_instance"
        host     = "__kubernetes_mate_host"

      # Example instance configuration
      #[[inputs.kubernetesprometheus.instances]]
      #  # Target resource type: Node, Pod, Service, Endpoint
      #  role = "node"
      #  # Limit to specific namespaces, leave empty for all
      #  namespaces = []
      #  # Label selector, e.g., "app=nginx"
      #  selector = ""
      #  # Whether to force scraping, ignoring annotation switches
      #  scrape = "true"
      #  # Communication protocol: http, https
      #  scheme = "https"
      #  # Scraping port, can use placeholders
      #  port = "__kubernetes_node_kubelet_endpoint_port"
      #  # Metric path
      #  path = "/metrics"
      #
      #  # Custom request headers
      #  [inputs.kubernetesprometheus.instances.http_headers]
      #    # Authorization = "Bearer your-token-here"
      #
      #  # Custom measurement and tags
      #  [inputs.kubernetesprometheus.instances.custom]
      #    # Measurement name
      #    measurement = "kubernetes_node_metrics"
      #    # Whether to use the job label as the measurement name
      #    job_as_measurement = false
      #    [inputs.kubernetesprometheus.instances.custom.tags]
      #      node_name = "__kubernetes_node_name"
      #
      #  # Authentication configuration
      #  [inputs.kubernetesprometheus.instances.auth]
      #    bearer_token_file = "/var/run/secrets/kubernetes.io/serviceaccount/token"
      #    [inputs.kubernetesprometheus.instances.auth.tls_config]
      #      # Whether to skip SSL verification
      #      insecure_skip_verify = true
      #      ca_certs = []
      #      cert = ""
      #      cert_key = ""
```

#### 3.2.3 Modify the Configuration Template
The following explains how to modify this configuration template.

For the `KubernetesPrometheus` collector, the basic configuration structure is as follows:
```toml
[inputs.kubernetesprometheus]
  ## Collector basic configuration block
  ...

  [inputs.kubernetesprometheus.global_tags]
  ## Collector global tag configuration block
  ...

  [[inputs.kubernetesprometheus.instances]]
  ### Instance-level configuration block 1
  ...
  [[inputs.kubernetesprometheus.instances]]
  ### Instance-level configuration block 2
  ...
  [[inputs.kubernetesprometheus.instances]]
  ### Instance-level configuration block 3
  ...
```
Among them, `[inputs.kubernetesprometheus]` marks the enabling of the collector. Datakit determines whether to enable the collector by identifying this Toml block. Please note that there should be only one `[inputs.kubernetesprometheus]` mark in all your configuration files. If multiple `[inputs.kubernetesprometheus]` are used, the `KubernetesPrometheus` collector will fail to start, and only the sub-configuration in the first loaded `[inputs.kubernetesprometheus]` will take effect. The rest of the configurations will not take effect.

##### Collector Basic Configuration Block and Global Tag Configuration Block
For the collector basic configuration block and the collector global tag configuration block, the allowed configuration items include the following:
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

Please note that the collector basic configuration block and the global public configuration block are not mandatory. Leaving them empty means that you will use the default configuration values in the above figure for auto-discovery scraping, and the global tags of the collector are empty, indicating that no additional data collection tags are added.

##### Supplementary Explanation of the Auto-discovery Mechanism
You may have questions about the combination of "enable_*" flags in the collector basic configuration block, such as when to set them to true. It is necessary to supplement the explanation of the effective mechanism of Datakit auto-discovery here.

Datakit auto-discovery has two effective mechanisms. The first is the **"enable_*" tag group + workload annotations** to enable auto-discovery. In this mode, when configuring the auto-discovery function of the Datakit `KubernetesPrometheus` collector, you only need to modify the corresponding item in the following switches to `True` for the type of resource you need to discover:
```toml
enable_discovery_of_prometheus_pod_annotations = false
enable_discovery_of_prometheus_service_annotations = false
enable_discovery_of_prometheus_pod_monitors = false
enable_discovery_of_prometheus_service_monitors = false
```
After completing this modification, you do not need to continue with the subsequent collector instance configuration. Service auto-discovery can be achieved by directly adding annotations to the workload. The purpose of this is to be compatible with the Prometheus ecosystem. For example, in some environments, customers have already configured auto-discovery annotations for Prometheus Exporters and hope that Datakit can be directly compatible with this configuration to achieve auto-discovery without modifying the workload. It can also simplify the configuration operation of your new workloads, such as eliminating the need to maintain a lengthy datakit collector configuration table. However, the disadvantage is that the configuration content supported by annotations is limited, and it is only applicable to scenarios where metrics are obtained through simple http. In addition, in this mode, the annotations related to auto-discovery in the workload are mandatory. If no annotations are configured, the auto-discovery function may fail to take effect.

The second is to **disable the flag** (set to `false` or not configure it, whose default value is `false`), and use the configuration content in the collector instance configuration block for service auto-discovery. In this configuration, the annotations of the workload do not take effect. The collector instance will implement auto-discovery and add various tags to the metric data based on the parameters in the instance configuration block and other configurations introduced below. The advantage of this configuration is that the continuous integration team does not need to maintain annotations for the workload, and only needs to deploy pods using standard delivery templates. All configuration management work is concentrated on the datakit collector. Although the configuration file is relatively complex, it concentrates all configuration information. It is suitable for scenarios where centralized management of auto-discovery rules is required.

These two methods need to be chosen based on your own needs. If both the "enable_*" flags and the instance configuration block are enabled at the same time, it may lead to duplicate data collection and confusion in use. Below, we will first continue to take the collector instance configuration as an example to explain how to configure the auto-discovery collector. After introducing this part of the content, we will introduce how to enable auto-discovery through the annotation method.

##### Instance-level Configuration Block
The collector instance configuration block `[[inputs.kubernetesprometheus.instances]]` marks the configuration of a collector instance in a `KubernetesPrometheus` collector. This configuration can be repeated multiple times, indicating that you need to enable multiple collection instances corresponding to different auto-discovery objects or collection configurations.

Assuming that in your environment, you need to configure auto-discovery configurations for Nginx, Redis, and MySQL instances at the same time, the basic configuration structure is:

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

Next, we will continue to introduce the detailed configuration content of the instance-level configuration block.

#### 3.2.4 Detailed Configuration Introduction of the Instance-level Configuration Block
For each collector instance, a general custom configuration template is in the [Configuration Description](https://docs.truewatch.com/integrations/kubernetesprometheus/#input-config-added) section, and its format is as follows:

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

Now, the configuration template is explained.

Each `[[inputs.kubernetesprometheus.instances]]` mainly includes the following parts:
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
  
  # http access parameters. This field allows you to add additional parameters to the metric acquisition path composed of port/path. If the Exporter you use or the built-in Prometheus metric interface of the workload allows carrying parameters, you can fill in the http parameters here.
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
  
  # The following are the configuration of ca certificate and cert_key path, and the switch configuration of whether to skip ssl verification. Please replace the relevant certificate paths with the actual mount paths in datakit
  [inputs.kubernetesprometheus.instances.auth.tls_config]
    insecure_skip_verify = false
    ca_certs = ["/opt/nginx/ca.crt"]
    cert     = "/opt/nginx/peer.crt"
    cert_key = "/opt/nginx/peer.key"
```

After completing the above `KubernetesPrometheus` collector instance configuration and mounting it to the specified directory of Datakit, the auto-discovery function of Datakit can be enabled.

#### 3.2.5 Datakit Placeholders
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

### 3.3 Service Auto-discovery Based on "enable_*" Tag Group + Workload Annotations
In the previous introduction, we introduced how to enable the auto-discovery function by modifying the Datakit configuration block, including the hierarchy, content, and role of the configuration block. Next, we will introduce another auto-discovery enabling method introduced in <Supplementary Explanation of the Auto-discovery Mechanism>, that is, enabling service auto-discovery through the `enable_*` flag plus annotations.

#### 3.3.1 Workload Annotation Rules
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

Please note the `annotations` field. Here, the fixed `key` value `"prometheus.io/*"` is used to inform the collector how to obtain the configuration information of the current workload during the auto-discovery process. These are all the tags currently supported by datakit auto-discovery. Users cannot add auto-discovery configuration items through custom methods. Please only use these 6 annotations for auto-discovery configuration. The explanation of the fields is as follows:

| Annotation | Required | Default Value | Description |
| :--- | :--- | :--- | :--- |
| `prometheus.io/scrape` | Yes | - | Flag to configure whether auto-discovery is needed from the workload side |
| `prometheus.io/port` | Yes | - | The port number for metric collection of the current workload. This port number must exist in the port definition of the Pod Template, that is, this port is first a port accessible by the pod |
| `prometheus.io/scheme` | No | `http` | The protocol type for metric collection of the current workload: `http` or `https` |
| `prometheus.io/path` | No | `/metrics` | The metric endpoint path of the current workload, filled in according to the actual exposed metric path of the workload |
| `prometheus.io/param_measurement` | No | Auto-generated | Custom measurement name. This is an optional configuration item that allows workload responsible personnel to specify a preferred measurement name |
| `prometheus.io/param_tags` | No | - | Custom tags, string, content format: `tag1=value1,tag2=value2`. This is an optional configuration item that allows workload responsible personnel to specify additional data tags for metrics. |

Please note that for enabling service auto-discovery through the annotation method, in principle, it is necessary to fully configure the values of all the above annotation `key`s to ensure that the auto-discovery function can normally collect the metric data of the workload.

#### 3.3.2 Datakit Collector Configuration
For service auto-discovery enabled through the annotation method, the configuration of the datakit collector becomes more concise. You only need to enable the corresponding `enable_*` switch according to the type of workload that needs to be auto-discovered, without adding other additional instance configuration blocks. For example, if you need to auto-discover Pod-type workloads, a configuration example is as follows:

```toml
[inputs.kubernetesprometheus]
  # Other configurations are configured as needed or deleted to use default values
  # node_local = true
  # scrape_interval = "30s"
  # keep_exist_metric_name = true
  # honor_timestamps = true

  enable_discovery_of_prometheus_pod_annotations = true
```

After completing the configuration, copy this configuration content to the `configMap` and mount it to the Datakit directory to enable collection. As we introduced earlier, this configuration method is very concise and does not require maintaining a large number of collector instance configurations. However, it is only applicable to simple `http` collection. If the workload requires `https` access, or the configuration needs to dynamically obtain pod metadata as data tags, or it is necessary to centrally maintain auto-discovery configurations to avoid the workload caused by large-scale workload annotation management, it is recommended to use the collector instance configuration method to implement service auto-discovery.

---

## Operation Examples

### Scenario 1
I have a workload that previously used Prometheus for auto-discovery and now want to integrate it into TrueWatch. I have deployed Datakit in the cluster. How to enable service auto-discovery?

**Configuration Steps:**

a. Prepare the following configuration content:
```toml
[inputs.kubernetesprometheus]
  enable_discovery_of_prometheus_pod_annotations = true
```

b. Add the corresponding `configMap` for it:
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

c. Modify the Datakit Daemonset and add the mount of this `configmap`:
```yaml
volumeMounts:
- mountPath: /usr/local/datakit/conf.d/kubernetesprometheus/kubernetesprometheus.conf
  name: datakit-conf
  subPath: kubernetesprometheus.conf
  readOnly: true
```
This configuration enables automatic collection.

### Scenario 2
I want to perform service auto-discovery and metric collection for all mysql instances in the workspace and hope to centrally manage the auto-discovery configuration;
I want the interval of the metric collection cycle to be 15s, and append the service name as a data tag to the collected metrics to facilitate subsequent query or aggregation;
I want to collect data based on svc to avoid metric collection interruptions caused by pod lifecycle changes;
The svc service label I configured for MySQL is `app:prod_db`, and the Exporter Sidecar has been installed. The metric query port is 19090, the name corresponding to this port configuration is mysqlexp, and the query path is `/db/metric`;

Based on the above conditions, how to enable auto-discovery in Datakit?

a. Prepare the following configuration content:
```toml
[inputs.kubernetesprometheus]
  # Enable NodeLocal mode to distribute collection pressure
  node_local = true
  
  # Corresponding to the requirement of 15s collection cycle interval
  scrape_interval = "15s"

  # This is the global tag configuration. If not needed, comment it out. In this scenario, the pod tag is only for mysql auto-discovery
  # Do not add it as a global tag, which will cause other tags added later to have the service_name tag, and some collectors may not need this tag
  # [inputs.kubernetesprometheus.global_tags]
  #   service_name = "__kubernetes_service_name"

  # Define a collection instance
  [[inputs.kubernetesprometheus.instances]]
    # Specify the auto-discovery load type as service
    role = "service"
    
    # No namespace is limited in the scenario requirements, so leave it empty here. If the namespace is limited, add the specified namespace to the array, separated by commas
    namespaces = []

    # The required matching application label is prod_db
    selector = "app=prod_db"

    # true means enabling collection
    scrape = "true"
    
    # Use http protocol for collection
    scheme = "http"

    # Dynamically obtain the port number according to the yaml configuration of the svc. Here, use the name of the service port to complete the placeholder assembly
    # For specific methods, refer to the introduction in section 3.2.4
    # port = "__kubernetes_service_port_mysqlexp_targetport"
    
    # Or set it directly to 19090 according to the scenario conditions
    port = 19090

    # Modify the value here according to the Prometheus metric path provided by the workload to enable auto-discovery to collect metrics through the correct path
    path = "/db/metrics"
    
    # Custom measurement and tags
    [inputs.kubernetesprometheus.instances.custom]
      # Specify the measurement name for the collected metrics
      measurement = "mysql_svc_metrics"

      [inputs.kubernetesprometheus.instances.custom.tags]
        # Append the service name as a tag to the collected metric data
        service_name = "__kubernetes_service_name"
```

b. Add this configuration to the `configMap` and mount it to the Datakit Daemonset
The steps are the same as above.