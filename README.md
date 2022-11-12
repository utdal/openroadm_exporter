# OpenROADM Exporter
A Prometheus exporter that collects metrics from OpenROADM devices using SSH NETCONF sessions and exposes them via HTTP, ready for collection by Prometheus.

## Getting Started
Start openroadm_exporter with a valid configuration file using the --config.path flag. To then collect the metrics of a device, pass the 'config' and 'target' parameter to the exporter's web interface. For example, http://exporter:9930/metrics?config=default&target=192.168.1.1.

Promethues configuraiton:
```
scrape_configs:
  - job_name: openroadm
    static_configs:
      - targets:
        - device1
    params:
      config: ['default']
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: junos_exporter:9930  # OpenROADM exporter's address and port.
```

Docker:
```
docker run --restart unless-stopped -d -p 9930:9930 -v /home/user/.ssh/ssh_key:/ssh_key  -v /home/user/config.yaml:/config.yaml utdal/openroadm_exporter
```
The above Docker commands assumes a configuration file that specifies the SSK key as /ssh_key is located locally in /home/user/config.yaml.

## Configuration file
OpenROADM Exporter requires a configuration file in the below format:
```
configs:
  default:                       # Name of the configuration
    timeout:                     # SSH Timeout in seconds. Optional.
    username:                    # SSH Username. Required.
    password:                    # SSH Password. Optional.    
    ssh_key:                     # SSH Key. Optional.        
    allowed_targets:             # List of targets that can be collected. Optional.
      -          
    enabled_collectors:          # Which collectors to enable. Required.
      - interface
      - environment
      - power
    interface_description_keys:   # List of JSON keys in the interface description to include as labels in the 'interface_description' metric. Optional.
      - 
    interface_metric_keys:        # List of JSON keys in the interface description to create static metrics from. Optional.
      -
global:
  timeout:                       # SSH Timeout in seconds, globally configured. Optional.
  allowed_targets:               # List of targets that can be collected, globally configured. Optional.
    -
  interface_description_keys:    # List of JSON keys in the interface description to include as labels in the 'interface_description' metric, globally configured. Optional.
      - 
  interface_metric_keys:         # List of JSON keys in the interface description to create static metrics from, globally configured. Optional.
```
### Example
```
configs:
  default:
    timeout: 30
    username: openroadm
    password: openroadm
    #ssh_key: ~/key.pem
    allowed_targets:
      - 10.1.0.0
    enabled_collectors:
      - interface
global:
  timeout: 30
  allowed_targets:
   - 10.0.0.0

```
### configs
Each configuration is called by passing the 'config' parameter to the exporter's export web interface. In the above example, to use the default config you would use http://exporter:9930/metrics?config=default and to use the bgp_only config, you would use http://exporter:9930/metrics?config=bgp_only.

### allowed_targets
If allowed_targets is specified, only those targets may be collected. This is a form of security that stops a malicious user trying to collect details, such as the username and password, by specifying a target they control.

### global
Global applies to all configs, where that configuration item has not already been set under a specific config.

## Metrics
The below metrics are currently implemented.
- Interface Statistics, from `show interface extensive`.
- BGP, from `show bgp summary`.
- Environment, from `show chassis environment`.
- Power, from `show chassis power detail`.
- Route Engine, from `show chassis routing-engine`.
- IPsec, from `show security ipsec security-associations` and `show security ipsec inactive-tunnels`.
- Optics, from `show interface diagnostics optics`

### BGP: junos_bgp_peer_types_up
Junos Exporter exposes a special metric, `junos_bgp_peer_types_up`, that can be used in scenarios where you want to create Prometheus queries that report on the number of types of BGP peers that are currently established, such as for Alertmanager. To implement this metric, a JSON formatted description must be configured on your BGP group. Junos Exporter will then use the value from the keys specific under the `bgp_peer_type_keys` configuration, and aggregate all BGP peers that are currently established and configured with that type.

For example, if you want to know how many BGP peers are currently established that provide internet, you'd set the description of all BGP groups that provide internet to `{"type":"internet"}` and query Prometheus with `junos_bgp_peer_types_up{type="internet"})`. Going further, if you want to create an alert when the number of established BGP peers that provide internet is 1 or less, you'd use `sum(junos_bgp_peer_types_up{type="internet"}) <= 1`.  

Example Junos configuration:
```
set protocols bgp group internet-provider1 description "{\"type\":\"internet\"}"
set protocols bgp group internet-provider2 description "{\"type\":\"internet\"}"
```

### Interface: Interface Description Metric
An interface description metric with a value of "1" and labels with values defined within the interface description can be generated by specifying `interface_description_keys` in the config or global section of the configuration. This tells the exporter to look at the JSON formatted description of all interfaces, extract the value of the specified key, and add it as a label to a metric named `junos_interface_description` that has a value of 1. This is useful when automating Prometheus alert rules or Dashboards. For example, the Prometheus query `sum(junos_interface_input_bps) * on (instance,interface) group_left(type) junos_interface_description{type="internet"}) > 10000` can be used to create a rule that triggers when the combined BPS of all internet interfaces is above 10000.

Example Junos configuration:
```
set interfaces ge-0/0/0 description "{\"type\":\"internet\"}"
set interfaces ge-0/0/1 description "{\"type\":\"internet\"}"
```

### Interface: Static Metrics from Interface Description Keys
It's also possible to use interface descriptions to generate individual static metrics by specifying `interface_metric_keys` in the config or global section of the configuration. This tells the exporter to look at the JSON formatted description of all interfaces, creating a new metric from each specified key with a name and value equal to the key/value in the interface description. Like the interface description metric, this is useful when automating Prometheus alert rules or Dashboards. For example, building on the previous example, the Prometheus query `sum(junos_interface_input_bps{"type"="internet"}) / sum(junos_interface_commit_bw * on (instance,interface) group_left(type) junos_interface_description{type="internet"}) > .8` can be used to create a rule that triggers when the combined utilization of all internet interfaces is greater than 80% of the specified commited bandwidth. For the example config below, this query would return true if the combined utilization was above 12Gbps (one interface with a committed bandwidth of 5Gbps, one with a committed bandwidth of 10Gbps).

Example Junos configuration:
```
set interfaces ge-0/0/0 description "{\"type\":\"internet\","\commit_bw\":\"5000000000\"}"
set interfaces ge-0/0/1 description "{\"type\":\"internet\",,"\commit_bw\":\"10000000000\"}"
```

## Development
### Building
```
go get github.com/utdal/openroadm_exporter
cd ${GOPATH}/src/github.com/utdal/openroadm_exporter
go build
```

### NETCONF Output
XML was chosen as the output format of NETCONF commands for the below reasons:

 - Junos devices return XML faster than any other format, more than half the time it takes fora JSON response. Presumedly this is because XML is the native configuration format of Junos.
 - It is only possible to use NETCONF filter tags when the output is XML.
