# TD Challenge 2026 — GKE Kubernetes

**Asignatura:** Tecnologías para la Digitalización  
**Universidad:** Universidad Pontificia Comillas — ICAI  
**Proyecto GCP:** `icai2026`  
**Cluster GKE:** `td-cluster` (zona: `us-east4-a`)

---

## Estructura del repositorio

```
.
├── k8s/
│   ├── challenge1-deployment.yaml   # Challenge 1: 4 réplicas nginx
│   ├── challenge2-service.yaml      # Challenge 2: LoadBalancer service
│   ├── challenge3-configmap.yaml    # Challenge 3: ConfigMap PROFESOR
│   ├── challenge4-pvc.yaml          # Challenge 4: PersistentVolumeClaim
│   ├── challenge4-deployment.yaml   # Challenge 4: Deployment final (1 réplica, PVC, ConfigMap)
│   └── nginx-pocho.yml              # Challenge 5: Pod corregido
├── .github/
│   └── workflows/
│       └── deploy.yml               # Challenge 6: GitHub Actions CI/CD
├── index.html                       # Página servida por nginx (montada en PVC)
└── README.md
```

---

## Challenge 1 — Ni una, ni dos, ni tres, sino CUATRO

**Objetivo:** Desplegar 4 réplicas de nginx con nombre `frontend-td-app`.

```bash
kubectl apply -f k8s/challenge1-deployment.yaml
kubectl get deployment frontend-td-app
```

---

## Challenge 2 — Hola Mundo

**Objetivo:** Exponer `frontend-td-app` al exterior en el puerto 80 mediante un LoadBalancer de GCP.  
**Nombre del servicio:** `frontend-app-td-service`

```bash
kubectl apply -f k8s/challenge2-service.yaml
kubectl get service frontend-app-td-service
```

La IP externa se asigna en unos minutos. Acceder en `http://<EXTERNAL-IP>`.

---

## Challenge 3 — Configurando que es gerundio

**Objetivo:** Separar configuración del código usando variables de entorno desde un ConfigMap.

ConfigMap `app-td-config` con clave `PROFESOR: "Juanvi"` inyectada como variable de entorno en el pod.

```bash
kubectl apply -f k8s/challenge3-configmap.yaml
# Verificar la variable dentro del pod:
kubectl exec -it <pod-name> -- env | grep PROFESOR
```

---

## Challenge 4 — Para toda la vida (PVC & Volumes)

**Objetivo:** Volumen persistente que sobrevive al reinicio/borrado del pod.

- PVC `frontend-td-app-pvc` (1Gi, ReadWriteOnce)
- Montado en `/usr/share/nginx/html`
- Réplicas reducidas a **1**

```bash
kubectl apply -f k8s/challenge4-pvc.yaml
kubectl apply -f k8s/challenge4-deployment.yaml

# Copiar index.html al pod:
kubectl cp index.html <pod-name>:/usr/share/nginx/html/index.html

# Borrar el pod para verificar persistencia:
kubectl delete pod <pod-name>
# Esperar a que levante el nuevo pod...
kubectl exec -it <new-pod-name> -- ls /usr/share/nginx/html/
# index.html debe seguir ahí
```

---

## Challenge 5 — Houston, tenemos un problema

**Objetivo:** Identificar y corregir por qué el pod `nginx-pocho` no arrancaba.

### Bugs encontrados en el YAML original

| Campo | Valor original (roto) | Valor corregido |
|---|---|---|
| `resources.requests.memory` | `"256Mi"` (> limits) | `"128Mi"` |
| `resources.limits.memory` | `"128Mi"` | `"256Mi"` |
| `resources.requests.cpu` | `"500"` (500 cores) | `"500m"` |

**Bug 1 — memory requests > limits:** Kubernetes rechaza pods donde `requests > limits`. La memoria de requests (256Mi) era mayor que el límite (128Mi). Se invirtieron los valores.

**Bug 2 — cpu requests `"500"`:** Sin la `m` (millicore), `"500"` significa 500 núcleos de CPU, lo que hace imposible el scheduling en cualquier nodo normal. Se corrigió a `"500m"` (0.5 CPU).

```bash
kubectl apply -f k8s/nginx-pocho.yml
kubectl get pod nginx-pocho
```

---

## Challenge 6 — A mano no se toca (GitHub Actions CI/CD)

**Objetivo:** Pipeline que se dispara en cada push a `main` y aplica los YAMLs en GKE.

### Configuración requerida en GitHub

Añadir el siguiente secret en **Settings > Secrets and variables > Actions**:

| Secret | Valor |
|---|---|
| `GCP_SA_KEY` | JSON de la Service Account con permisos `container.developer` |

### Crear la Service Account

```bash
# Crear service account
gcloud iam service-accounts create github-actions-sa \
  --display-name="GitHub Actions SA" \
  --project=icai2026

# Asignar permisos de GKE
gcloud projects add-iam-policy-binding icai2026 \
  --member="serviceAccount:github-actions-sa@icai2026.iam.gserviceaccount.com" \
  --role="roles/container.developer"

# Generar clave JSON
gcloud iam service-accounts keys create sa-key.json \
  --iam-account=github-actions-sa@icai2026.iam.gserviceaccount.com

# Copiar el contenido de sa-key.json como secret GCP_SA_KEY en GitHub
cat sa-key.json
```

### Verificación

Hacer un cambio menor (ej. editar `index.html`) y ejecutar:

```bash
git add .
git commit -m "test: trigger GitHub Actions deploy"
git push origin main
```

Ir a **Actions** en GitHub y verificar que el workflow despliega automáticamente.
