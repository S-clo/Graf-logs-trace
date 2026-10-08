# Graf-logs-trace
observability logs and trace
# Observability + alerting stack with traces (single host, 4 vCPU / 16 GB)

Designed to sit beside Zabbix: Zabbix owns host and infrastructure metrics (CPU, memory, disk, network, Windows, SNMP);
this stack owns logs, traces, service availability, certificates, container health and alert routing for those.

Prometheus (metrics, alert rules) + Alertmanager (alert routing) + Loki (logs) + Tempo (traces)
+ Grafana (UI) + Alloy (log and trace collector) + Blackbox Exporter (uptime/TLS probes)
+ cAdvisor (container metrics). Host metrics are NOT collected here (Zabbix does that).

How it fits together:
- Prometheus scrapes cAdvisor and blackbox probes, receives Tempo span metrics, and evaluates alert rules.
- Alerts go to Alertmanager, which groups, de-duplicates, silences and routes them.
- Alloy tails container logs into Loki, and receives application traces (OTLP) into Tempo.
- Tempo generates span metrics (rate/errors/duration per service) into Prometheus, so you can alert on tracing data.
- Loki's ruler evaluates log alert rules and sends them to the same Alertmanager.
- Grafana ties it together: metric -> logs -> trace links, plus an Alertmanager view.

## Resource requirements (this bundle)

Target: 4 vCPU, 16 GB RAM, 200-300 GB SSD, Ubuntu 22.04/24.04.

Memory limits: Prometheus 3G, Loki 3G, Tempo 2G, Grafana 1G, Alloy 512M, cAdvisor 256M, Alertmanager 128M,
Blackbox 64M (about 10 GB ceiling, leaving 5-6 GB for OS, Docker and bursts).

Rough capacity (estimates; re-check after a week of real data): ~30 hosts' worth of containers, ~100k series,
10-20 GB/day logs, moderate traces with 10-20% sampling on busy services.

Suggested retention: metrics 30-60 days (`--storage.tsdb.retention.time` in compose), logs 14-30 days
(`retention_period` in loki.yaml), traces 3-7 days (`block_retention` in tempo.yaml).

Disk rules of thumb: logs ~10:1 compression (5 GB/day raw is ~0.5-1 GB/day on disk); metrics ~1.5 bytes/sample;
traces 5-20 GB for 7 days at this scale. Logs and traces are where most disk goes, so have Zabbix watch this server's disk.

If you outgrow one box: add object storage (S3/MinIO) and consider Mimir, or move to Kubernetes.

## Deployment steps

1. Provision an Ubuntu 22.04/24.04 VM (Pilot tier, SSD). Install Docker Engine and the Compose plugin.
2. Copy this folder to the server. `cp .env.example .env`, set a strong `GRAFANA_ADMIN_PASSWORD` and `GRAFANA_ROOT_URL`, and bump image versions to current stable releases.
3. Validate: `docker compose config`
4. Start: `docker compose up -d`, then `docker compose ps` (all Up).
5. Validate configs:
   - `docker compose exec prometheus promtool check config /etc/prometheus/prometheus.yml`
   - `docker compose exec alertmanager amtool check-config /etc/alertmanager/alertmanager.yml`
   (the Grafana image downloads the Zabbix plugin on first start, so the server needs internet access then)
6. Open Grafana (port 3000). Datasources (Prometheus, Loki, Tempo, Alertmanager) are pre-provisioned; click Save & test on each.
7. Verify data:
   - Explore > Prometheus: `up` (all targets 1); Status > Targets lives at http://localhost:9090/targets on the server
   - Explore > Loki: `{service="grafana"}`
   - Alloy UI: http://localhost:12345 on the server
8. Import dashboards from grafana.com: Node Exporter Full (1860), a cAdvisor dashboard, Blackbox Exporter (7587).
9. Configure notifications: edit `config/alertmanager/alertmanager.yml` (email / Teams / webhook receivers), then reload:
   `docker compose restart alertmanager`
