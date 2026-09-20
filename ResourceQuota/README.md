# ResourceQuota
---
## Il manifest del deployment di test

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels: {app: nginx}
spec:
  replicas: 3
  selector:
    matchLabels: {app: nginx}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels: {app: nginx}
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
          - containerPort: 8080
        resources:
          limits:
            cpu: "100m"
            memory: "64Mi"
```

## Test A — numero di pod

Manifest del ResouceQuota:

```bash
apiVersion: v1
kind: ResourceQuota
metadata:
  name: t4-quota
  namespace: t4
spec:
  hard:
    requests.cpu: "500m"
    requests.memory: 512Mi
    limits.cpu: "1"
    limits.memory: 1Gi
    pods: "2"
```

Utilizziamo i comandi:
```bash
kubectl apply -f quota.yaml        # applica la quota
kubectl apply -f deployment.yaml   # applica il deployment
kubctl get deployment              # verifichiamo i pod ready
```
output della verifica:
```bash
NAME    READY   UP-TO-DATE   AVAILABLE
nginx   2/3     2            2
```

**`kubectl apply` non fallisce.** Il Deployment non consuma `pods`, passa. A sbattere contro la quota è il ReplicaSet:

```bash
kubectl describe rs -l app=nginx | grep -A 3 FailedCreate
```

output:

```
Error creating: pods "nginx-6f8b...-p7v2n" is forbidden: exceeded quota: t4-pods,
requested: pods=1, used: pods=2, limited: pods=2
```

---

## Test B — CPU e memoria

Manifest del pod con cui testiamo lo sforo dei limiti:

```bash
apiVersion: v1
kind: Pod
metadata:
  name: test
  namespace: t4
  labels:
    test: quota
spec:
  restartPolicy: Never
  containers:
    - name: test
      image: nginx:1.27
      resources:
        limits:
          cpu: "600m"
          memory: "256Mi"
```

Se applichiamo il manifest esce come output:

```bash
exceeded quota: t4-quota, requested: requests.cpu=600m, used: requests.cpu=0,
limited: requests.cpu=500m
```

Il limite può anche sforare con più pod nello stesso namespace:

```bash
apiVersion: v1
kind: Pod
metadata:
  name: test1
  namespace: t4
  labels: {test: quota}
spec:
  restartPolicy: Never
  containers:
  - name: test1
    image: nginx:1.27
    resources:
      limits: {cpu: "200m", memory: "64Mi"}
---
apiVersion: v1
kind: Pod
metadata:
  name: test2
  namespace: t4
  labels: {test: quota}
spec:
  restartPolicy: Never
  containers:
  - name: test2
    image: nginx:1.27
    resources:
      limits: {cpu: "200m", memory: "64Mi"}
---
apiVersion: v1
kind: Pod
metadata:
  name: test3
  namespace: t4
  labels: {test: quota}
spec:
  restartPolicy: Never
  containers:
  - name: test3
    image: nginx:1.27
    resources:
      limits: {cpu: "200m", memory: "64Mi"}
```

Quando si applicano questi manifest danno come output:

```bash
pod/acc1 created
pod/acc2 created
Error from server (Forbidden): error when creating "pod.yaml": pods "acc3" is
forbidden: exceeded quota: t4-quota, requested: requests.cpu=200m,
used: requests.cpu=400m, limited: requests.cpu=500m
```

---

## Test C — deadlock del RollingUpdate

Quota dimensionata **esattamente** su 3 repliche:

```bash
apiVersion: v1
kind: ResourceQuota
metadata: {name: t4-tight, namespace: t4}
spec:
  hard:
    pods: "3"
    requests.cpu: "300m"
    limits.cpu: "300m"
    requests.memory: 192Mi
    limits.memory: 192Mi
```

```bash
kubectl apply -f deployment.yaml                         # 3/3, quota satura
kubectl set image deploy/nginx nginx=nginx:1.27-alpine
kubectl rollout status deploy/nginx --timeout=60s.       # eseguiamo il rollout
```

output:

```
error: timed out waiting for the condition
```

**Perché.** Con `maxUnavailable: 0` non si può terminare un pod vecchio. Con `maxSurge: 1` bisogna un quarto ma `pods: 3` è già saturo → 403. Nessuna mossa possibile, stallo permanente.

Per sbloccarlo si può aumentare il numero di pod massimi direttamente nella quota.

**Alternative se la quota non è modificabile:**

| Opzione | Costo |
|---|---|
| `maxSurge: 0` + `maxUnavailable: 1` | durante il rollout scendi a 2/3 di capacità |
| `kubectl rollout undo` | torna al RS vecchio (già in esecuzione, non serve creare nulla) |

---

## Test D — risorsa non dichiarata

Manifest pod di test:

```bash
apiVersion: v1
kind: Pod
metadata:
  name: test
  namespace: t4
spec:
  restartPolicy: Never
  containers:
  - name: test
    image: nginx:1.27
```

Se applichiamo il pod su un namespace con una resourcequota da come output:

```bash
Error from server (Forbidden): error when creating "pod.yaml": pods "test" is
forbidden: failed quota: t4-quota: must specify limits.cpu for: test;
limits.memory for: test; requests.cpu for: test; requests.memory for: test
```

---

## LimitRange

Risolve il test D iniettando default a livello di namespace:

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: LimitRange
metadata: {name: t4-limits, namespace: t4}
spec:
  limits:
  - type: Container
    default:        {cpu: "200m", memory: 128Mi}   # → limits
    defaultRequest: {cpu: "100m", memory: 64Mi}    # → requests
    max:            {cpu: "1",    memory: 512Mi}
    min:            {cpu: "10m",  memory: 16Mi}
EOF
```

Il pod viene accettato con le risorse iniettate, pur non avendole dichiarate.