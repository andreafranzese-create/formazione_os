# Kubernetes Network Policy – Frontend / Backend / Database

Esercizio sulle **NetworkPolicy**: tre applicazioni minimali in un namespace dedicato (`netpol`) e regole di rete che permettono solo i flussi necessari.

```
Utente ──TCP 80──▶ frontend ──TCP 80──▶ backend ──TCP 5432 ──▶ database
                       │                   │                 (nessun egress)
                       └──── UDP/TCP 53 ───┴──▶ CoreDNS (kube-system)
```

## Prerequisiti

Le NetworkPolicy vengono applicate dal **plugin di rete (CNI)**, non da Kubernetes. Se il CNI non le supporta, vengono create ma **ignorate** senza errori.

```bash
minikube start --cni=calico
```

## File

| File | Contenuto |
|---|---|
| `frontend.yaml` | Deployment nginx (`app=frontend`) + Service **NodePort** 30080, esposto all'esterno |
| `backend.yaml` | Deployment nginx (`app=backend`) + Service ClusterIP :80 |
| `database.yaml` | Deployment Postgres (`app=database`) + Service ClusterIP :5432 |
| `default-deny.yaml` | Blocca **tutto** il traffico, in ingresso e in uscita, per ogni pod del namespace |
| `frontend-policy.yaml` | Ingress: TCP 80 dall'esterno. Egress: backend :80 + DNS |
| `backend-policy.yaml` | Ingress: solo dal frontend su :80. Egress: database :5432 + DNS |
| `database-policy.yaml` | Ingress: solo dal backend su :5432. Egress: nessuno |

Frontend e backend sono due nginx che rispondono con un testo riconoscibile (`Ciao dal FRONTEND` / `Ciao dal BACKEND`).

## Esecuzione

```bash
# 1. Namespace (e impostalo come default)
kubectl create namespace netpol
kubectl config set-context --current --namespace=netpol

# 2. Applicazioni (i file non hanno namespace: -n è importante)
kubectl apply -n netpol -f database.yaml -f backend.yaml -f frontend.yaml

# 3. Default deny (le policy hanno già namespace: netpol)
kubectl apply -f default-deny.yaml

# 4. Policy specifiche
kubectl apply -f frontend-policy.yaml -f backend-policy.yaml -f database-policy.yaml
```
## Anatomia di una NetworkPolicy

Come esempio prendiamo `backend-policy.yaml`, che contiene tutti i pezzi principali.

### 1. Intestazione

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: netpol
```

- `apiVersion` / `kind`: il tipo di risorsa. Le NetworkPolicy stanno nel gruppo API `networking.k8s.io`.
- `namespace`: la policy vale **solo** per i pod di questo namespace.

### 2. `podSelector`: a chi si applica

```yaml
spec:
  podSelector:
    matchLabels:
      app: backend
```

Sceglie i pod **protetti** da questa policy, in base alle loro label. Le regole che seguono descrivono il traffico che entra ed esce da **questi** pod.
`podSelector: {}` (vuoto) significa *tutti i pod del namespace*: è il trucco usato in `default-deny.yaml`.

### 3. `policyTypes`: quali direzioni isolare

```yaml
  policyTypes:
    - Ingress   # traffico in entrata verso i pod selezionati
    - Egress    # traffico in uscita dai pod selezionati
```

Per ogni direzione elencata, i pod diventano **isolati**: passa solo ciò che è scritto nella sezione corrispondente. Se una direzione elencata non ha regole, è **tutto bloccato**.

| Scrittura | Effetto |
|---|---|
| `Egress` in `policyTypes`, nessuna sezione `egress:` | Blocca tutto in uscita |
| `egress: [{}]` | Consente tutto in uscita |
| `egress:` con `to:` e `ports:` | Consente solo quelle destinazioni su quelle porte |

### 4. `ingress`: chi può entrare

```yaml
  ingress:
    - from:                     # DA chi
        - podSelector:
            matchLabels:
              app: frontend
      ports:                    # SU quali porte
        - protocol: TCP
          port: 80
