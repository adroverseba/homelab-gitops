# homelab-gitops

Manifiestos de Kubernetes del homelab: **la fuente de verdad de lo que
corre en el clúster**. La decisión y sus trade-offs están en el ADR 0005
del vault del homelab ("Repositorio GitOps de manifiestos").

## Reglas

1. **No se aplica nada que no esté commiteado.**
2. **Nunca secretos en el repo.** Se crean con `kubectl` en el
   control-plane (ver "Secretos por app"). El `.gitignore` bloquea los
   nombres de archivo típicos, pero la responsabilidad es de quien
   commitea.
3. **Imágenes por digest** (`imagen:tag@sha256:…`): lo que corre es
   exactamente lo que se eligió, aunque alguien vuelva a publicar el tag.
4. **Un namespace por app**, con Pod Security `restricted`.
5. Commits chicos, en español, con Conventional Commits y el porqué en el
   cuerpo.

## Estructura

```
apps/                         lo de cada app, en su namespace
└── habit-tracker/            una carpeta por app
    ├── 00-namespace.yaml     el prefijo numérico ordena el apply
    ├── 10-postgres.yaml      Service sin selector + EndpointSlice → VM db
    ├── 20-frontend.yaml      ServiceAccount + Deployment + Service
    ├── 30-backend.yaml       ConfigMap + ServiceAccount + Deployment + Service
    └── 40-httproute.yaml     rutas en el Gateway compartido: /api → backend, / → frontend
platform/                     lo compartido por todas las apps
├── argocd/                   Argo CD, el despliegue continuo (ADR 0010)
│   ├── 00-namespace.yaml     namespace "argocd" (restricted)
│   ├── install-v3.5.5.yaml   archivo oficial de la release (non-HA), sin tocar
│   ├── 10-project-habit-tracker.yaml  AppProject: qué puede desplegar (repo, namespace, tipos)
│   ├── 20-app-habit-tracker.yaml      Application: apps/habit-tracker/ de main
│   ├── kustomization.yaml    los cambios propios (réplicas, recursos, digests)
│   └── patch-argocd-cm.yaml  los tipos que Argo CD ignora, sin EndpointSlice
├── envoy-gateway/            controlador de la Gateway API (ADR 0006)
│   ├── install-v1.9.2.yaml   archivo oficial de la release, sin tocar
│   └── kustomization.yaml    los cambios propios (Pod Security, digest)
└── gateway/                  la entrada HTTP compartida (ADR 0006 y 0007)
    ├── 00-namespace.yaml     namespace "gateway" (restricted)
    ├── 10-gatewayclass.yaml  GatewayClass "envoy"
    ├── 20-envoyproxy.yaml    su Envoy: Service NodePort 30080 (lo usa HAProxy)
    └── 30-gateway.yaml       Gateway "homelab", HTTP :80, solo namespaces con label
```

## Cómo se aplica

### Las apps (`apps/`): las aplica Argo CD, solas

Desde el 2026-10-21, **Argo CD sincroniza `apps/habit-tracker/` solo**
(`automated` + `selfHeal`, sin `prune`; ver `platform/argocd/`):

- **Un push a `main` llega al clúster en ≤ 2–3 min**, sin ningún
  `kubectl apply`. Por eso el cambio se revisa **antes** del push.
- **Nunca `kubectl apply` ni `kubectl edit` sobre la app:** un cambio a
  mano en un campo que define git se deshace en segundos (`selfHeal`).
  Ojo: lo que se **agrega** en un campo que git no menciona no se deshace
  ni se detecta (con *server-side apply* tiene otro dueño).
- **Volver atrás:** `git revert <commit>` + push.
- **Si algo desaparece de git, no se borra del clúster** (`prune:
  false`): Argo CD lo marca y se borra a mano.
- **Emergencia:** `argocd app set habit-tracker --sync-policy none
  --insecure` apaga la sincronización (queda distinto de git: anotarlo y
  volver a encenderla con el commit que corresponda).

Para mirar qué cambiaría antes de un push (en el control-plane, con el
túnel y el login de la sección de Argo CD):

```bash
argocd app diff habit-tracker --insecure    # código 1 = hay diferencias
argocd app get habit-tracker --refresh --insecure
```

### La plataforma (`platform/`): a mano, desde el control-plane

```bash
git -C ~/homelab-gitops pull --ff-only
kubectl diff --server-side -k ~/homelab-gitops/platform/argocd/    # se revisa como un PR
kubectl apply --server-side -k ~/homelab-gitops/platform/argocd/
```

`kubectl diff` sale con código 1 cuando **hay** diferencias: es lo
esperado, no un error.

## Plataforma

### Envoy Gateway (`platform/envoy-gateway/`)

Controlador de la Gateway API: lee los `Gateway` y las `HTTPRoute` y
programa los Envoy que reciben el tráfico. Decisión y trade-offs: ADR
0006 del vault.

- **Versión:** `v1.9.2`. Es la última línea que soporta Kubernetes 1.33.
  Antes de pasar a la 1.10, hay que actualizar Kubernetes.
