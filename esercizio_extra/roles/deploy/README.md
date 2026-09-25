# Ruolo `deploy`

Installa **node_exporter** su tutte le VM e poi, in base all'hostname, deploya il servizio della VM in un container Docker:

| VM | File di task incluso | Cosa viene deployato |
|---|---|---|
| `elasticsearch-host` | `tasks/elasticsearch-host.yaml` | Elasticsearch con heap a 1G |
| `monitoring-grafana` | `tasks/monitoring-grafana.yaml` | Grafana con datasource Prometheus e dashboard Node Exporter Full |
| `monitoring-prometheus` | `tasks/monitoring-prometheus.yaml` | Prometheus che fa scraping dei 3 node_exporter |

---

## Task comuni – `tasks/main.yml` (tutte le VM)

Qui viene installato node_exporter **senza Docker** e **senza root**: il binario sta in `/usr/local/bin`, ma il servizio è un servizio systemd **utente** di `vagrant`.

| # | Task | Modulo | Cosa fa |
|---|---|---|---|
| 1 | **Installa la libreria requests per i moduli Docker** | `ansible.builtin.dnf` | Installa `python3-requests` sulla VM. I moduli `community.docker.*` parlano con il demone Docker tramite questa libreria. |
| 2 | **Directory dei servizi utente** | `ansible.builtin.file` | Crea `/home/<utente>/.config/systemd/user`, cartella in cui systemd cerca le unit dell'utente. Proprietario: `node_exporter_user`. |
| 3 | **Scarica ed estrai node_exporter** | `ansible.builtin.unarchive` | Scarica il tar.gz da GitHub (`node_exporter_url`) e lo estrae direttamente sulla VM in `/tmp` (`remote_src: true`). Con `creates` non riscarica se il binario estratto c'è già. |
| 4 | **Copia il binario** | `ansible.builtin.copy` | Copia `node_exporter` in `node_exporter_bin` (`/usr/local/bin/node_exporter`), eseguibile (`0755`). `setype: bin_t` imposta il contesto SELinux corretto: un file copiato da `/tmp` avrebbe contesto `tmp_t` e systemd non riuscirebbe ad eseguirlo. Se il binario cambia, notifica il riavvio. |
| 5 | **Servizio utente di node_exporter** | `ansible.builtin.template` | Scrive `node_exporter.service` in `~/.config/systemd/user/` dal template `node_exporter.service.j2`, con la porta della VM. Se cambia, notifica il riavvio. |
| 6 | **Mantieni attivo il systemd dell'utente anche senza login** | `ansible.builtin.command` | Esegue `loginctl enable-linger <utente>`. Senza linger i servizi utente partono solo quando l'utente fa login e si fermano al logout; con linger partono al boot. `creates` rende il task idempotente (il file `/var/lib/systemd/linger/<utente>` esiste già se il linger è attivo). |
| 7 | **Avvia e abilita node_exporter** | `ansible.builtin.systemd_service` | Con `scope: user` e `become_user: <utente>` fa `daemon-reload`, avvia il servizio e lo abilita all'avvio. È l'equivalente di `systemctl --user enable --now node_exporter` lanciato come `vagrant`. |
| 8 | **Includi task in base all'hostname** | `ansible.builtin.include_tasks` | Include `{{ inventory_hostname }}.yaml`, cioè il file di task con lo stesso nome della VM. |

---

## Task per `elasticsearch-host` – `tasks/elasticsearch-host.yaml`

