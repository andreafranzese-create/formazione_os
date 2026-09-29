# Esercizio 1 – Deployment, Service, ReplicaSet, Ingress e DaemonSet su Kubernetes

Questo esercizio mette in piedi, su un cluster **minikube**, un'applicazione web (nginx) in ambiente di sviluppo e un agente di logging su ogni nodo.

| File | Oggetti creati | Namespace |
|---|---|---|
| `deployment.yaml` | Deployment `web-deployment` | `default` |
| `web-service.yaml` | Service `web-service` di tipo LoadBalancer | `default` |
| `ingress.yaml` | Ingress `web-ingress` sul dominio `formazionesou.local` | `default` |
| `daemonset.yaml` | Namespace `logging`, ConfigMap `fluent-bit-config`, DaemonSet `logging-agent` | `logging` |

## Come si collega tutto

```
                      Browser / curl
                           │  http://formazionesou.local
                           ▼
              ┌─────────────────────────┐
              │  Ingress  web-ingress   │  instrada per nome di dominio
              └────────────┬────────────┘
                           ▼
              ┌─────────────────────────┐
              │ Service  web-service    │  selector: app=web, environment=dev
              │ (LoadBalancer, porta 80)│
              └────────────┬────────────┘
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          ┌─────┐       ┌─────┐       ┌─────┐
          │ pod │       │ pod │       │ pod │   nginx:1.27
          └─────┘       └─────┘       └─────┘
             ▲  gestiti dal ReplicaSet, a sua volta gestito dal Deployment
             │
             │  i log dei container finiscono su disco del nodo
             ▼
   ┌───────────────────────────────────────────┐
   │ DaemonSet logging-agent (Fluent Bit)      │  un pod per ogni nodo,
   │ legge /var/log/containers/*.log           │  raccoglie i log di tutti i pod
   └───────────────────────────────────────────┘
```

---

## 1. Deployment – `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
  labels:
    environment: dev
    app: web
    region: EU
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      environment: dev
  template:
    metadata:
      labels:
        environment: dev
        app: web
        region: EU
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            limits:
              cpu: "200m"
              memory: "256Mi"
            requests:
              cpu: "100m"
              memory: "128Mi"
```

Il Deployment dichiara "voglio sempre 3 copie di nginx attive" e si occupa di aggiornarle (rolling update) quando cambi il template.

| Parametro | Significato |
|---|---|
| `apiVersion: apps/v1` | Gruppo API delle risorse di tipo "workload" (Deployment, ReplicaSet, DaemonSet). |
| `metadata.name` | Nome del Deployment; i pod e il ReplicaSet ne erediteranno il prefisso. |
| `metadata.labels` | Labels **dell'oggetto Deployment** (`environment=dev`, `app=web`, `region=EU`), come richiesto dall'esercizio. Servono per cercarlo, es. `kubectl get deploy -l app=web`. |
| `spec.replicas: 3` | Numero di pod da tenere sempre in esecuzione. |
| `spec.selector.matchLabels` | Quali pod appartengono al Deployment. Deve essere **contenuto** nelle labels del template. Non è modificabile dopo la creazione. |
| `spec.template` | Il "modello" di ogni pod. |
| `template.metadata.labels` | Labels **dei pod**. Sono quelle che userà il Service per trovarli. |
| `containers[].name` | Nome del container dentro il pod (compare nel nome dei file di log). |
| `containers[].image` | Immagine da eseguire; la versione è fissata (`1.27`) per avere ambienti riproducibili. |
| `ports[].containerPort` | Porta su cui ascolta nginx dentro il container. |
| `resources.requests` | Risorse **riservate** dallo scheduler: `100m` = 0,1 CPU, `128Mi` di RAM. |
| `resources.limits` | Tetto massimo: oltre `200m` la CPU viene rallentata, oltre `256Mi` il container viene terminato. |

---

## 2. Service – `web-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  labels:
    app: web
    environment: dev
