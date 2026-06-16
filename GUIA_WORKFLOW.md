# GitHub Actions → Docker Hub → Minikube

Guía completa para el proyecto `prueba-gha`: imagen con SHA del commit,
push a Docker Hub y deploy automático en Minikube local.

---

## Estructura de archivos relevantes

```
prueba-gha/
├── Dockerfile
├── index.html
├── .github/
│   ├── ci.yml          # Build sin push (ramas != master)
│   └── cd.yml          # Build + push + deploy (rama master)
└── .kustomize/
    ├── deployment.yaml
    ├── service.yaml
    └── kustomization.yaml
```

---

## Paso 1 — Secrets en GitHub

En tu repo: **Settings → Secrets and variables → Actions → New repository secret**

| Secret | Valor |
|---|---|
| `DOCKER_USERNAME` | Tu usuario de Docker Hub |
| `DOCKER_PASSWORD` | Token de Docker Hub (no la contraseña — ve a hub.docker.com → Account Settings → Personal access tokens) |
| `KUBE_CONFIG_DATA` | kubeconfig de Minikube en base64 (ver Paso 2) |

---

## Paso 2 — Exponer el kubeconfig de Minikube

GitHub Actions corre en runners remotos que no pueden llegar a `localhost`.
Hay dos opciones:

### Opción A — Self-hosted runner (recomendada para dev local)

1. Instala el runner en tu máquina local:
   - En GitHub: **Settings → Actions → Runners → New self-hosted runner**
   - Seguí las instrucciones que te da GitHub (descarga + `./config.sh` + `./run.sh`)

2. Iniciá el runner:
   ```bash
   ./run.sh   # o instalar como servicio: sudo ./svc.sh install && sudo ./svc.sh start
   ```

3. Cambiá `runs-on` en el job `deploy-to-cluster` de tu `cd.yml`:
   ```yaml
   runs-on: self-hosted   # en lugar de ubuntu-latest
   ```

4. Con el runner en tu máquina, `kubectl` ya puede hablar con Minikube directamente.
   No necesitás el secret `KUBE_CONFIG_DATA` en este caso.

### Opción B — Exportar kubeconfig con acceso externo (ngrok u otro tunnel)

1. Asegurate de que Minikube esté corriendo:
   ```bash
   minikube start
   ```

2. Exponé el API server de Minikube con ngrok (u otro tunnel):
   ```bash
   ngrok tcp $(minikube ip):8443
   ```
   Ngrok te va a dar una URL pública tipo `tcp://0.tcp.ngrok.io:PORT`.

3. Generá un kubeconfig que apunte a esa URL:
   ```bash
   # Exportá el kubeconfig actual
   kubectl config view --raw --minify > /tmp/kube-minikube.yaml

   # Reemplazá la IP interna por la URL pública de ngrok
   sed -i 's|https://$(minikube ip):8443|https://0.tcp.ngrok.io:PORT|g' /tmp/kube-minikube.yaml

   # Convertilo a base64 (una línea, sin saltos)
   base64 -w 0 /tmp/kube-minikube.yaml
   ```

4. Copiá ese base64 como valor del secret `KUBE_CONFIG_DATA` en GitHub.

---

## Paso 3 — Namespace en Minikube

Los manifests de Kustomize usan el namespace `prueba`. Crealo en Minikube:

```bash
kubectl create namespace prueba
```

---

## Paso 4 — El workflow CI (`.github/ci.yml`)

Se dispara en cualquier push a ramas **que no sean** `master`.
Solo **buildea** la imagen (sin push) para validar que el Dockerfile compila.

```yaml
on:
  push:
    branches-ignore:
    - master

name: CI
jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      - name: Build and (not) push
        uses: docker/build-push-action@v5
        with:
          push: false
          tags: TU_USUARIO/prueba-gha:${{ github.sha }}
```

---

## Paso 5 — El workflow CD (`.github/cd.yml`)

Se dispara solo en push a `master`. Hace build + push + deploy.

```yaml
on:
  push:
    branches:
      - master

name: CD
jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: TU_USUARIO/prueba-gha:${{ github.sha }}

  deploy-to-cluster:
    name: deploy to cluster
    runs-on: ubuntu-latest   # cambiar a self-hosted si usás Opción A
    needs: docker             # espera que el build/push termine
    steps:
      - name: deploy to cluster
        uses: steebchen/kubectl@v2.0.0
        with:
          config: ${{ secrets.KUBE_CONFIG_DATA }}
          command: -n prueba set image deployment/prueba-gha prueba-gha=TU_USUARIO/prueba-gha:${{ github.sha }}
      - name: verify deployment
        uses: steebchen/kubectl@v2.0.0
        with:
          config: ${{ secrets.KUBE_CONFIG_DATA }}
          command: -n prueba rollout status deployment/prueba-gha
```

**Qué hace cada step:**
- `needs: docker` — el deploy solo arranca si el build+push fue exitoso.
- `set image` — actualiza el tag de la imagen en el Deployment existente.
- `rollout status` — espera hasta que todos los pods del Deployment estén Running. Falla el workflow si el deploy no levanta.

---

## Paso 6 — Manifests de Kubernetes (`.kustomize/`)

### `deployment.yaml`
Define el Deployment con 3 réplicas del contenedor nginx.
La imagen base es `TU_USUARIO/prueba-gha` — el tag lo sobreescribe el comando `set image` del workflow.

### `service.yaml`
Expone el Deployment como `LoadBalancer` en el puerto 80.
En Minikube, usá `minikube tunnel` para que el LoadBalancer obtenga una IP accesible:

```bash
minikube tunnel
```

### Aplicar manualmente (primera vez)

Antes de que el workflow corra por primera vez, aplicá los manifests a mano para que el Deployment exista:

```bash
kubectl apply -k .kustomize/ -n prueba
```

---

## Paso 7 — Flujo completo end-to-end

```
developer → git push (rama feature)
              ↓
         [CI workflow]
         Build imagen (sin push)
         Verifica que el Dockerfile compila
              ↓
developer → merge a master / git push master
              ↓
         [CD workflow — Job: docker]
         Login Docker Hub
         Build imagen
         Tag: TU_USUARIO/prueba-gha:<SHA_COMMIT>
         Push a Docker Hub
              ↓
         [CD workflow — Job: deploy-to-cluster]
         kubectl set image → actualiza el tag en el Deployment
         kubectl rollout status → verifica que los pods levanten
              ↓
         Minikube (local)
         Pods se actualizan con la nueva imagen
```

---

## Verificación final en Minikube

```bash
# Ver pods actualizándose
kubectl get pods -n prueba -w

# Ver el tag de imagen que está corriendo
kubectl get deployment prueba-gha -n prueba \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Acceder al servicio (con tunnel activo)
minikube service prueba-gha -n prueba
```

---

## Resumen de secrets necesarios

| Secret | Cuándo se necesita |
|---|---|
| `DOCKER_USERNAME` | Siempre (CI y CD) |
| `DOCKER_PASSWORD` | Siempre (CI y CD) |
| `KUBE_CONFIG_DATA` | Solo en CD, y solo si usás Opción B (runner remoto) |

Con **self-hosted runner** (Opción A) no necesitás `KUBE_CONFIG_DATA` porque el
runner ya tiene acceso al cluster local.
