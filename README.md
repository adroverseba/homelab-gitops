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
├── envoy-gateway/            controlador de la Gateway API (ADR 0006)
│   ├── install-v1.9.2.yaml   archivo oficial de la release, sin tocar
│   └── kustomization.yaml    los cambios propios (Pod Security, digest)
└── gateway/                  la entrada HTTP compartida (ADR 0006 y 0007)
    ├── 00-namespace.yaml     namespace "gateway" (restricted)
    ├── 10-gatewayclass.yaml  GatewayClass "envoy"
    ├── 20-envoyproxy.yaml    su Envoy: Service NodePort 30080 (lo usa HAProxy)
    └── 30-gateway.yaml       Gateway "homelab", HTTP :80, solo namespaces con label
```

## Cómo se aplica (a mano hasta la Fase 7)

En el control-plane:

```bash
git -C ~/homelab-gitops pull --ff-only
kubectl diff -f ~/homelab-gitops/apps/habit-tracker/    # se revisa como un PR
kubectl apply -f ~/homelab-gitops/apps/habit-tracker/
```

`kubectl diff` sale con código 1 cuando **hay** diferencias: es lo
esperado, no un error. En la Fase 7, Argo CD hace el pull y el apply
solo.

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
