# Alloy Upgrade + Endpoint Migration — Design

**Date:** 2026-06-18
**Status:** Approved

## Goal

Upgrade the `grafana.grafana.alloy` deployment to the latest collection and
migrate the logs/metrics/traces ingestion endpoints to the new Tailscale-network
k8s endpoints. The new endpoints auto-inject the multi-tenant `X-Scope-Org-ID`
header, so all client-side tenant configuration is removed.

## Affected files

- `playbooks/monitoring.yml` — host group `nuc`
- `playbooks/monitoring-hole.yml` — host `hole`
- `playbooks/monitoring-jitsi.yml` — host `meet` (jitsi)
- `requirements.yml`

## Endpoint mapping

`nuc` keeps the `homelab` tenant lineage; `hole` and `jitsi` (formerly the
`cedi-dev` tenant) move to the `spechtlabs` endpoints.

| Host  | Metrics (Mimir)                          | Logs (Loki)                              | Traces (Tempo)                            |
|-------|------------------------------------------|------------------------------------------|-------------------------------------------|
| nuc   | `homelab-mimir.k8s.specht-labs.de`       | `homelab-loki.k8s.specht-labs.de`        | `homelab-tempo.k8s.specht-labs.de`        |
| hole  | `spechtlabs-mimir.k8s.specht-labs.de`    | `spechtlabs-loki.k8s.specht-labs.de`     | `spechtlabs-tempo.k8s.specht-labs.de`     |
| jitsi | `spechtlabs-mimir.k8s.specht-labs.de`    | `spechtlabs-loki.k8s.specht-labs.de`     | `spechtlabs-tempo.k8s.specht-labs.de`     |

Full URLs/paths (all HTTPS):

- **Loki**: `https://<host>/loki/api/v1/push`
- **Mimir**: `https://<host>/api/v1/push`
- **Tempo**: `https://<host>/otlp` (Alloy's `otelcol.exporter.otlphttp` appends `/v1/traces`)

## Changes

### All three playbooks

1. **Collection version**: bump the inline play-level pin
   `grafana.grafana:5.7.0` → `grafana.grafana:6.1.0`.

2. **Loki (`loki.write "grafana_loki"`)**:
   - Set `url` to the host's Loki push URL.
   - **Remove** the `tenant_id` argument (server auto-injects the tenant header).

3. **Mimir (`prometheus.remote_write "mimir"`)**:
   - Set `url` to the host's Mimir push URL.
   - **Remove** the `headers = { "X-Scope-OrgID" = ... }` block.

4. **Tempo**: replace `otelcol.exporter.otlp "tempo"` (gRPC) with
   `otelcol.exporter.otlphttp "tempo"`:

   ```alloy
   otelcol.exporter.otlphttp "tempo" {
     client {
       endpoint = "https://<host>/otlp"
     }
   }
   ```

### monitoring.yml (nuc) — receiver reference only

The `otelcol.receiver.otlp "default"` output block keeps forwarding
`metrics`, `logs`, and `traces` (left as-is per decision), but the exporter
component name changes, so the three references update from
`otelcol.exporter.otlp.tempo.input` →
`otelcol.exporter.otlphttp.tempo.input`.

### monitoring-hole.yml & monitoring-jitsi.yml — systemd override fix

Both files currently have a malformed, double-nested `alloy_systemd_override`:

```yaml
alloy_systemd_override:
  alloy_systemd_override: |
    [Service]
    User=root
```

Flatten to the correct string form (matching `monitoring.yml`) so the
`User=root` override actually applies:

```yaml
alloy_systemd_override: |
  [Service]
  User=root
```

### requirements.yml

Pin the collection so renovate can track and bump it:

```yaml
collections:
  - name: community.general
  - name: ansible.posix
  - name: grafana.grafana
    version: "6.1.0"
```

## Preserved as-is

- `jitsi`'s `loki.write` `external_labels { hostname = "meet...." }` (not tenancy).
- All `prometheus.scrape` jobs, log file/journal sources, relabel rules,
  `alloy_user_groups`, and `alloy_env_file_vars`.

## Out of scope (noted, not changed)

- `site.yml` references a non-existent `metrics-collector.yml` and does not
  import `monitoring-hole.yml` — pre-existing, untouched.
- Alloy binary version stays at the role default (`latest`).

## Verification

- `mise exec -- ansible-playbook -i inventory/homelab.yaml playbooks/monitoring.yml --syntax-check`
  (and the same for `monitoring-hole.yml`, `monitoring-jitsi.yml`).
- Re-pull collections: `ansible-galaxy collection install -r requirements.yml`
  resolves `grafana.grafana==6.1.0`.
