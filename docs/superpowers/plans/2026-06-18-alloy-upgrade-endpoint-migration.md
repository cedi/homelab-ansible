# Alloy Upgrade + Endpoint Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Upgrade the `grafana.grafana.alloy` deployment to collection 6.1.0 and migrate logs/metrics/traces to the new Tailscale-network k8s endpoints, fixing the OTLP receiver routing that currently drops all OTLP metrics and logs.

**Architecture:** Three Ansible playbooks (`monitoring.yml`, `monitoring-hole.yml`, `monitoring-jitsi.yml`) configure the `grafana.grafana.alloy` role with an inline Alloy river config. Each playbook's embedded config is edited to point at the new endpoints, drop all client-side tenant config (endpoints auto-inject `X-Scope-Org-ID`), and switch Tempo to OTLP/HTTP. The `nuc` config additionally gains bridge exporters so OTLP-ingested metrics→Mimir and logs→Loki instead of all signals into Tempo.

**Tech Stack:** Ansible (ansible-core 2.20.6 via mise), `grafana.grafana` collection 6.1.0, Grafana Alloy (river config), ansible-galaxy.

## Global Constraints

- Collection: `grafana.grafana` pinned to `6.1.0` (inline play pin AND `requirements.yml`).
- Drop ALL client-side tenant config: no `tenant_id` (loki.write), no `X-Scope-OrgID` headers (mimir). Endpoints auto-inject the tenant header.
- All new endpoints are HTTPS.
- Endpoint mapping:
  - `nuc` → `homelab-{mimir,loki,tempo}.k8s.specht-labs.de`
  - `hole` → `spechtlabs-{mimir,loki,tempo}.k8s.specht-labs.de`
  - `jitsi` → `spechtlabs-{mimir,loki,tempo}.k8s.specht-labs.de`
- URL forms: Loki `https://<host>/loki/api/v1/push`; Mimir `https://<host>/api/v1/push`; Tempo `https://<host>/otlp` via `otelcol.exporter.otlphttp`.
- Tooling: invoke ansible via `mise exec --` per the repo's tool-installation directive.
- Preserve `jitsi`'s `external_labels { hostname }`, all scrape jobs, log sources, relabel rules, `alloy_user_groups`, `alloy_env_file_vars`.
- No unit-test framework exists; verification = `ansible-playbook --syntax-check`, galaxy resolution, and live journal inspection on `nuc` (`ssh root@nuc`).

---

### Task 1: Pin collection 6.1.0 in requirements.yml

**Files:**
- Modify: `requirements.yml`

**Interfaces:**
- Produces: collection `grafana.grafana==6.1.0` resolvable by ansible-galaxy and by the inline play pins in Tasks 2–4.

- [ ] **Step 1: Edit requirements.yml**

Change the `grafana.grafana` entry from the bare name to a pinned version. Final file:

```yaml
---
collections:
  - name: community.general
  - name: ansible.posix
  - name: grafana.grafana
    version: "6.1.0"

roles:
  - name: geerlingguy.node_exporter
  - name: artis3n.tailscale
  - name: geerlingguy.nut_client
```

- [ ] **Step 2: Install/resolve the collection**

Run: `cd /Users/cedi/src/gh/cedi/homelab-ansible && mise exec -- ansible-galaxy collection install -r requirements.yml -p .ansible/collections --force`
Expected: output includes `grafana.grafana:6.1.0 was installed successfully`.

- [ ] **Step 3: Commit**

```bash
git add requirements.yml
git commit -m "build: pin grafana.grafana collection to 6.1.0"
```

---

### Task 2: Migrate nuc (monitoring.yml) — endpoints, tenancy, Tempo, OTLP routing fix

**Files:**
- Modify: `playbooks/monitoring.yml`

**Interfaces:**
- Consumes: `grafana.grafana` 6.1.0 (Task 1).
- Produces: working `nuc` Alloy config where OTLP metrics→Mimir, logs→Loki, traces→Tempo.

- [ ] **Step 1: Bump the inline collection pin**

In `playbooks/monitoring.yml`, change:

```yaml
  collections:
    - grafana.grafana:5.7.0
```

to:

```yaml
  collections:
    - grafana.grafana:6.1.0
```

- [ ] **Step 2: Migrate the Loki endpoint and drop tenant_id**

Replace the `loki.write "grafana_loki"` block:

```alloy
          loki.write "grafana_loki" {
            endpoint {
              url       = "https://loki-gateway.sphinx-map.ts.net/loki/api/v1/push"
              tenant_id = "homelab"
            }
          }
```

with:

```alloy
          loki.write "grafana_loki" {
            endpoint {
              url = "https://homelab-loki.k8s.specht-labs.de/loki/api/v1/push"
            }
          }
```

- [ ] **Step 3: Migrate the Mimir endpoint and drop the X-Scope-OrgID header**

Replace the `prometheus.remote_write "mimir"` block:

```alloy
          prometheus.remote_write "mimir" {
            endpoint {
              url = "https://mimir-nginx.sphinx-map.ts.net/api/v1/push"
              headers = {
                "X-Scope-OrgID" = "homelab",
              }
            }
          }
```

