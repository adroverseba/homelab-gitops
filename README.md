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
apps/
└── habit-tracker/          una carpeta por app
    ├── 00-namespace.yaml   el prefijo numérico ordena el apply
    └── 20-frontend.yaml    ServiceAccount + Deployment + Service
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

## Secretos por app

Se crean a mano en el control-plane. Los valores están en Bitwarden,
nunca acá.

### habit-tracker

| Secret | Tipo | Para qué | Origen |
|---|---|---|---|
| `ghcr-pull` | `kubernetes.io/dockerconfigjson` | Bajar las imágenes privadas de `ghcr.io` (lo usan los ServiceAccount de la app) | Token clásico de GitHub, **solo `read:packages`**, vence el 2026-12-24 |

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