- **`install-v1.9.2.yaml` es el archivo oficial, sin tocar.** Su
  `sha256` es
  `0412a72907e57ff9b73c56a7bf6df5190bf0f6e4f8bb4bba34e38630bbab5778`
  (bajado el 2026-09-28). Para comprobar que es idéntico al de la
  release:

  ```bash
  curl -fsSL https://github.com/envoyproxy/gateway/releases/download/v1.9.2/install.yaml | sha256sum
  ```

- **Los cambios propios** van en `kustomization.yaml`: labels de Pod
  Security `restricted` en `envoy-gateway-system` y la imagen del
  controlador por digest.
- **Se aplica del lado del servidor y con `-k`** (Kustomize), porque los
  CRDs no entran en la anotación del *apply* clásico:

  ```bash
  kubectl kustomize ~/homelab-gitops/platform/envoy-gateway/ > /tmp/eg.yaml   # revisar lo que se va a aplicar
  kubectl apply --server-side -k ~/homelab-gitops/platform/envoy-gateway/
  ```

- **Para actualizar:** bajar el `install.yaml` nuevo con su versión en el
  nombre, anotar su `sha256` acá, cambiar `resources` y el digest en
  `kustomization.yaml`, y leer las notas de la release. Un commit por
  actualización, sin mezclarla con otros cambios.

### Argo CD (`platform/argocd/`)

Mira este repo y aplica en el clúster lo que cambia (GitOps). Decisión y
trade-offs: ADR 0010 del vault. **Al principio gestiona solo
`apps/habit-tracker/`**; la plataforma (esta carpeta incluida) sigue a
mano.

- **Versión:** `v3.5.5`, modo **non-HA** (el HA pide 3 nodos para Redis y
  hay 2 workers). La serie 3.5 está probada con Kubernetes 1.33 a 1.36.
- **`install-v3.5.5.yaml` es el archivo oficial, sin tocar.** Su `sha256`
  es
  `90dc9094bf4c5964afd9c0e35d735e1c2fcbbda4f8e36799c092d9864ed65f50`
  (bajado el 2026-10-17). Para comprobarlo:

  ```bash
  curl -fsSL https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.5/manifests/install.yaml | sha256sum
  ```

- **Los cambios propios** van en `kustomization.yaml`: el namespace con
  Pod Security `restricted`, Dex y las notificaciones en 0 réplicas (no se
  usan todavía), pedidos de recursos iniciales (estimados, a medir) y las
  tres imágenes por digest. Y `patch-argocd-cm.yaml`: la lista oficial de
  tipos que Argo CD ignora, **sin EndpointSlice** (la app usa uno para
  apuntar a la VM `db`). **Al actualizar Argo CD, comparar esa copia con
  la lista del `install.yaml` nuevo.**
- **Permisos amplios:** el controlador tiene permiso sobre todo el clúster
  (así viene el `install.yaml`). Lo que Argo CD puede desplegar se acota
  con un AppProject, pero eso no achica los permisos de su cuenta de
  servicio en Kubernetes.
- **Cada app, un AppProject con lista blanca por tipo** (el de
  habit-tracker: `10-project-habit-tracker.yaml`). Si una app empieza a
  usar un tipo nuevo, Argo CD no lo aplica hasta sumarlo al proyecto: es a
  propósito.
- **La Application no lleva el *finalizer* de borrado:** si se borra, Argo
  CD deja de mirar la app, pero la app sigue corriendo.
- **Acceso:** el usuario `admin` (contraseña en Bitwarden). Por ahora, solo
  con el CLI desde el control-plane, por un túnel:

  ```bash
  ( kubectl -n argocd port-forward svc/argocd-server 8080:443 > /tmp/argocd-pf.log 2>&1 & )
  argocd login localhost:8080 --username admin --insecure    # pide la contraseña
  ```

  `--insecure` (certificado autofirmado; el tráfico va dentro del túnel de
  `kubectl`) va en **todos** los comandos que hablan con el servidor,
  también en `logout`, o preguntan `y/n`.
- **Se aplica del lado del servidor y con `-k`:**

  ```bash
  kubectl kustomize ~/homelab-gitops/platform/argocd/ > /tmp/argocd.yaml   # revisar lo que se va a aplicar
  kubectl apply --server-side -k ~/homelab-gitops/platform/argocd/
  ```

- **Para actualizar:** igual que Envoy Gateway (archivo nuevo con la
  versión en el nombre, su `sha256` acá, los digests nuevos y las notas
  de la release), un commit por actualización.

### El Gateway compartido (`platform/gateway/`)

Una sola entrada HTTP para todas las apps: el Gateway `homelab` (namespace
`gateway`), con un Envoy que Envoy Gateway despliega en
`envoy-gateway-system` y expone en el **NodePort 30080** de los workers.
Desde la red de casa se llega por el HAProxy del host (ADR 0007).

Para publicar una app:

1. Su namespace lleva el label `homelab/gateway: allowed`. Sin ese label,
   el Gateway ignora sus rutas.
2. La app declara sus `HTTPRoute` en su carpeta, con `parentRefs` al
   Gateway `homelab` del namespace `gateway`.