with:

```alloy
          prometheus.remote_write "mimir" {
            endpoint {
              url = "https://homelab-mimir.k8s.specht-labs.de/api/v1/push"
            }
          }
```

- [ ] **Step 4: Replace the Tempo exporter and fix OTLP receiver routing**

Replace the entire trailing block (the `otelcol.exporter.otlp "tempo"` and `otelcol.receiver.otlp "default"` definitions):

```alloy
          otelcol.exporter.otlp "tempo" {
            client {
              endpoint = "tempo-distributor.sphinx-map.ts.net/otlp-http"
            }
          }

          otelcol.receiver.otlp "default" {
            grpc {
              endpoint = "0.0.0.0:4317"
            }
            http {
              endpoint = "0.0.0.0:4318"
            }
            output {
              metrics = [otelcol.exporter.otlp.tempo.input]
              logs    = [otelcol.exporter.otlp.tempo.input]
              traces  = [otelcol.exporter.otlp.tempo.input]
            }
          }
```

with:

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

- [ ] **Step 5: Syntax-check**

Run: `cd /Users/cedi/src/gh/cedi/homelab-ansible && mise exec -- ansible-playbook -i inventory/homelab.yaml playbooks/monitoring.yml --syntax-check`
Expected: exit 0, no errors (prints the playbook name).

- [ ] **Step 6: Commit**

```bash
git add playbooks/monitoring.yml
git commit -m "feat: migrate nuc alloy to homelab k8s endpoints and fix OTLP routing"
```

---

### Task 3: Migrate hole (monitoring-hole.yml) — endpoints, tenancy, Tempo, systemd fix

**Files:**
- Modify: `playbooks/monitoring-hole.yml`

**Interfaces:**
- Consumes: `grafana.grafana` 6.1.0 (Task 1).
- Produces: working `hole` Alloy config on `spechtlabs-*` endpoints with a correctly-applied `User=root` override.

- [ ] **Step 1: Bump the inline collection pin**

Change `- grafana.grafana:5.7.0` to `- grafana.grafana:6.1.0`.

- [ ] **Step 2: Fix the malformed alloy_systemd_override nesting**

Replace:

```yaml
        alloy_systemd_override:
          alloy_systemd_override: |
            [Service]
            User=root
```

with:

```yaml
        alloy_systemd_override: |
          [Service]
          User=root
```

- [ ] **Step 3: Migrate the Loki endpoint and drop tenant_id**

Replace:

```alloy
          loki.write "grafana_loki" {
            endpoint {
              url       = "https://loki-gateway.sphinx-map.ts.net/loki/api/v1/push"
              tenant_id = "cedi-dev"
            }
          }
```

with:

```alloy
          loki.write "grafana_loki" {
            endpoint {
              url = "https://spechtlabs-loki.k8s.specht-labs.de/loki/api/v1/push"
            }
          }
```

- [ ] **Step 4: Migrate the Mimir endpoint and drop the X-Scope-OrgID header**

Replace:

```alloy
          prometheus.remote_write "mimir" {
            endpoint {
              url = "https://mimir-nginx.sphinx-map.ts.net/api/v1/push"
              headers = {
                "X-Scope-OrgID" = "cedi-dev",
              }
            }
          }
```

with:

```alloy
          prometheus.remote_write "mimir" {
            endpoint {
              url = "https://spechtlabs-mimir.k8s.specht-labs.de/api/v1/push"
            }
          }
```

- [ ] **Step 5: Migrate the Tempo exporter to OTLP/HTTP**

Replace:

```alloy
          otelcol.exporter.otlp "tempo" {
            client {
              endpoint = "tempo-distributor.sphinx-map.ts.net/otlp-http"
            }
          }
```

with:

```alloy
          otelcol.exporter.otlphttp "tempo" {
            client {
              endpoint = "https://spechtlabs-tempo.k8s.specht-labs.de/otlp"
            }
          }
```

- [ ] **Step 6: Syntax-check**

Run: `cd /Users/cedi/src/gh/cedi/homelab-ansible && mise exec -- ansible-playbook -i inventory/homelab.yaml playbooks/monitoring-hole.yml --syntax-check`
Expected: exit 0, no errors.

- [ ] **Step 7: Commit**

```bash
git add playbooks/monitoring-hole.yml
git commit -m "feat: migrate hole alloy to spechtlabs k8s endpoints and fix systemd override"
```

---

### Task 4: Migrate jitsi (monitoring-jitsi.yml) — endpoints, tenancy, Tempo, systemd fix

**Files:**
- Modify: `playbooks/monitoring-jitsi.yml`

**Interfaces:**
- Consumes: `grafana.grafana` 6.1.0 (Task 1).
- Produces: working `jitsi` Alloy config on `spechtlabs-*` endpoints, `external_labels` preserved.

- [ ] **Step 1: Bump the inline collection pin**

Change `- grafana.grafana:5.7.0` to `- grafana.grafana:6.1.0`.

- [ ] **Step 2: Fix the malformed alloy_systemd_override nesting**

Replace:

