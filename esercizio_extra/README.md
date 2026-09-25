# Esercizio extra – Stack di monitoring con Vagrant, Docker e Ansible

Questo progetto crea con Ansible **3 virtual machine Rocky Linux 9** tramite Vagrant e ci installa sopra docker ed uno stack di monitoring completo:

- **Elasticsearch** in container Docker sulla VM `elasticsearch-host`
- **Grafana** in container Docker sulla VM `monitoring-grafana`
- **Prometheus** in container Docker sulla VM `monitoring-prometheus`
- **node_exporter** installato **senza Docker** su tutte e 3 le VM, come servizio systemd **utente (non root)**

Prometheus raccoglie le metriche dei 3 node_exporter e Grafana le mostra nella dashboard **Node Exporter Full (ID 1860)**, importata in automatico.



---

## Architettura

| VM | IP | Servizio (Docker) | Porta servizio | Porta node_exporter |
|---|---|---|---|---|
| `elasticsearch-host` | 192.168.56.10 | Elasticsearch 9.5.4 | 9200 | 9100 |
| `monitoring-grafana` | 192.168.56.11 | Grafana 13.2.2 | 3000 | 9101 |
| `monitoring-prometheus` | 192.168.56.12 | Prometheus v3.14.0 | 9090 | 9102 |

```
                ┌────────────────────────────┐
                │  monitoring-prometheus     │
                │  Prometheus :9090          │──── scrape /metrics ────┐
                │  node_exporter :9102       │                         │
                └─────────────▲──────────────┘                         │
                              │ datasource                             │
                ┌─────────────┴──────────────┐          ┌──────────────▼─────────────┐
                │  monitoring-grafana        │          │  elasticsearch-host        │
                │  Grafana :3000             │          │  Elasticsearch :9200       │
                │  node_exporter :9101       │          │  node_exporter :9100       │
                └────────────────────────────┘          └────────────────────────────┘
```

Ogni node_exporter ascolta su una porta diversa (9100, 9101, 9102) e Prometheus ha 3 target statici, uno per VM.


---

## I ruoli

### `vagrant` – creazione delle VM
Gira su `localhost`. Con il modulo `community.vagrant.vagrant` genera il `Vagrantfile` e avvia le 3 VM Rocky 9 su VirtualBox, poi scrive l'inventory `inventory/vagrant.ini`, aggiorna il `ansible.cfg` e aggiunge le VM all'inventory in memoria.
Dettagli: [roles/vagrant/README.md](roles/vagrant/README.md)

### `docker` – installazione di Docker
Installa Docker Engine su tutte le VM, avvia il servizio e aggiunge l'utente `vagrant` al gruppo `docker`. Questo ruolo è documentato nella sua repo.
Dettagli: [roles/docker/README.md](roles/docker/README.md)

### `deploy` – node_exporter e servizi
Su tutte le VM installa node_exporter come servizio systemd utente (non root, con `loginctl enable-linger`). Poi, in base all'hostname, include il file di task della VM:

- `elasticsearch-host.yaml` → container Elasticsearch con heap a 1G
- `monitoring-grafana.yaml` → container Grafana, datasource Prometheus e dashboard 1860
- `monitoring-prometheus.yaml` → container Prometheus con i 3 target node_exporter

Dettagli: [roles/deploy/README.md](roles/deploy/README.md)

---

## Requisiti

- Le collection Ansible:

```bash
ansible-galaxy collection install community.vagrant community.docker community.grafana community.general ansible.posix
```

---

## Verifica

| Cosa | Dove |
|---|---|
| Elasticsearch | http://192.168.56.10:9200 |
| Grafana (admin / admin123) | http://192.168.56.11:3000 → dashboard **Node Exporter Full** |
| Prometheus – stato dei target | http://192.168.56.12:9090/targets (devono essere 3, tutti `UP`) |
| Metriche node_exporter | http://192.168.56.10:9100/metrics · http://192.168.56.11:9101/metrics · http://192.168.56.12:9102/metrics |

Heap di Elasticsearch:

```bash
curl -s "http://192.168.56.10:9200/_nodes/jvm?pretty" | grep heap_max
```