```

"Accetta connessioni **dai pod `app=frontend`** sulla **porta TCP 80**." `from` e `ports` della stessa regola sono in **AND**. Se `ports` manca, sono consentiti tutti i protocolli e tutte le porte.

### 5. `egress`: verso chi si può uscire

```yaml
  egress:
    - to:                       # regola 1: database
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    - to:                       # regola 2: DNS
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

Ogni `- to:` è una **regola**, e le regole tra loro sono in **OR**: il backend può andare al database sulla 5432 **oppure** a CoreDNS sulla 53.

### 6. I selettori dentro `from` / `to`

| Selettore | Seleziona |
|---|---|
| `podSelector` | Pod con quelle label **nello stesso namespace** della policy |
| `namespaceSelector` | Tutti i pod dei namespace con quelle label |
| `namespaceSelector` + `podSelector` nello **stesso elemento** | Pod con quelle label **dentro** quei namespace (AND) |
| `ipBlock` (`cidr`, `except`) | Intervalli di IP, pensati per indirizzi esterni al cluster (usato in `frontend-policy.yaml` con `0.0.0.0/0`) |

Il trattino conta:

```yaml
# AND: un elemento → "pod kube-dns che stanno in kube-system"
- namespaceSelector: {...}
  podSelector: {...}

# OR: due elementi → "qualsiasi pod di kube-system" OPPURE "pod kube-dns del mio namespace"
- namespaceSelector: {...}
- podSelector: {...}
```

### 7. `ports`

- `protocol`: `TCP` (default), `UDP` o `SCTP`. **ICMP non è previsto**, per questo il ping non passa.
- `port`: la porta del **container** (la `targetPort`), non quella del Service. Può essere anche il nome della porta.
- `endPort` (facoltativo): per indicare un intervallo, ad esempio `port: 8000` + `endPort: 8080`.

## Verifica

```bash
kubectl get networkpolicies
kubectl describe networkpolicy backend-policy
```

**Traffico applicativo (TCP):**

```bash
kubectl exec deploy/frontend -- wget -qO- -T 3 http://backend        # ✅ Ciao dal BACKEND
kubectl exec deploy/backend  -- nc -zv -w 3 database 5432           # ✅ open
kubectl exec deploy/frontend -- nc -zv -w 3 database 5432           # ⛔ timeout
kubectl exec deploy/backend  -- wget -qO- -T 3 http://frontend      # ⛔ timeout
kubectl exec deploy/database -- wget -qO- -T 3 http://example.com   # ⛔ bad address
kubectl exec deploy/database -- nslookup backend                    # ⛔ timeout
```

**Accesso dall'esterno al frontend:**

```bash
curl $(minikube service frontend -n netpol --url)                    # ✅ Ciao dal FRONTEND
```

Usa la NodePort e non `kubectl port-forward`: il port-forward passa dal kubelet e non è soggetto alle NetworkPolicy.

**Ping (ICMP):**

```bash
FE_IP=$(kubectl get pod -n netpol -l app=frontend -o jsonpath='{.items[0].status.podIP}')
BE_IP=$(kubectl get pod -n netpol -l app=backend  -o jsonpath='{.items[0].status.podIP}')
DB_IP=$(kubectl get pod -n netpol -l app=database -o jsonpath='{.items[0].status.podIP}')

kubectl exec deploy/frontend -- ping -c 2 -W 2 $BE_IP
kubectl exec deploy/backend  -- ping -c 2 -W 2 $DB_IP
```

Risultato: **tutti i ping falliscono (100% packet loss)**, ed è corretto. L'API NetworkPolicy permette di indicare solo TCP, UDP e SCTP. Tutte le regole qui hanno porte TCP/UDP, quindi ICMP non rientra in nessuna eccezione. Per farlo passare servirebbe una regola **senza `ports:`**, che però apre tutti i protocolli e tutte le porte tra quei pod.

### Risultati attesi

| Da → A | Prima delle policy | Dopo |
|---|---|---|
| esterno → frontend :80 | ✅ | ✅ |
| frontend → backend :80 | ✅ | ✅ |
| backend → database :5432 | ✅ | ✅ |
| frontend → database | ✅ | ⛔ |
| backend → frontend | ✅ | ⛔ |
| database → chiunque / internet | ✅ | ⛔ |
| ping tra qualsiasi pod | ✅ | ⛔ |