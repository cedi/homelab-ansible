# Alloy Upgrade + Endpoint Migration — Design

**Date:** 2026-06-18
**Status:** Pending re-approval (scope expanded to fix OTLP metrics/logs routing)

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

### monitoring.yml (nuc) — fix OTLP receiver routing (metrics/logs failure)

**Root cause (confirmed on the live host, alloy v1.8.1):** the
`otelcol.receiver.otlp "default"` output block forwards `metrics`, `logs`, AND
`traces` all into the single Tempo exporter. Tempo only ingests traces, and the
old Tempo endpoint is now down — the journal shows a continuous stream of
`otelcol.exporter.otlp.tempo` errors (`connection refused`, `sending queue is
full`, `Dropping data`). Every OTLP-ingested metric and log is dropped along
with the traces. The direct paths (`prometheus.scrape`→Mimir, file/journal→Loki)
are healthy.

Fix: add bridge exporters so each signal goes to its correct backend.

```alloy
otelcol.exporter.otlphttp "tempo" {
  client {
    endpoint = "https://homelab-tempo.k8s.specht-labs.de/otlp"
  }
}

otelcol.exporter.prometheus "to_mimir" {
  forward_to = [prometheus.remote_write.mimir.receiver]
}

otelcol.exporter.loki "to_loki" {
  forward_to = [loki.write.grafana_loki.receiver]
}

otelcol.receiver.otlp "default" {
  grpc {
    endpoint = "0.0.0.0:4317"
  }
  http {
    endpoint = "0.0.0.0:4318"
  }
  output {
    metrics = [otelcol.exporter.prometheus.to_mimir.input]
    logs    = [otelcol.exporter.loki.to_loki.input]
    traces  = [otelcol.exporter.otlphttp.tempo.input]
  }
}
```

This supersedes step 4's Tempo change for `monitoring.yml` (the `otelcol.exporter.otlphttp "tempo"` block above is the nuc form).

### hole / jitsi — Tempo exporter

`hole` and `jitsi` define an `otelcol.exporter.*` "tempo" but have **no**
`otelcol.receiver.otlp`, so nothing feeds it (no OTLP ingestion on those hosts).
Step 4 still applies: convert it to `otelcol.exporter.otlphttp "tempo"` pointed
at `https://spechtlabs-tempo.k8s.specht-labs.de/otlp`. It remains an inert
(unreferenced) component, matching the pre-existing structure.

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
