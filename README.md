# creative-ideas — manifiestos Kustomize

Manifiestos de Kubernetes (Kustomize) para desplegar la app **Creative
Ideas API** (código fuente en [`app/`](app/), API REST en Go con
arquitectura hexagonal — ver [`app/README.md`](app/README.md)) en tres
entornos, todos a partir de la **misma imagen de contenedor**
(`ghcr.io/ags-devops/creative-ideas-api`), cambiando solo configuración vía
variables de entorno:

| Entorno | Réplicas API | Base de datos              | Namespace              |
|---------|:---:|----------------------------------------|--------------------------|
| dev     | 1   | SQLite (archivo en PVC del propio pod) | `ideas-creativas-dev`     |
| test    | 3   | MariaDB, 1 sola instancia               | `ideas-creativas-test`    |
| prod    | 5   | MariaDB, 1 sola instancia               | `ideas-creativas-prod`    |

`DB_DRIVER` (sqlite/mysql) se elige por variable de entorno en cada overlay
de `web/` (ver `web/overlays/*/kustomization.yaml`); es el mismo binario de
la API el que decide qué adaptador usar al arrancar. En los tres entornos
se usa Valkey (compatible con el protocolo Redis) como acelerador de caché
para el listado de ideas (patrón cache-aside).

**Nombres:** los recursos de Kubernetes están en español. Los Services son
`ideas-creativas-api`, `ideas-creativas-cache` y, en test y prod,
`mariadb-test` o `mariadb-prod`. En test y prod la base de datos MariaDB se
llama `ideas_creativas`: la crea la imagen de MariaDB al arrancar con el
volumen vacío (a partir de `MARIADB_DATABASE`, en `db/overlays/*`) y la API la
usa mediante `DB_DSN` (en `web/overlays/*`). Como el nombre vive en dos
componentes distintos, ambos valores deben coincidir.

## Estructura del repositorio

La app tiene tres componentes independientes — **web** (la API), **cache**
(Valkey) y **db** (MariaDB, solo en test/prod) — y cada uno se despliega y
personaliza por separado con su propio `base/` + `overlays/{dev,test,prod}`.
No hay un overlay "raíz" que los junte: cada componente incluye su propio
`Namespace` y lo declara en su `kustomization.yaml` (`namespace: ideas-creativas-dev`,
etc.), así que aplicar los tres de cualquier entorno por separado los deja
conviviendo en el mismo Namespace sin chocar entre sí.

```
app/                     Código fuente de la API (Go) + Dockerfile + docs
web/                     Kustomize: Deployment/Service/ConfigMap de la API
  base/
  overlays/{dev,test,prod}          cada uno incluye su propio namespace.yaml
db/                      Kustomize: una sola instancia MariaDB "genérica" en base/
  base/                    Deployment/Service/PVC genéricos (nombre "mariadb")
  overlays/test/           mariadb-test, con su namespace.yaml
  overlays/prod/           mariadb-prod, con su namespace.yaml
cache/                   Kustomize: Valkey (Redis-compatible)
  base/
  overlays/{dev,test,prod}          cada uno incluye su propio namespace.yaml
```

`db/overlays/test` y `db/overlays/prod` son casi idénticos entre sí: ambos
instancian el mismo `db/base/` genérico, y solo cambian el `nameSuffix`
(`-test`/`-prod`), el Namespace y los valores del Secret de credenciales.
`dev` no tiene carpeta en `db/`: SQLite vive en un PVC montado directamente
en el pod de la API (ver `web/overlays/dev/pvc.yaml`), no hace falta un
workload de base de datos aparte.

## Requisitos previos

- Un clúster de Kubernetes (local: [kind](https://kind.sigs.k8s.io/) o
  [minikube](https://minikube.sigs.k8s.io/); o uno remoto).
- `kubectl` (incluye Kustomize integrado: `kubectl kustomize` / `kubectl apply -k`).
- La imagen publicada en GHCR.
  Mientras tanto, puedes construirla local (`docker build -t creative-ideas-api:local app/`)
  y cargarla a un clúster kind (`kind load docker-image creative-ideas-api:local`),
  ajustando la imagen del overlay que pruebes.

## Desplegar un entorno

Cada componente se aplica con su propio comando. Por ejemplo, para levantar
**test** completo (API + caché + base de datos):

```bash
kubectl apply -k db/overlays/test
kubectl apply -k cache/overlays/test
kubectl apply -k web/overlays/test
```

El orden no importa: el primero que se aplique crea el Namespace
`ideas-creativas-test`, y los siguientes simplemente lo encuentran ya
existente. Para **dev** solo hacen falta dos componentes (no hay `db/overlays/dev`,
SQLite vive dentro del propio pod de la API):

```bash
kubectl apply -k cache/overlays/dev
kubectl apply -k web/overlays/dev
```

Y para **prod** (misma forma que test, solo cambian réplicas y credenciales):

```bash
kubectl apply -k db/overlays/prod
kubectl apply -k cache/overlays/prod
kubectl apply -k web/overlays/prod
```

Para ver los manifiestos ya resueltos de cualquier componente sin aplicarlos:

```bash
kubectl kustomize web/overlays/test
```

### Verificar

```bash
kubectl -n ideas-creativas-dev get pods
kubectl -n ideas-creativas-dev port-forward svc/ideas-creativas-api 8080:80
curl localhost:8080/healthz
```

## Notas

Los manifiestos se mantienen deliberadamente mínimos (sin `resources`,
`securityContext`, `PodDisruptionBudget` ni `affinity`, y sin replicación de
MariaDB): el objetivo de este repo es mostrar arquitectura hexagonal y
Kustomize de la forma más legible posible, no ser una base lista para producción.