| # | Task | Modulo | Cosa fa |
|---|---|---|---|
| 1 | **Apri le porte nel firewall** | `ansible.posix.firewalld` | Apre in modo permanente e immediato le porte di `elasticsearch_ports`: 9200 (Elasticsearch) e 9100 (node_exporter). |
| 2 | **Directory per le opzioni JVM** | `ansible.builtin.file` | Crea sulla VM `elasticsearch_jvm_options_dir` (`/opt/elasticsearch/jvm.options.d`). |
| 3 | **File con l'heap di Elasticsearch** | `ansible.builtin.template` | Genera `heap.options` dal template `heap.j2` con `-Xms1g` e `-Xmx1g`. Se cambia, riavvia il container. |
| 4 | **Scarica l'immagine di Elasticsearch** | `community.docker.docker_image` | Fa il pull di `elasticsearch_image:elasticsearch_tag`. |
| 5 | **Crea il volume per i dati di Elasticsearch** | `community.docker.docker_volume` | Crea il volume Docker `elasticsearch_volume` per avere dati persistenti. |
| 6 | **Avvia il container Elasticsearch** | `community.docker.docker_container` | Avvia il container con `restart_policy: unless-stopped` e porta 9200 pubblicata. Monta il volume dati in `/usr/share/elasticsearch/data` e il file `heap.options` in sola lettura in `/usr/share/elasticsearch/config/jvm.options.d/`, come chiede la [documentazione Elastic](https://www.elastic.co/docs/reference/elasticsearch/jvm-settings). Variabili d'ambiente: `discovery.type=single-node` (nodo singolo, nessun cluster) e `xpack.security.enabled=false` (niente TLS e password, così Elasticsearch risponde in HTTP semplice). |

---

## Task per `monitoring-grafana` – `tasks/monitoring-grafana.yaml`

| # | Task | Modulo | Cosa fa |
|---|---|---|---|
| 1 | **Apri la porta nel firewall** | `ansible.posix.firewalld` | Apre le porte di `grafana_ports`: 3000 (Grafana) e 9101 (node_exporter). |
| 2 | **Directory di provisioning delle datasource** | `ansible.builtin.file` | Crea `<grafana_provisioning_dir>/datasources` sulla VM. |
| 3 | **File di provisioning delle datasource** | `ansible.builtin.template` | Genera `datasources.yaml` dal template: Grafana, all'avvio, crea da solo la datasource Prometheus (così non va aggiunta a mano). Se cambia, riavvia il container. |
| 4 | **Scarica l'immagine di Grafana** | `community.docker.docker_image` | Pull di `grafana_image:grafana_tag`. |
| 5 | **Crea il volume per i dati di Grafana** | `community.docker.docker_volume` | Crea il volume persistente `grafana_volume`. |
| 6 | **Avvia il container Grafana** | `community.docker.docker_container` | Avvia Grafana sulla porta 3000. Monta il volume in `/var/lib/grafana` (database, dashboard, utenti) e la cartella di provisioning in `/etc/grafana/provisioning/datasources` (sola lettura). Utente e password admin arrivano dalle variabili `GF_SECURITY_ADMIN_USER` / `GF_SECURITY_ADMIN_PASSWORD`. |
| 7 | **Attendi che Grafana risponda** | `ansible.builtin.uri` | Interroga `http://localhost:3000/api/health` fino a ricevere `200` (massimo 30 tentativi ogni 5 secondi). Serve perché il container impiega qualche secondo ad avviarsi e il task successivo fallirebbe. |
| 8 | **Importa le dashboard da grafana.com** | `community.grafana.grafana_dashboard` | Scarica da grafana.com la dashboard `grafana_dashboard_id` (1860, Node Exporter Full) nella revisione `grafana_dashboard_revision` e la importa tramite API con le credenziali admin. `overwrite: true` la sovrascrive se esiste già. |

---

## Task per `monitoring-prometheus` – `tasks/monitoring-prometheus.yaml`

| # | Task | Modulo | Cosa fa |
|---|---|---|---|
| 1 | **Apri la porta nel firewall** | `ansible.posix.firewalld` | Apre le porte di `prometheus_ports`: 9090 (Prometheus) e 9102 (node_exporter). |
| 2 | **Directory di configurazione di Prometheus** | `ansible.builtin.file` | Crea `prometheus_config_dir` (`/opt/prometheus`). |
| 3 | **File di configurazione di Prometheus** | `ansible.builtin.template` | Genera `prometheus.yaml` dal template con i 3 target node_exporter. Se cambia, riavvia il container. |
| 4 | **Scarica l'immagine di Prometheus** | `community.docker.docker_image` | Pull di `prometheus_image:prometheus_tag`. |
| 5 | **Crea il volume per i dati di Prometheus** | `community.docker.docker_volume` | Crea il volume persistente `prometheus_volume`. |
| 6 | **Avvia il container Prometheus** | `community.docker.docker_container` | Avvia Prometheus sulla porta 9090. Monta il volume in `/prometheus` (database TSDB) e la cartella di config in `/etc/prometheus` (sola lettura). Parametri di avvio: `--config.file` (il file generato), `--storage.tsdb.path=/prometheus` e `--storage.tsdb.retention.time` (per quanto tempo conservare le metriche). |

---

## Handler – `handlers/main.yml`

| Handler | Modulo | Cosa fa | Notificato da |
|---|---|---|---|
| **Riavvia Elasticsearch** | `community.docker.docker_container` | Riavvia il container `<elasticsearch_container_name>` (`state: started` + `restart: true`) | modifica di `heap.options` |
| **Riavvia Grafana** | `community.docker.docker_container` | Riavvia il container `<grafana_container_name>` (`state: started` + `restart: true`) | modifica di `datasources.yaml` |
| **Riavvia Prometheus** | `community.docker.docker_container` | Riavvia il container `<prometheus_container_name>` (`state: started` + `restart: true`) | modifica di `prometheus.yaml` |
| **Riavvia node_exporter** | `ansible.builtin.systemd_service` | `daemon-reload` e restart del servizio utente (`scope: user`), eseguito come `node_exporter_user` | modifica del binario o della unit |

---

## Template – `templates/`

### `heap.j2` → `heap.options`
```
-Xms{{ elasticsearch_heap }}
-Xmx{{ elasticsearch_heap }}
```
Heap minimo e massimo della JVM uguali (1g), come raccomandato da Elastic.

### `node_exporter.service.j2` → `~/.config/systemd/user/node_exporter.service`
Unit systemd utente. `ExecStart` lancia il binario con `--web.listen-address=:<porta>`, dove la porta viene presa da `node_exporter_ports[inventory_hostname]` (quindi è diversa su ogni VM). `Restart=on-failure` lo riavvia se va in crash. `WantedBy=default.target` è il target dei servizi utente.

### `prometheus.yaml.j2` → `/opt/prometheus/prometheus.yaml`
- `global.scrape_interval` → ogni quanto Prometheus legge le metriche.
- Un job `node` con **3 target statici**, uno per VM, ognuno con la label `vm` (nome della VM) per distinguerli nelle query e in Grafana. Prometheus legge l'endpoint `/metrics` di default.

La porta di ogni target è il secondo elemento della lista `*_ports` della VM (`elasticsearch_ports[1]`, `grafana_ports[1]`, `prometheus_ports[1]`).

### `datasources.yaml.j2` → `/opt/grafana/provisioning/datasources/datasources.yaml`
File di provisioning di Grafana che crea la datasource:

| Campo | Significato |
|---|---|
| `name: Prometheus` | Nome mostrato in Grafana. |
| `type: prometheus` | Tipo di datasource. |
| `uid: prometheus` | ID fisso, così le dashboard possono riferirsi alla datasource in modo stabile. |
| `access: proxy` | Le query partono dal server Grafana, non dal browser. |
| `url` | Indirizzo di Prometheus (`grafana_prometheus_url`). |
| `isDefault: true` | Datasource predefinita. |
| `editable: false` | Non modificabile dalla UI (è gestita da Ansible). |

---

## Variabili (`defaults/main.yml`)

### Elasticsearch

| Variabile | Default | Significato |
|---|---|---|
| `elasticsearch_ip` | `192.168.56.10` | IP della VM Elasticsearch, usato da Prometheus come target. |
| `elasticsearch_volume` | `elasticsearch-data` | Nome del volume Docker per i dati. |
| `elasticsearch_image` | `docker.elastic.co/elasticsearch/elasticsearch` | Immagine Docker ufficiale. |
| `elasticsearch_tag` | `"9.5.4"` | Versione dell'immagine. |
| `elasticsearch_container_name` | `elasticsearch` | Nome del container (usato anche dall'handler). |
| `elasticsearch_ports` | `[ 9200, 9100 ]` | `[0]` = porta di Elasticsearch, `[1]` = porta di node_exporter su questa VM. Entrambe aperte nel firewall. |
| `elasticsearch_heap` | `1g` | Dimensione dell'heap JVM (`-Xms` e `-Xmx`). |
| `elasticsearch_jvm_options_dir` | `/opt/elasticsearch/jvm.options.d` | Cartella sulla VM dove viene scritto `heap.options`. |

### Grafana

| Variabile | Default | Significato |
|---|---|---|
| `grafana_ip` | `192.168.56.11` | IP della VM Grafana, usato da Prometheus come target. |
| `grafana_image` | `grafana/grafana` | Immagine Docker. |
| `grafana_tag` | `"13.2.2"` | Versione dell'immagine. |
| `grafana_container_name` | `grafana` | Nome del container. |
| `grafana_ports` | `[ 3000, 9101 ]` | `[0]` = porta di Grafana, `[1]` = porta di node_exporter su questa VM. |
| `grafana_volume` | `grafana-data` | Volume persistente montato in `/var/lib/grafana`. |
| `grafana_provisioning_dir` | `/opt/grafana/provisioning` | Cartella sulla VM con i file di provisioning. |
| `grafana_admin_user` | `admin` | Utente admin di Grafana (usato anche per importare la dashboard). |
| `grafana_admin_password` | `admin123` | Password admin. In un ambiente reale va messa in Ansible Vault. |
| `grafana_prometheus_url` | `"http://{{ prometheus_ip }}:9090"` | URL della datasource Prometheus. |
| `grafana_dashboard_id` | `1860` | ID della dashboard su grafana.com (Node Exporter Full). |
| `grafana_dashboard_revision` | `45` | Revisione della dashboard da scaricare. |

### Prometheus

| Variabile | Default | Significato |
|---|---|---|
| `prometheus_ip` | `192.168.56.12` | IP della VM Prometheus (target e URL della datasource). |
| `prometheus_image` | `prom/prometheus` | Immagine Docker. |
| `prometheus_tag` | `"v3.14.0"` | Versione dell'immagine. |
| `prometheus_container_name` | `prometheus` | Nome del container. |
| `prometheus_ports` | `[ 9090, 9102 ]` | `[0]` = porta di Prometheus, `[1]` = porta di node_exporter su questa VM. |
| `prometheus_volume` | `prometheus-data` | Volume persistente montato in `/prometheus`. |
| `prometheus_config_dir` | `/opt/prometheus` | Cartella sulla VM con `prometheus.yaml`. |
| `prometheus_scrape_interval` | `15s` | Intervallo di scraping. |
| `prometheus_retention` | `15d` | Per quanto tempo Prometheus conserva le metriche. |

### node_exporter

| Variabile | Default | Significato |
|---|---|---|
| `node_exporter_version` | `"1.12.1"` | Versione da scaricare. |
| `node_exporter_arch` | `amd64` | Architettura del pacchetto. |
| `node_exporter_user` | `vagrant` | Utente non root che esegue il servizio systemd utente. |
| `node_exporter_bin` | `/usr/local/bin/node_exporter` | Dove viene installato il binario. |
| `node_exporter_url` | URL della release GitHub | Costruito da versione e architettura: `.../v1.12.1/node_exporter-1.12.1.linux-amd64.tar.gz`. |
| `node_exporter_ports` | `elasticsearch-host: 9100`<br>`monitoring-grafana: 9101`<br>`monitoring-prometheus: 9102` | Porta di ascolto di node_exporter per ogni VM (chiave = `inventory_hostname`). Usata nella unit systemd. |
| `node_exporter_options_elasticsearch`<br>`node_exporter_options_grafana`<br>`node_exporter_options_prometheus` | `"--web.listen-address=:<porta>"` | Opzioni di avvio per VM. **Al momento non sono usate** da nessun task o template (la unit costruisce l'opzione da `node_exporter_ports`). |