```yaml
        alloy_systemd_override:
          alloy_systemd_override: |
            [Service]
            User=root
```

with:

```yaml
        alloy_systemd_override: |
          [Service]
          User=root
```

- [ ] **Step 3: Migrate the Loki endpoint and drop tenant_id (preserve external_labels)**

Replace:

```alloy
          loki.write "grafana_loki" {
            endpoint {
              url       = "https://loki-gateway.sphinx-map.ts.net/loki/api/v1/push"
              tenant_id = "cedi-dev"
            }

            external_labels = {
              hostname = "meet.realschule-rottweil.de",
            }
          }
```

with:

```alloy
          loki.write "grafana_loki" {
            endpoint {
              url = "https://spechtlabs-loki.k8s.specht-labs.de/loki/api/v1/push"
            }

            external_labels = {
              hostname = "meet.realschule-rottweil.de",
            }
          }
```

- [ ] **Step 4: Migrate the Mimir endpoint and drop the X-Scope-OrgID header**

Replace:

```alloy
          prometheus.remote_write "mimir" {
            endpoint {
              url = "https://mimir-nginx.sphinx-map.ts.net/api/v1/push"
              headers = {
                "X-Scope-OrgID" = "cedi-dev",
              }
            }
          }
```

with:

```alloy
          prometheus.remote_write "mimir" {
            endpoint {
              url = "https://spechtlabs-mimir.k8s.specht-labs.de/api/v1/push"
            }
          }
```

- [ ] **Step 5: Migrate the Tempo exporter to OTLP/HTTP**

Replace:

```alloy
          otelcol.exporter.otlp "tempo" {
            client {
              endpoint = "tempo-distributor.sphinx-map.ts.net/otlp-http"
            }
          }
```

with:

```alloy
          otelcol.exporter.otlphttp "tempo" {
            client {
              endpoint = "https://spechtlabs-tempo.k8s.specht-labs.de/otlp"
            }
          }
```

- [ ] **Step 6: Syntax-check**

Run: `cd /Users/cedi/src/gh/cedi/homelab-ansible && mise exec -- ansible-playbook -i inventory/homelab.yaml playbooks/monitoring-jitsi.yml --syntax-check`
Expected: exit 0, no errors.

- [ ] **Step 7: Commit**

```bash
git add playbooks/monitoring-jitsi.yml
git commit -m "feat: migrate jitsi alloy to spechtlabs k8s endpoints and fix systemd override"
```

---

### Task 5: Deploy to nuc and verify the OTLP/endpoint fix live

This is the real acceptance test: `nuc` is the only host with an OTLP receiver and was actively dropping data. Apply and confirm the error stream stops.

**Files:** none (deploy + verify only)

- [ ] **Step 1: Capture a baseline error count (pre-deploy)**

Run: `ssh root@nuc 'journalctl -u alloy --since "-5min" --no-pager | grep -c "exporter.otlp.tempo"'`
Expected: a non-zero count (the current failing stream).

- [ ] **Step 2: Apply the nuc playbook**

Run: `cd /Users/cedi/src/gh/cedi/homelab-ansible && mise exec -- ansible-playbook -i inventory/homelab.yaml playbooks/monitoring.yml`
Expected: play recap shows `failed=0`; the alloy config/service tasks report `changed`.

- [ ] **Step 3: Confirm the new config is deployed**

Run: `ssh root@nuc 'grep -E "otelcol.exporter|specht-labs" /etc/alloy/config.alloy'`
Expected: shows `otelcol.exporter.otlphttp "tempo"`, `otelcol.exporter.prometheus "to_mimir"`, `otelcol.exporter.loki "to_loki"`, and the `homelab-*.k8s.specht-labs.de` endpoints. No `sphinx-map.ts.net` remains.

- [ ] **Step 4: Confirm Alloy is healthy and errors stopped**

Run: `ssh root@nuc 'systemctl is-active alloy; sleep 30; journalctl -u alloy --since "-1min" --no-pager | grep -iE "error|Dropping data|queue is full" | tail -20'`
Expected: `active`; no `otelcol.exporter.otlp.tempo` connection-refused / queue-full / dropping-data errors. (Brief startup warnings are acceptable; a continuing error stream is not.)

- [ ] **Step 5: Confirm no component-evaluation errors**

Run: `ssh root@nuc 'journalctl -u alloy --since "-1min" --no-pager | grep -iE "level=error|failed to (build|evaluate)" | tail -20'`
Expected: empty (no config-build/evaluation errors — confirms `otelcol.exporter.prometheus`/`otelcol.exporter.loki` resolve correctly on the deployed Alloy build).

---

## Notes on hole / jitsi deployment

`hole` and `jitsi` are not in the default `mise` apply flow and `hole` is on a remote network. Deploy them when reachable with:
`mise exec -- ansible-playbook -i inventory/homelab.yaml playbooks/monitoring-hole.yml` (and the jitsi equivalent). Their Alloy has no OTLP receiver, so the key checks are: service `active`, config shows `spechtlabs-*` endpoints, and `prometheus.remote_write`/`loki.write` show no push errors in the journal.