spec:
  type: LoadBalancer
  selector:
    app: web
    environment: dev
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
```

I pod hanno IP che cambiano ogni volta che vengono ricreati. Il Service offre un **indirizzo stabile** e distribuisce il traffico tra tutti i pod che corrispondono al suo `selector`.

| Parametro | Significato |
|---|---|
| `type: LoadBalancer` | Oltre all'IP interno (ClusterIP) chiede un **IP esterno** al cluster. In cloud è un load balancer vero; su minikube serve `minikube tunnel`, che lo espone su `127.0.0.1`. |
| `selector` | Seleziona **tutti i pod** con `app=web` **e** `environment=dev`. I pod hanno anche `region=EU`, ma non importa: basta che contengano le labels indicate. |
| `ports[].name` | Nome descrittivo della porta. |
| `ports[].port` | Porta su cui risponde il Service. |
| `ports[].targetPort` | Porta del container verso cui inoltrare (80 di nginx). |
| `metadata.labels` | Labels del Service stesso: permettono di trovarlo con `kubectl get svc -l app=web,environment=dev`. |

---

## 3. ReplicaSet

L'esercizio chiede un ReplicaSet che gestisca le repliche del Deployment, con le stesse labels.

**Il Deployment crea già il suo ReplicaSet automaticamente**, ed è quello che effettivamente gestisce le 3 repliche. Per questo non c'è un file separato: creare a mano un secondo ReplicaSet con le stesse labels nello stesso namespace lo metterebbe in conflitto con il Deployment, che proverebbe ad "adottarlo".

Il ReplicaSet generato:

- si chiama `web-deployment-<hash>` (es. `web-deployment-67887685fd`);
- eredita le labels del template (`app=web`, `environment=dev`, `region=EU`) più `pod-template-hash`, che identifica la versione del template;
- viene sostituito da uno nuovo a ogni modifica del template (è così che funziona il rolling update).

```bash
kubectl get rs --show-labels
```

```
NAME                        DESIRED   CURRENT   READY   AGE   LABELS
web-deployment-67887685fd   3         3         3       3h    app=web,environment=dev,pod-template-hash=67887685fd,region=EU
```

---

## 4. Ingress – `ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  labels:
    app: web
    environment: dev
spec:
  ingressClassName: nginx
  rules:
    - host: formazionesou.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

L'Ingress instrada il traffico HTTP in base al **nome di dominio** e al **percorso**. Un solo punto di ingresso può servire molte applicazioni.

| Parametro | Significato |
|---|---|
| `ingressClassName: nginx` | Quale Ingress Controller deve gestire questo Ingress (NGINX, abilitato con `minikube addons enable ingress`). Senza controller l'Ingress non fa nulla. |
| `rules[].host` | Dominio personalizzato. Solo le richieste con `Host: formazionesou.local` usano questa regola; le altre ricevono 404. |
| `paths[].path: /` + `pathType: Prefix` | Tutto ciò che inizia con `/`, cioè tutto il sito. |
| `backend.service.name` | Il Service di destinazione (`web-service`). |
| `backend.service.port.number` | La `port` del Service (80), non la `targetPort`. |

Per raggiungere il dominio da Mac, in `/etc/hosts`:

```
127.0.0.1  formazionesou.local
```

e in un terminale separato, lasciato aperto:

```bash
minikube tunnel
```

---

## 5. DaemonSet di logging – `daemonset.yaml`

### Cos'è un DaemonSet

Un Deployment dice "voglio N copie da qualche parte nel cluster". Un DaemonSet dice "voglio **esattamente una copia su ogni nodo**". Non ha il campo `replicas`: il numero di pod è uguale al numero di nodi. Se aggiungi un nodo, il pod ci viene creato automaticamente; se lo rimuovi, il pod sparisce.

È la scelta tipica per gli **agenti di sistema**: raccolta log, monitoring, rete. Nel nostro caso: ogni nodo scrive su disco i log dei propri container, quindi serve un agente su ogni nodo che li legga.

### Cos'è Fluent Bit

**Fluent Bit** è un raccoglitore di log open source: legge i log da una o più sorgenti, li elabora e li inoltra a una destinazione.
Lavora a pipeline:

