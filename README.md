# formazione_os

Raccolta di esercizi svolti durante il percorso di formazione su **sistemi, DevOps e Kubernetes**.

## Contenuto

| Cartella | Argomento | Strumenti |
|---|---|---|
| [`CA/`](CA/) | Creazione di una Root CA e firma di certificati tramite CSR | OpenSSL |
| [`web_deployment/`](web_deployment/) | Applicazione web con Deployment, Service, Ingress e DaemonSet di logging | Kubernetes, minikube, Fluent Bit |
| [`ResourceQuota/`](ResourceQuota/) | Limiti di risorse per namespace con ResourceQuota e LimitRange | Kubernetes |
| [`network_policy/`](network_policy/) | Isolamento del traffico tra frontend, backend e database | Kubernetes, Calico |
| [`esercizio_extra/`](esercizio_extra/) | Provisioning di VM e stack di monitoring automatizzato | Ansible, Vagrant, Docker, Prometheus, Grafana, Elasticsearch |

## Requisiti generali

A seconda dell'esercizio servono:

- **OpenSSL** per la parte sui certificati
- **kubectl** e **minikube** per gli esercizi Kubernetes
- **Ansible**, **Vagrant** e **VirtualBox** per l'esercizio extra