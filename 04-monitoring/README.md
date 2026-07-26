# 04 — Monitoring (Prometheus + Grafana)

Watch CPU, RAM, disk, and per-container resource use across the lab. **Prometheus** scrapes metrics, **node_exporter** exposes host stats, **cAdvisor** exposes per-container stats, and **Grafana** draws the dashboards.

**Runs on:** an LXC (1 core / 1.5GB RAM / 16GB disk), Debian 13, Docker + nesting enabled — the standard [LXC baseline](../00-proxmox-host/#3--the-lxc-baseline-used-by-every-container-service).

Set this up early: it's the fastest way to catch a container that's eating all your RAM as you add services.

## Files here

| File | What it is |
|---|---|
| `docker-compose.yml` | The four-container stack, all on one `monitoring` docker network. |
| `prometheus/prometheus.yml` | Scrape config — targets containers by name (`node_exporter:9100`, `cadvisor:8080`). |

## 1. Folder layout

Inside the LXC:

```
~/monitoring/
├── docker-compose.yml
└── prometheus/
    └── prometheus.yml
```

Copy `docker-compose.yml` and `prometheus/prometheus.yml` (both in this deliverable) into that structure.

## 2. Start the stack

```bash
cd ~/monitoring
docker compose up -d
docker compose ps
```

All four containers (prometheus, node_exporter, cadvisor, grafana) sit on the `monitoring` docker network, so they can reach each other by container name — that's why `prometheus.yml` targets `node_exporter:9100` and `cadvisor:8080` instead of an IP.

## 3. Verify Prometheus is scraping

Go to `http://<LXC-IP>:9090/targets`. You should see three targets (`prometheus`, `node_exporter`, `cadvisor`) all `UP`. If node_exporter or cadvisor show as `DOWN`, it's almost always the container name not resolving — confirm all four services are actually on the `monitoring` network (`docker network inspect monitoring`).

## 4. Set up Grafana

1. Go to `http://<LXC-IP>:3000` — default login is `admin` / `admin` (you'll be forced to change it on first login).
2. **Connections → Data sources → Add data source → Prometheus.**
3. Set the URL to `http://prometheus:9090` (container-to-container, same network — not localhost, not the LXC IP).
4. **Save & test** — should show "Successfully queried the Prometheus API."

## 5. Import dashboards

Instead of building panels from scratch, grab these community dashboards (**Dashboards → New → Import**, then paste the ID):

- **Node Exporter Full** — ID `1860`. Covers CPU, RAM, disk, network for the LXC/host.
- **cAdvisor / Docker Container Monitoring** — ID `19908` or `193` (either works; `19908` is more current). Covers per-container CPU/mem/network.

When importing, select your Prometheus data source from step 4 when prompted.

## 6. Resource note

1.5GB RAM for Prometheus + Grafana + node_exporter + cadvisor is workable at small scale, but Prometheus memory grows with retention and cardinality. If you start monitoring more hosts/containers later or bump retention beyond the default 15 days, keep an eye on `docker stats` — you may need to bump the LXC to 2GB RAM or add `--storage.tsdb.retention.time=15d` explicitly to cap it.

---

## Next

→ **[05-homarr](../05-homarr/)** — one dashboard to link everything together.