```
INPUT  ──►  PARSER  ──►  FILTER  ──►  OUTPUT
da dove     interpreta    modifica/     dove
leggere     il formato    arricchisce   spedire
```

**Perché serve**: senza un raccoglitore i log di Kubernetes sono sparsi su ogni nodo, vengono cancellati insieme ai pod e si leggono solo con `kubectl logs` un pod alla volta. Fluent Bit li raccoglie e li può spedire a un sistema centrale (Elasticsearch, Grafana Loki, CloudWatch…). In questo esercizio li stampa semplicemente sul proprio output, così si vedono con `kubectl logs`.

### Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: logging
```

Tiene gli strumenti di sistema separati dalle applicazioni.

### ConfigMap – la configurazione di Fluent Bit

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
  labels:
    app: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush        5
        Log_Level    debug

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Exclude_Path      /var/log/containers/logging-agent-*.log
        Tag               kube.*
        multiline.parser  docker, cri
        Refresh_Interval  10

    [OUTPUT]
        Name   stdout
        Match  *
```

Un ConfigMap contiene configurazione **separata dall'immagine**: per cambiare un parametro si modifica il ConfigMap, non si ricostruisce l'immagine.

| Parametro YAML | Significato |
|---|---|
| `namespace: logging` | Deve essere lo stesso del DaemonSet: un pod può montare solo ConfigMap del proprio namespace. |
| `data.fluent-bit.conf` | Ogni chiave di `data` diventa un **file** quando il ConfigMap è montato. Qui diventa il file `fluent-bit.conf`. |
| `\|` | In YAML: testo su più righe, a capo mantenuti. |

**Sezione `[SERVICE]`** – impostazioni generali

| Parametro | Valore | Significato |
|---|---|---|
| `Flush` | `5` | Ogni 5 secondi i log accumulati nel buffer vengono inviati all'output. |
| `Log_Level` | `debug` | Verbosità dei messaggi **di Fluent Bit stesso** (non dei log raccolti). `debug` mostra quali file vengono agganciati o scartati. |

**Sezione `[INPUT]`** – da dove leggere

| Parametro | Valore | Significato |
|---|---|---|
| `Name` | `tail` | Plugin che legge file e ne segue le nuove righe, come `tail -f`. |
| `Path` | `/var/log/containers/*.log` | Quali file leggere: un file per ogni container del nodo, con nome `<pod>_<namespace>_<container>-<id>.log`. |
| `Exclude_Path` | `/var/log/containers/logging-agent-*.log` | Esclude i log di Fluent Bit stesso. Senza, Fluent Bit leggerebbe il proprio output, lo ristamperebbe, lo rileggerebbe… un **ciclo infinito**. |
| `Tag` | `kube.*` | Etichetta assegnata a ogni riga. L'`*` viene sostituito dal percorso del file (con `/` → `.`), es. `kube.var.log.containers.web-deployment-...log`. I tag servono all'OUTPUT per scegliere cosa ricevere. |
| `multiline.parser` | `docker, cri` | Il runtime non scrive il log "nudo" ma lo avvolge in un formato proprio (JSON per Docker, `timestamp stream F messaggio` per containerd/CRI). Il parser toglie l'involucro e ricompone i messaggi spezzati su più righe. |
| `Refresh_Interval` | `10` | Ogni 10 secondi riscansiona la cartella per trovare i file dei **nuovi** container. |

**Sezione `[OUTPUT]`** – dove mandare

| Parametro | Valore | Significato |
|---|---|---|
| `Name` | `stdout` | Stampa i log sull'output del container, quindi visibili con `kubectl logs`. In produzione si usa `es`, `loki`, `cloudwatch_logs`… |
| `Match` | `*` | Invia qui tutti i log il cui tag corrisponde a `*`, cioè tutti. |

### DaemonSet – parametri

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: logging-agent
  namespace: logging
  labels:
    app: logging