Se aplica como las apps (`kubectl diff` no sirve la primera vez: los
objetos van en un namespace que todavía no existe):

```bash
kubectl apply -f ~/homelab-gitops/platform/gateway/
```

## Secretos por app

Se crean a mano en el control-plane. Los valores están en Bitwarden,
nunca acá.

### argocd (plataforma)

| Secret | Tipo | Para qué | Origen |
|---|---|---|---|
| `repo-homelab-gitops` | `Opaque`, label `argocd.argoproj.io/secret-type=repository` | Que Argo CD lea este repo (privado) por SSH | Una *deploy key* **propia de Argo CD, solo lectura** (GitHub: "argocd (homelab, solo lectura)"). La privada vive solo en el Secret |

La contraseña de `admin` no es un Secret propio: Argo CD la guarda como
*hash* en `argocd-secret`. El Secret inicial (`argocd-initial-admin-secret`)
se borró después de cambiarla.

Cómo se crea (la clave nace en una carpeta `700`, va al Secret por su ruta
y se borra del disco; sin passphrase, porque Argo CD no puede tipearla):

```bash
install -d -m 700 ~/argocd-key
ssh-keygen -q -t ed25519 -N '' -C 'argocd@homelab (solo lectura homelab-gitops)' -f ~/argocd-key/id_ed25519
cat ~/argocd-key/id_ed25519.pub    # → GitHub, Settings → Deploy keys, SIN "Allow write access"
kubectl -n argocd create secret generic repo-homelab-gitops --from-literal=type=git \
  --from-literal=url=git@github.com:adroverseba/homelab-gitops.git \
  --from-file=sshPrivateKey="$HOME/argocd-key/id_ed25519"
kubectl -n argocd label secret repo-homelab-gitops argocd.argoproj.io/secret-type=repository app.kubernetes.io/part-of=homelab-platform
shred -u ~/argocd-key/id_ed25519 && rm ~/argocd-key/id_ed25519.pub && rmdir ~/argocd-key
argocd repo list --refresh hard --insecure    # STATUS: Successful
```

Rotación: clave nueva y su *deploy key* en GitHub → `kubectl -n argocd
delete secret repo-homelab-gitops` → crearlo de nuevo → `argocd repo list`
en `Successful` → borrar la *deploy key* vieja en GitHub.

### habit-tracker

| Secret | Tipo | Para qué | Origen |
|---|---|---|---|
| `ghcr-pull` | `kubernetes.io/dockerconfigjson` | Bajar las imágenes privadas de `ghcr.io` (lo usan los ServiceAccount de la app) | Token clásico de GitHub, **solo `read:packages`**, vence el 2026-12-24 |
| `habit-tracker-backend` | `Opaque` | Variables secretas del backend: `DB_USER`, `DB_PASSWORD` y `APP_ACCESS_SECRET` | La contraseña del rol `habit_tracker` de la VM `db`, y una clave de acceso **propia del homelab** (distinta a la de producción) |

Cómo se crea `ghcr-pull` sin que el token quede en el historial, en un
archivo o en los argumentos de un proceso (`read -rs` lo lee sin
mostrarlo; `printf` es interno de bash):

```bash
read -rs GHCR_TOKEN
printf '{"auths":{"ghcr.io":{"auth":"%s"}}}' "$(printf 'adroverseba:%s' "$GHCR_TOKEN" | base64 -w0)" \
  | kubectl create secret generic ghcr-pull -n habit-tracker \
      --type=kubernetes.io/dockerconfigjson --from-file=.dockerconfigjson=/dev/stdin
unset GHCR_TOKEN
kubectl label secret ghcr-pull -n habit-tracker app.kubernetes.io/part-of=habit-tracker
```

Rotación (antes del vencimiento): token nuevo → `kubectl delete secret
ghcr-pull -n habit-tracker` → crearlo de nuevo → probar un pull →
revocar el token viejo en GitHub.

Cómo se crea `habit-tracker-backend`, con la misma técnica. Cada `read -rs`
se pega **solo** (queda esperando el valor, que no se ve). Los valores van
por *stdin* como un *env-file*: una línea `CLAVE=valor` por variable, así
que no pueden tener saltos de línea.

```bash
read -rs DB_PASSWORD
read -rs APP_ACCESS_SECRET
printf 'DB_USER=habit_tracker\nDB_PASSWORD=%s\nAPP_ACCESS_SECRET=%s\n' "$DB_PASSWORD" "$APP_ACCESS_SECRET" \
  | kubectl create secret generic habit-tracker-backend -n habit-tracker --from-env-file=/dev/stdin
unset DB_PASSWORD APP_ACCESS_SECRET
kubectl label secret habit-tracker-backend -n habit-tracker \
  app.kubernetes.io/name=habit-tracker-backend app.kubernetes.io/component=backend app.kubernetes.io/part-of=habit-tracker
```

Rotación: valor nuevo (en `db`, `\password habit_tracker`; o una clave de
acceso nueva) → `kubectl delete secret habit-tracker-backend -n
habit-tracker` → crearlo de nuevo → `kubectl rollout restart` del backend
(las variables de entorno se leen solo al arrancar).
