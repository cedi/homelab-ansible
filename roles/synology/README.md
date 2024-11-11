# Ansible Role Synology

This ansible role copies the config files for my docker-compose setup running on my Synology NAS.

The o11y stack for monitoring my Synology using SNMP is from [wozniakpawel/synology-grafana-prometheus-overly-comprehensive-dashboard].

## Installed files

* [media/docker-compose.yaml]
  * Starts the following Containers:
    * Jellyfin
    * nginx-proxy-manager
* [o11y/docker-compose.yaml]
  * Starts the following Containers:
    * node-exporter
    * snmp-exporter
      * [o11y/snmp-exporter/snmp.yaml]
    * cadvisor
    * alloy

## Example Playbook

```yaml
- hosts: synology
  roles:
    - role: synology
```

## License

MIT / BSD

[media/docker-compose.yaml]: files/media/docker-compose.yaml
[o11y/snmp-exporter/snmp.yaml]: files/o11y/snmp-exporter/snmp.yaml
[wozniakpawel/synology-grafana-prometheus-overly-comprehensive-dashboard]: https://github.com/wozniakpawel/synology-grafana-prometheus-overly-comprehensive-dashboard