spec:
  selector:
    matchLabels:
      app: logging
  template:
    metadata:
      labels:
        app: logging
    spec:
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:3.1
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              memory: 128Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: dockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config
              mountPath: /fluent-bit/etc/
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: dockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: config
          configMap:
            name: fluent-bit-config
```

**Metadata e selector**

| Parametro | Significato |
|---|---|
| `metadata.labels.app: logging` | Label richiesta dall'esercizio, sull'oggetto DaemonSet. |
| `spec.selector.matchLabels` | Quali pod appartengono al DaemonSet; deve corrispondere alle labels del template. Immutabile dopo la creazione. |
| `template.metadata.labels` | Labels applicate a ogni pod creato. |
| *(nessun `replicas`)* | Il numero di pod lo decide il numero di nodi. |

**Container**

| Parametro | Significato |
|---|---|
| `image: fluent/fluent-bit:3.1` | Versione fissata, così ogni nodo usa la stessa. All'avvio l'immagine legge `/fluent-bit/etc/fluent-bit.conf`. |
| `requests.cpu: 50m` | Riserva 0,05 CPU: un agente che gira su ogni nodo deve consumare poco. |
| `requests.memory: 64Mi` | RAM riservata dallo scheduler. |
| `limits.memory: 128Mi` | Oltre questa soglia il container viene terminato e riavviato: un agente difettoso non può esaurire la memoria del nodo. |
| *(nessun limite CPU)* | Voluto: un limite stretto rallenterebbe l'agente, che resterebbe indietro coi log. |

**Volumi**

| Volume | Tipo | Montato in | Perché |
|---|---|---|---|
| `varlog` | `hostPath: /var/log` | `/var/log` (sola lettura) | Porta nel pod la cartella dei log **del nodo**, dove si trovano `/var/log/containers` e `/var/log/pods`. |
| `dockercontainers` | `hostPath: /var/lib/docker/containers` | stesso percorso (sola lettura) | Qui Docker scrive **fisicamente** i log (vedi sotto). |
| `config` | `configMap: fluent-bit-config` | `/fluent-bit/etc/` | Il ConfigMap diventa il file `/fluent-bit/etc/fluent-bit.conf`, esattamente dove l'immagine lo cerca. |

`readOnly: true`: l'agente deve solo leggere; un `hostPath` dà accesso al disco del nodo, quindi è buona pratica di sicurezza.

**Perché servono sia `/var/log` sia `/var/lib/docker/containers`**

Su minikube con runtime Docker i log sono una **catena di collegamenti simbolici**:

```
/var/log/containers/web-deployment-xxx.log            ← dove Fluent Bit cerca (Path)
      │ symlink ↓
/var/log/pods/default_web-deployment-xxx/web/0.log
      │ symlink ↓
/var/lib/docker/containers/<id>/<id>-json.log          ← il file VERO
```

Un pod vede solo le cartelle del nodo che gli vengono montate. Con solo `/var/log`, Fluent Bit trova i collegamenti ma non il file finale, e li scarta:

```
[debug] [input:tail:tail.0] skip (invalid) entry=/var/log/containers/web-deployment-67887685fd-5v9gp_default_web-cdf78d7f....log
```

Montando anche `/var/lib/docker/containers` la catena si chiude. Le cartelle vanno montate **allo stesso percorso** del nodo, perché i symlink contengono percorsi assoluti.

Sui cluster con runtime **containerd** (il caso più comune oggi) il file vero sta già in `/var/log/pods`, e basterebbe montare `/var/log`. Il runtime si vede con `kubectl get nodes -o wide` (colonna `CONTAINER-RUNTIME`).

## 6. Verifica

```bash
kubectl get deployments,services,replicasets,ingress,daemonsets -o wide
```

```
NAME                             READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES       SELECTOR
deployment.apps/web-deployment   3/3     3            3           3h    web          nginx:1.27   app=web,environment=dev

NAME                  TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE   SELECTOR
service/kubernetes    ClusterIP      10.96.0.1      <none>        443/TCP        5h    <none>
service/web-service   LoadBalancer   10.98.41.176   127.0.0.1     80:31234/TCP   3h    app=web,environment=dev

