![picture alt](https://cdn.rawgit.com/fabric8io/fabric8-devops/93ca9bc/prometheus/src/main/fabric8/icon.png "Prometheus")
# Ansible Role Prometheus

This started of as a simple proof-of-concept for installing [Prometheus](http://prometheus.io) from RPM or Deb, but evolved with time considering distribution packages aren't as frequent as prometheus releases :(
This role basically installs rpm/deb and setups systemd unit files.

Out of the box, it will install the following prometheus components:

  - [Prometheus](http://prometheus.io) version `2.0.0`
  - [Alertmanager](https://prometheus.io/docs/alerting/alertmanager/) version `0.11.0`
  - [Node exporter](https://github.com/prometheus/node_exporter) version `0.15.1`
  - [Pushgateway](https://github.com/prometheus/pushgateway) version `0.4.0`. Important [notes](https://prometheus.io/docs/practices/pushing/) on pushing over pulling.
  (still with no official release using)
  - [Elasticsearch exporter](https://github.com/justwatchcom/elasticsearch_exporter) version `1.0.2`
  - [CrateDB adapter](https://github.com/crate/crate_adapter) version `0.1-2017100611-d24e213`


## Basic Features & Defaults

* Prometheus listening on default port of 9090 `http://localhost:9090`
* Node exporter listening on default port of 9100 `http://localhost:9100/metrics`
* Alertmanager listening on default port of 9093 `http://localhost:9093`

## Additional Features:

* Pushgateway listening on default port of 9091 `http://localhost:9091`
* Cratedb adapter listening on port: 9268
* Custom Alertmanager configurations - example below
* Custom Prometheus configurations - example below

## Role Variables

A few Prometheus tweaks out of the box:

* Prometheus configuration:
    * _DEPRECATED_ `prometheus_opt_storage_local_memory_chunks: 1048576`
    * _DEPRECATED_ `prometheus_opt_storage_local_max_chunks_to_persist: 524288`
    * `prometheus_opt_storage_local_target_heap_size: '3221225472'`
  (see also [this PDF](https://schd.ws/hosted_files/cloudnativeeu2017/ce/Slides.pdf
))
    * `prometheus_opt_storage_local_retention: "720h" # 1month`
* Logging:
    * `prometheus_opt_log_format: 'logger:syslog?appname=prometheus&local=7'`
    * `prometheus_opt_log_level: info`
  (You might disable this when using some logging stack or might use it for debugging when you log stack goes to hell and back)

* `prometheus_skip_config: false` - if you want to install without configuration / stay with the default apt/rpm (or similar style via GitHub) config - set this to `true`
* `promehtues_use_role_rules: false` - There some sample alerts I scattered and found useful to build upon so feel free to reuse (Note: Option will not work without prometheus_skip_config `true`)

In general any command-line arguments you wish to pass to `prometheus`, `node_exporter` & `alertmanager` is done by setting or adding one of the `prometheus_components_opts`:

As an example with Alertmanager:

    prometheus_alertmanager_opts:
      - "web.listen-address={{ prometheus_alertmanager_opt_web_listen_address }}"
      - "log.format={{ prometheus_alertmanager_log_format }}"

## Alertmanager Configuration

In order to use this option you should set `prometheus_alertmanager_custom_config` to `true`
Then use the following Options:

* To define receivers:
The tricky part below was customising the slack template see the `slack.tmpl.j2` to understand how sick this looks like but it works!

```
prometheus_alertmanager_recievers:
  receivers:
    - name: 'slack'
      slack_configs:
      - channel: '#autobots'
        send_resolved: true
        api_url: https://hooks.slack.com/services/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
        title: "{{'{{'}} template \"slack.custom.title\" . {{'}}'}}"
        text: "{{'{{'}} template \"slack.custom.text_long\" . {{'}}'}}"
        color: "{{ '{{' }}  if eq .Status \"firing\" {{'}}'}}danger{{'{{'}} else {{'}}'}}good{{'{{'}} end {{'}}'}}"
        title_link: "{{ '{{' }}  template \"slack.custom.titlelink\" {{ '}}' }}"
```
* To define routes:

```
prometheus_alertmanager_routes:
  route:
    group_by: [ 'alertname', 'cluster', 'service' ]
    group_wait: 30s
    group_interval: 5m
    receiver: slack
```
* To define inhibit_rules:

```
prometheus_alertmanager_inhibit_rules:
    inhibit_rules:
    - source_match:
        severity: 'major'
      target_match:
        severity: 'critical'
      equal: ['alertname']
    - source_match:
        severity: 'critical'
      target_match:
        severity: 'warning'
      equal: ['alertname']
    - source_match:
        severity: 'warning'
      target_match:
        severity: 'indeterminate'
      equal: ['alertname']
```

## TODO / WIP (Pull request anyone ? ;)

* [ ] Add a Prometheus job:
  Similar to the way the role appends a file to `rules` / `scrape_configs`
* [ ] Add alertmanager.yml custom configurations for slack, pager_duty, victorops etc.

## Supported Customisations

* Install and configure CrateDB adapter, see more info [here](https://github.com/crate/crate_adapter) - disabled by default need to pass:
* If you have a poor service discovery like myself at the moment I added support for [file based discovery](https://prometheus.io/docs/operating/configuration/#<file_sd_config>) in the following format:

    pass a variable named `prometheus_static_targets` with a value in the following formats:

    1) - configuring a single target

          - { name: traefik,
              host: traefik.mydomain.com,
              port: 8080,
              metrics_path: /metrics
            }

    2) - configuring a multiple targets

          - { name: swarm,
              hosts: "[
                      'swarm-01.mydomain.com:9100',
                      'swarm-02.mydomain.com:9100',
                      'swarm-03.mydomain.com:9100',
                      'swarm-04.mydomain.com:9100',
                      'swarm-05.mydomain.com:9100',
                      'swarm-07.mydomain.com:9100'
                     ]",
              metrics_path: /metrics,
              port: 9100
            }
    3) Combine the 2 it will still work ;)

    `prometheus_static_targets` will invoke a task to create an entry under the `job: 'node'` in `the /etc/prometheus/prometheus.yml` like so:

          - job_name: 'node'
            file_sd_configs:
              - files:
                - '{{ prometheus_file_sd_config_path }}/*.yml'

    In the above example the resulted files will reside under `/etc/prometheus/jobs/<name>.yml`

## Example Playbook

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    ---
    - hosts: centos,ubuntu
      become: true
      roles:
        - role: ansible-role-prometheus


A more complete solution would look something like:

    ---
    - hosts: centos,ubuntu
      become: true
      tasks:
      roles:
         - role: shelleg.ntp
         - role: shelleg.cratedb
           cratedb_adapter_testing: true
           cratedb_adapter_prometheus_init: true
         - role: ansible-role-prometheus
           prometheus_cratedb_remote_write: true
           prometheus_cratedb_remote_read: true
           prometheus_components:
             - alertmanager
             - node_exporter
             - prometheus
             - crate_adapter
           promehtues_use_role_rules: true
           prometheus_static_targets:
           - { name: traefik,
               host: traefik.mydomain.com,
               port: 8080,
               metrics_path: /metrics
               }


## License

[Apache 2](https://choosealicense.com/licenses/apache-2.0/)


## Author Information

[Haggai Philip Zagury](http://www.tikalk.com/devops/haggai)