10. Test an alert end to end: `docker compose stop cadvisor`. InstanceDown fires in ~2-3 minutes. Start it again to see the resolve.
11. Put a TLS reverse proxy (Nginx/Caddy/Traefik) in front of Grafana. Firewall 3000/4317/4318 to internal networks.
    Prometheus and Alertmanager are bound to localhost only.
    Make sure Zabbix monitors this server (especially its disk).
12. Back up volumes (`prometheus-data`, `loki-data`, `tempo-data`, `grafana-data`, `alertmanager-data`) or snapshot the VM. Keep the config folder in Git.

## Day-to-day changes (no restarts needed)

- **Add a server:** nothing to do here for host metrics; add it in Zabbix. Add its logs via Alloy (below) and its services as probes.
- **Monitor a website/API:** add the URL to `targets/http-endpoints.yml` (set `severity: critical` for important ones). TLS expiry alerts come free with this.
- **Monitor a port (database, FIX gateway, SMTP):** add `host:port` to `targets/tcp-endpoints.yml`.
- **Track a non-web certificate (SMTPS, LDAPS):** add `host:port` to `targets/tls-endpoints.yml`.
- **Zabbix in Grafana:** the Zabbix plugin is installed on first start. Create a read-only Zabbix API user, fill in `grafana/provisioning/datasources/zabbix.yaml.example`, rename it to `zabbix.yaml`, and `docker compose restart grafana`. Then enable the plugin under Administration > Plugins if prompted.
- **Keep alerts distinguishable:** send both systems to the same Teams/email channel but prefix subjects ([Zabbix] / [Obs]).
- **Change alert rules:** edit `config/prometheus/rules/*.yml`, validate with promtool, then `curl -X POST http://localhost:9090/-/reload`.
- **Log alerts:** edit `config/loki-rules/log-alerts.yml`, then `docker compose restart loki`.

## Traces

Send OpenTelemetry data to `http://<server>:4317` (gRPC) or `:4318` (HTTP). Zero-code instrumentation uses agents
configured only by environment variables, for example:

    OTEL_SERVICE_NAME=my-app
    OTEL_EXPORTER_OTLP_ENDPOINT=http://<server>:4317
    OTEL_TRACES_EXPORTER=otlp

- Java: `java -javaagent:opentelemetry-javaagent.jar -jar app.jar`
- Python: `opentelemetry-instrument python app.py`
- Node.js: `node --require @opentelemetry/auto-instrumentations-node/register app.js`
- PHP/WordPress: OTel PHP extension plus Composer packages, configured through environment variables
- Any language, no restart flags: eBPF-based tools (Grafana Beyla / OpenTelemetry eBPF), run alongside the app

Review captured span attributes (SQL, URLs, headers) for sensitive data before production use. Test on staging first.
Once traces flow, Grafana's Tempo view shows the service graph, and the `ServiceHighErrorRate` rule becomes active.

## Collecting from other hosts

- **Logs:** run Alloy on the remote host and point `loki.write` at `http://<server>:3100/loki/api/v1/push`. Windows Event Logs use `loki.source.windowsevent`.
- **Metrics:** only container and probe metrics live here. Hosts remain in Zabbix.
- Publish Loki's port 3100 only behind a proxy with authentication, limited to known source IPs.

## Hardening checklist

- Grafana SSO (Microsoft Entra ID or LDAP) with role-based access; disable the default admin once it works
- Alloy mounts the Docker socket read-only: treat the host as sensitive and do not expose Alloy's UI port
- Redact sensitive fields from logs in Alloy (`loki.process`) and from spans before storing
- `--web.enable-lifecycle` and the remote-write receiver are on for Prometheus: keep it internal-only (it is localhost-bound here)
- Move to object storage (S3/MinIO) and consider Mimir when data outgrows one disk or you need high availability