NAME                                        DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES       SELECTOR
replicaset.apps/web-deployment-67887685fd   3         3         3       3h    web          nginx:1.27   app=web,environment=dev,pod-template-hash=67887685fd

NAME                                    CLASS   HOSTS                 ADDRESS        PORTS   AGE
ingress.networking.k8s.io/web-ingress   nginx   formazionesou.local   192.168.49.2   80      3h
```

Il DaemonSet **non compare** perché il comando guarda solo il namespace `default`, mentre `logging-agent` è in `logging`:

```bash
kubectl get daemonsets -n logging -o wide
```

```
NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE   CONTAINERS   IMAGES                  SELECTOR
logging-agent   1         1         1       1            1           <none>          2h    fluent-bit   fluent/fluent-bit:3.1   app=logging
```

Oppure tutto insieme con `-A` (tutti i namespace), che però mostra anche gli oggetti di sistema.

Cosa controllare:

| Oggetto | Colonna | Atteso |
|---|---|---|
| Deployment | `READY` | `3/3` |
| Service | `TYPE` / `EXTERNAL-IP` | `LoadBalancer` / `127.0.0.1` (con tunnel attivo; senza resta `<pending>`) |
| ReplicaSet | `DESIRED / CURRENT / READY` | `3 / 3 / 3` |
| Ingress | `HOSTS` / `ADDRESS` | `formazionesou.local` / IP del nodo |
| DaemonSet | `DESIRED` = `READY` | uguale al numero di nodi (1 su minikube) |

Verifica delle labels:

```bash
kubectl get deploy,rs --show-labels
kubectl get ds -n logging --show-labels
```

```
NAME                             READY   UP-TO-DATE   AVAILABLE   AGE   LABELS
deployment.apps/web-deployment   3/3     3            3           3h    app=web,environment=dev,region=EU

NAME                                        DESIRED   CURRENT   READY   AGE   LABELS
replicaset.apps/web-deployment-67887685fd   3         3         3       3h    app=web,environment=dev,pod-template-hash=67887685fd,region=EU

NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE   LABELS
logging-agent   1         1         1       1            1           <none>          2h    app=logging
```

### Pod dell'applicazione web in ambiente dev

```bash
kubectl get pods -l app=web,environment=dev
```

```
NAME                              READY   STATUS    RESTARTS   AGE
web-deployment-67887685fd-5v9gp   1/1     Running   0          3h
web-deployment-67887685fd-nhq9k   1/1     Running   0          3h
web-deployment-67887685fd-t98kg   1/1     Running   0          3h
```

3 pod `Running`, con il nome composto da `<deployment>-<hash del ReplicaSet>-<suffisso>`. Sono gli stessi pod che il Service seleziona:

```bash
kubectl get endpoints web-service
```

```
NAME          ENDPOINTS                                      AGE
web-service   10.244.0.12:80,10.244.0.13:80,10.244.0.14:80   3h
```

### Accesso tramite il dominio dell'Ingress

Dal browser: **http://formazionesou.local** → pagina "Welcome to nginx!".

Da terminale:

```bash
curl -H "Host: formazionesou.local" http://127.0.0.1
```

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

### Verifica del DaemonSet di logging

Si fa una richiesta con un percorso riconoscibile; nginx la scrive nel proprio log di accesso e Fluent Bit deve raccoglierla.

```bash
curl -H "Host: formazionesou.local" http://127.0.0.1/test-1
sleep 6        # attende il Flush di 5 secondi
kubectl logs -n logging -l app=logging --tail=-1 | grep test-1
```

La richiesta riceve 404 perché la pagina non esiste, ma la riga `GET /test-1` dimostra che il log è stato raccolto.

Controllo che nessun file venga scartato:

```bash
kubectl logs -n logging -l app=logging --tail=-1 | grep "skip (invalid)" | grep web-deployment
```

Nessun output = tutti i file dei pod web sono stati agganciati correttamente.

---