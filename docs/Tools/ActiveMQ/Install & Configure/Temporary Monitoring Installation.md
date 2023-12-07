This is a guide on how to setup a temporary Metricbeat monitoring installation for monitoring ActiveMQ. It is a temporary broker, but we a little modification to the guide can become a permanent solution.

This page was originally created because of the shutdown in Oregon on August 2023 where the entire datacenters (JF4 & JFS1) were down for a week and a half and we had to setup ActiveMQ in another site an FAST with monitoring.

1. SSH Into the ActiveMQ host.
2. Pull Docker image

```shell
docker pull docker.elastic.co/beats/metricbeat:7.14.1
```


```shell
docker run -d --rm \
  --name=metricbeat \
  --user=root \
  --network=host \
  --volume="$(pwd)/metricbeat.docker.yml:/usr/share/metricbeat/metricbeat.yml:ro" \
  --volume="/var/run/docker.sock:/var/run/docker.sock:ro" \
  --volume="/sys/fs/cgroup:/hostfs/sys/fs/cgroup:ro" \
  --volume="/proc:/hostfs/proc:ro" \
  --volume="/:/hostfs:ro" \
  docker.elastic.co/beats/metricbeat:7.14.1 && docker logs metricbeat -t -f
```

Check the logs of the ActiveMQ broker for errors using the command: 

```shell
docker logs metricbeat -t -f
```

To stop the ActiveMQ broker:

```
docker stop metricbeat
```

To restart the ActiveMQ broker:

```
docker restart metricbeat
```

To exec the ActiveMQ broker:

```
docker exec -it metricbeat /bin/bash
```

Changed Config fields:

```yml
metricbeat.config.modules:
  reload.enabled: true
```

Under the general section add `fields`:
```yml
# ================================== General ===================================
fields:
  source: metricbeat
  kafka_topic: daas-beats
  site: IIL
  app: iBI
```

```yml
# ------------------------------ Kafka Output -------------------------------
output.kafka:
  # initial brokers for reading cluster metadata
  hosts: ["libi-kafka.iil.intel.com:9092"]

  # message topic selection + partitioning
  topic:  '%{[fields.kafka_topic]}'
```
## IIL Config
```yml
###################### Metricbeat Configuration Example #######################

# This file is an example configuration file highlighting only the most common
# options. The metricbeat.reference.yml file from the same directory contains all the
# supported options with more comments. You can use it as a reference.
#
# You can find the full configuration reference here:
# https://www.elastic.co/guide/en/beats/metricbeat/index.html

# =========================== Modules configuration ============================

metricbeat.config.modules:
  # Glob pattern for configuration loading
  path: ${path.config}/modules.d/*.yml

  # Set to true to enable config reloading
  reload.enabled: false

  # Period on which files under path should be checked for changes
  #reload.period: 10s

# ======================= Elasticsearch template setting =======================

setup.template.settings:
  index.number_of_shards: 1
  index.codec: best_compression
  #_source.enabled: false


# ================================== General ===================================

# The name of the shipper that publishes the network data. It can be used to group
# all the transactions sent by a single shipper in the web interface.
#name:

# The tags of the shipper are included in their own field with each
# transaction published.
#tags: ["service-X", "web-tier"]

# Optional fields that you can specify to add additional information to the
# output.
#fields:
#  env: staging
fields:
  source: metricbeat
  kafka_topic: daas-beats
  site: IIL
  app: iBI

# ================================= Dashboards =================================
# These settings control loading the sample dashboards to the Kibana index. Loading
# the dashboards is disabled by default and can be enabled either by setting the
# options here or by using the `setup` command.
#setup.dashboards.enabled: false

# The URL from where to download the dashboards archive. By default this URL
# has a value which is computed based on the Beat name and version. For released
# versions, this URL points to the dashboard archive on the artifacts.elastic.co
# website.
#setup.dashboards.url:

# =================================== Kibana ===================================

# Starting with Beats version 6.0.0, the dashboards are loaded via the Kibana API.
# This requires a Kibana endpoint configuration.
setup.kibana:

  # Kibana Host
  # Scheme and port can be left out and will be set to the default (http and 5601)
  # In case you specify and additional path, the scheme is required: http://localhost:5601/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
  #host: "localhost:5601"

  # Kibana Space ID
  # ID of the Kibana Space into which the dashboards should be loaded. By default,
  # the Default Space will be used.
  #space.id:

# =============================== Elastic Cloud ================================

# These settings simplify using Metricbeat with the Elastic Cloud (https://cloud.elastic.co/).

# The cloud.id setting overwrites the `output.elasticsearch.hosts` and
# `setup.kibana.host` options.
# You can find the `cloud.id` in the Elastic Cloud web UI.
#cloud.id:

# The cloud.auth setting overwrites the `output.elasticsearch.username` and
# `output.elasticsearch.password` settings. The format is `<user>:<pass>`.
#cloud.auth:

# ================================== Outputs ===================================

# Configure what output to use when sending the data collected by the beat.
# ------------------------------ Kafka Output -------------------------------
output.kafka:
  # initial brokers for reading cluster metadata
  hosts: ["ibi-kafka.ibicloud.intel.com:9092"]

  # message topic selection + partitioning
  topic:  '%{[fields.kafka_topic]}'
  partition.round_robin:
    reachable_only: false

  required_acks: 1
  compression: gzip
  max_message_bytes: 1000000

# ---------------------------- Elasticsearch Output ----------------------------
#output.elasticsearch:
  # Array of hosts to connect to.
  # hosts: ["localhost:9200"]

  # Protocol - either `http` (default) or `https`.
  #protocol: "https"

  # Authentication credentials - either API key or username/password.
  #api_key: "id:api_key"
  #username: "elastic"
  #password: "changeme"

# ------------------------------ Logstash Output -------------------------------
#output.logstash:
  # The Logstash hosts
  #hosts: ["localhost:5044"]

  # Optional SSL. By default is off.
  # List of root certificates for HTTPS server verifications
  #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]

  # Certificate for SSL client authentication
  #ssl.certificate: "/etc/pki/client/cert.pem"

  # Client Certificate Key
  #ssl.key: "/etc/pki/client/cert.key"

# ================================= Processors =================================

# Configure processors to enhance or manipulate events generated by the beat.

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~


# ================================== Logging ===================================

# Sets log level. The default log level is info.
# Available log levels are: error, warning, info, debug
#logging.level: debug

# At debug level, you can selectively enable logging only for some components.
# To enable all selectors use ["*"]. Examples of other selectors are "beat",
# "publisher", "service".
#logging.selectors: ["*"]

# ============================= X-Pack Monitoring ==============================
# Metricbeat can export internal metrics to a central Elasticsearch monitoring
# cluster.  This requires xpack monitoring to be enabled in Elasticsearch.  The
# reporting is disabled by default.

# Set to true to enable the monitoring reporter.
#monitoring.enabled: false

# Sets the UUID of the Elasticsearch cluster under which monitoring data for this
# Metricbeat instance will appear in the Stack Monitoring UI. If output.elasticsearch
# is enabled, the UUID is derived from the Elasticsearch cluster referenced by output.elasticsearch.
#monitoring.cluster_uuid:

# Uncomment to send the metrics to Elasticsearch. Most settings from the
# Elasticsearch output are accepted here as well.
# Note that the settings should point to your Elasticsearch *monitoring* cluster.
# Any setting that is not set is automatically inherited from the Elasticsearch
# output configuration, so if you have the Elasticsearch output configured such
# that it is pointing to your Elasticsearch monitoring cluster, you can simply
# uncomment the following line.
#monitoring.elasticsearch:

# ============================== Instrumentation ===============================

# Instrumentation support for the metricbeat.
#instrumentation:
    # Set to true to enable instrumentation of metricbeat.
    #enabled: false

    # Environment in which metricbeat is running on (eg: staging, production, etc.)
    #environment: ""

    # APM Server hosts to report instrumentation results to.
    #hosts:
    #  - http://localhost:8200

    # API Key for the APM Server(s).
    # If api_key is set then secret_token will be ignored.
    #api_key:

    # Secret token for the APM Server(s).
    #secret_token:


# ================================= Migration ==================================

# This allows to enable 6.7 migration aliases
#migration.6_to_7.enabled: true
```

