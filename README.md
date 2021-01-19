# Netdata

[![Build Status](https://drone.osshelp.ru/api/badges/ansible/netdata/status.svg)](https://drone.osshelp.ru/ansible/netdata)

The role installs Netdata and setups it configuration.

## Deploy example (with all default parametrs)

```yaml
  roles:
    - role: netdata
      web_lob_plugin_enabled: false
```

Make sure that you call this role **before** the [monit](https://gitea.osshelp.ru/ansible/monit) role, if used together. Otherwise netdata presence will not be automatically detected and Monit cfg generation for it will be skipped.

## Configure only

Changing configuration without installation. For v2+ base images or based on those images only.

```yaml
  roles:
    - role: netdata
      netdata_setup: configure
```

When using legacy images -- make sure to set `apikey` variable manually:

```yaml
  roles:
    - role: netdata
      apikey: 11111111-2222-3333-4444-555555555555
      netdata_setup: configure
```

See detailed explanation about API key setting below.

## Important parameters

| Param | Default | Description |
| -------- | -------- | -------- |
| `netdata_setup` | `full` | Setup mode. See [OSSHelp KB article](https://oss.help/kb4895) |
| `force_install` | `false` | Used for reinstall or update (for official) |
| `physical` | `false` | Used for physical servers (for official) |
| `netdata_enable_app_plugin_in_lxc` | `false` | Used for enable app plugin in lxc |
| `netdata_force_create_db_user` | `false` | Used for enable creating users in MySQL, PostreSQL, RabbitMQ, MongoDB |
| `netdata_force_all_plugins_installation` | `false` | For role builds only! Using this parameter in production builds could cause unforeseen consequences. Forces role to configure all plugins, even if their services are missing |
| `netdata_force_host_system` | `false` | For role builds only! Using this parameter in production builds could cause unforeseen consequences. Forces role to act like if being deployed to host system (even in lxc-containers) |

### Restart timeout control

| Param | Default | Description |
| -------- | -------- | -------- |
| `netdata_timeout_stop_sec` | `10` | Default timeout in seconds |
| `netdata_dynamic_stop_timeout` | `true` | Whether to calculate timeout from `netdata_dbengine_multihost_disk_space` value |

When deploying to host system with `netdata_memory_mode` param set to `dbengine`, the calculation of `TimeoutStopSec` parameter in systemd unit override is performed automatically, depending on the value of `netdata_dbengine_multihost_disk_space`. The formula is:

```plain
netdata_timeout_stop_sec + netdata_dbengine_multihost_disk_space / 100
```

The automatic generation can be disabled entirely by switching `netdata_dynamic_stop_timeout` to `false` and overriding `netdata_timeout_stop_sec` with a value you need.

## Another deploy example

```yaml
  roles:
    - role: netdata:
      force_monit_plugin_installation: true
      web_lob_plugin_enabled: true
      plugin_retries: 300
      plugin_autodetection_retry: 300
      mysql_socket_path: '/var/run/mysqld/mysqld.sock'
      nginx_stat_url: 'http://localhost/nginx_status'
      apache_stat_url: 'http://localhost:8080/server-status?auto'
      fpm_stat_url: "http://localhost/fpm-status-{{pool_name}}?full&json"
      monit_url: 'http://localhost:2812'
      memcached_port: 11211
      haproxy_user: user
      haproxy_pass: PaSSw0rd
      haproxy_url: 'http://127.0.0.1:9000/haproxy;csv;norefresh'
      haproxy_config_path: '/etc/haproxy/haproxy.cfg'
      fping_hosts: '8.8.8.8 8.8.4.4'
      smartd_exclude_drives: some_drive
      tomcat_service_name: tomcat
      tomcat_stat_url: 'http://localhost:8285/manager/status?XML=true'
      tomcat_user: monitor
      tomcat_pass: paSSw0rd
      tomcat_connector: http-bio-8285
      phpfpm_pools:
        - prod
        - dev
      general_fpm_url_part: 'http://localhost/fpm-status'
      web_log_params:
        name: 'example.com'
        path: '/var/log/nginx/access.log'
        histogram: '50,100,200,500,1000,2000,5000'
        pattern: '(?P<address>[\da-f.:]+).*(?P<date>\[.+\])\s(?P<code>[1-9]\d{2})\s\".*\"\s\"(?P<method>[A-Z]+)\s(?P<url>.*?)\s.+\"\s(?P<  bytes_sent>\d+)\s(?P<resp_time>\d+\.\d+)'
      web_log_categories:
        - { key: main, value: '^/$' }
        - { key: admin, value: '^/(\w{2}\/)?admin/' }
```

## Multiple logs support in web_log

``` yaml
- role: netdata
  web_log_params:
    - name: logfile1
      path: '/var/log/nginx/logfile1.log'
      histogram: '50,100,200,500,1000,2000,5000'
      pattern: '(?P<address>[\da-f.:]+).*(?P<date>\[.+\])\s(?P<code>[1-9]\d{2})\s\".*\"\s\"(?P<method>[A-Z]+)\s(?P<url>.*?)\s.+\"\s(?P<bytes_sent>\d+)\s(?P<resp_time>\d+\.\d+)'
      categories:
        - { key: main, value: '^/$' }
        - { key: cat1, value: '^/cat1/' }
        - { key: cat2, value: '^/cat2/' }
    - name: logfile2
      path: '/var/log/nginx/logfile2.log'
      categories:
        - { key: main, value: '^/$' }
        - { key: cat1, value: '^/cat1/' }
        - { key: cat2, value: '^/cat2/' }
```

Defaults:

- histogram. `50,100,200,500,1000,2000,5000`
- pattern. `(?P<address>[\da-f.:]+).*(?P<date>\[.+\])\s(?P<code>[1-9]\d{2})\s\".*\"\s\"(?P<method>[A-Z]+)\s(?P<url>.*?)\s.+\"\s(?P<bytes_sent>\d+)\s(?P<resp_time>\d+\.\d+)`

## Deploy example with custom general params

``` yaml
- role: netdata
  netdata_memory_mode: dbengine
  netdata_page_cache_size: 32
  netdata_dbengine_multihost_disk_space: 4096
```

You can calculate the required system resources using the [official calculator](https://learn.netdata.cloud/docs/store/change-metrics-storage#calculate-the-system-resources-ram-disk-space-needed-to-store-metrics).

## Deploy example with fixed Netdata version

``` yaml
- role: netdata
  netdata_version: '1.23.0'
```

## Prometheus endpoint collector

Netdata can gather metrics from Prometheus endpoints and generate new charts with the same high-granularity, per-second frequency as you expect from other collectors. (Only xenial and bionic support in role) See [docs](https://learn.netdata.cloud/docs/agent/collectors/go.d.plugin/modules/prometheus) for more information.

### Deploy example with node_exporter job and custom global params

``` yaml
- role: netdata
  netdata_prometheus_params:
    autodetection_retry: 150
    max_time_series: 2048
    update_every: 15
  netdata_prometheus_jobs:
    - name: node_exporter_local
      url: 'http://127.0.0.1:9100/metrics'
```

### Collecting metrics from Dnsmasq-exporter

Port 9153/tcp, that is being used by dnsmasq-exporter, is already occupied by CoreDNS (see [Default port allocations](https://github.com/prometheus/prometheus/wiki/Default-port-allocations)) so default configuration won't fit. For metric collection you need to override the default job list with `netdata_prometheus_jobs`, e.g.:

``` yaml
  netdata_prometheus_jobs:
    - name: dnsmasq_exporter_local
      url: 'http://127.0.0.1:9153/metrics'
```

### Collectiong metrics from script-exporter

For now only exporter's metrics will be collected with default configuration, so no scripts will be executed at all. You can try to prepare a suitable job, but make sure to set the appropriate `update_every` value or provide your script with metric-caching mechanisms.

## About the API key setting

If `apikey` variable is empty (it is by default) the role will search for environment variable `NETDATA_API_KEY` and try to use it's value. `NETDATA_API_KEY` must be set in you test/build plugins' environment in your drone.yml from orgsecret `netdata_api_key` to be available for role. If by any means this is not possible - you can always override this behaviour by pinning `apikey` in role params, so `NETDATA_API_KEY` will be never checked.

### Trick for role testing on workspace

`NETDATA_API_KEY` is being passed to environment in provisioner section of molecule.yml where needed (non-configure scenarios). Without it builds with molecule will fail when executed on workspace.

## Useful links

- [Official documentation](https://learn.netdata.cloud/docs/agent)
- [OSSHelp KB Articles](https://oss.help/kbg114)

## TODO

- add check mode support

## License

GPL3

## Author

OSSHelp Team, see <https://oss.help>