## Folsom (FM) Config

```yml
###################### Metricbeat Configuration Example #######################
# =========================== Modules configuration ============================

metricbeat.config.modules:
  # Glob pattern for configuration loading
  path: ${path.config}/modules.d/*.yml

  # Set to true to enable config reloading
  reload.enabled: false

  # Period on which files under path should be checked for changes
  #reload.period: 10s

# ======================= Elasticsearch template setting =======================

setup.template.settings:
  index.number_of_shards: 1
  index.codec: best_compression
  #_source.enabled: false

# ================================== General ===================================

# The name of the shipper that publishes the network data. It can be used to group
# all the transactions sent by a single shipper in the web interface.
#name:

# The tags of the shipper are included in their own field with each
# transaction published.
#tags: ["service-X", "web-tier"]

# Optional fields that you can specify to add additional information to the
# output.
#fields:
#  env: staging
fields:
  source: metricbeat
  kafka_topic: daas-beats
  site: FM
  app: iBI
# ================================== Outputs ===================================

# Configure what output to use when sending the data collected by the beat.
# ------------------------------ Kafka Output -------------------------------
output.kafka:
  # initial brokers for reading cluster metadata
  hosts: ["ibi-kafka.ibicloud.intel.com:9092"]

  # message topic selection + partitioning
  topic:  '%{[fields.kafka_topic]}'
  partition.round_robin:
    reachable_only: false

  required_acks: 1
  compression: gzip
  max_message_bytes: 1000000

# ================================= Processors =================================

# Configure processors to enhance or manipulate events generated by the beat.

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~
```

## ActiveMQ Module

```
jpico $(pwd)/metricbeat.activemq.module.docker.yml
```

```yml
# Module: activemq
# Docs: https://www.elastic.co/guide/en/beats/metricbeat/7.x/metricbeat-module-activemq.html

- module: activemq
  metricsets: ['broker', 'queue', 'topic']
  period: 10s
  hosts: ['localhost:8161']
  path: '/api/jolokia/?ignoreErrors=true&canonicalNaming=false'
  username: admin # default username
  password: admin # default password
```