# Registro de resultados – Integración de infraestructura, Kubernetes y despliegue

Repositorio: https://github.com/Jechev1/Lab5-DevOps (creado a partir de `k8s-azure-template`)
Fecha: 2026-11-06

## Comprobaciones previas del equipo local

| Herramienta | Resultado |
|---|---|
| `git --version` | git version 2.53.0.windows.2 |
| `az version` | azure-cli 2.91.0 |
| `terraform version` | Terraform v1.15.8 (windows_amd64) |
| `ssh -V` | OpenSSH_for_Windows_9.5p2, LibreSSL 3.8.2 |
| `scp` | Muestra la sintaxis de uso |

- Clave SSH: `~/.ssh/id_ed25519` y `~/.ssh/id_ed25519.pub` ya existían (`Test-Path` → `True` en ambos); se conservó el par existente.
- IPv4 pública de origen: obtenida con `api.ipify.org` y usada como `ssh_source_cidr = "x.x.x.x/32"`.
- Azure: sesión iniciada con `az login`; suscripción activa **Azure for Students** (`az account show`).

## 1. Repositorio e imagen

Cambios realizados:
- `.github/workflows/publish-image.yml`: `push: false` → `push: true`.
- Además se añadió un paso que normaliza el nombre de la imagen a minúsculas (`ghcr.io/${GITHUB_REPOSITORY,,}`), porque GHCR rechaza nombres con mayúsculas y el usuario es `Jechev1`.

Ejecución del workflow **Publicar imagen en GHCR**:
- Run: https://github.com/Jechev1/Lab5-DevOps/actions/runs/42249186540, resultado `success`.
- Pruebas: 3 ejecutadas, 3 superadas, 0 fallidas.
- Imagen publicada: `ghcr.io/jechev1/lab5-devops:sha-8f18d5a76e3e8e5c4aff7ea875fdb27df0b611a3`
- Digest: `sha256:392c150ca379b5f7aa4d0bb85189c1f8b7574ad123c09b70145913da904e97f4`
- Visibilidad del paquete: **Public**. Se verificó descargando el manifiesto sin credenciales (HTTP 200).

**Comprobación.** La etiqueta `sha-8f18d5a…` es el SHA completo del commit que ejecutó el workflow (`github.sha`), así que la imagen se puede rastrear hasta la revisión exacta del código que la produjo. Como una etiqueta puede reasignarse, la identidad estricta del contenido la da el *digest*.

## 2. Configuración de la infraestructura

- `terraform.tfvars` se creó desde `terraform.tfvars.example` con el `subscription_id` y el `ssh_source_cidr` reales. Está excluido por `.gitignore` y no se versiona.
- Valores usados: `location = "mexicocentral"`, `vm_size = "Standard_B2s_v2"`, `vm_zone = "1"`, `resource_prefix = "devops-k3s"`.
- **Actividad:** se añadió a `outputs.tf` el output `resource_group_name = azurerm_resource_group.lab.name`.

## 3. Aprovisionamiento y K3s

| Comando | Resultado |
|---|---|
| `terraform init` / `terraform validate` | Correctos |
| `terraform plan -out=lab.tfplan` | `Plan: 8 to add, 0 to change, 0 to destroy.` |
| `terraform apply lab.tfplan` | `Apply complete! Resources: 8 added, 0 changed, 0 destroyed.` |

Los 8 recursos del plan son: grupo de recursos, red virtual, subred, IP pública, NSG, NIC, asociación NIC–NSG y VM Linux.

Outputs: `public_ip = 158.23.84.126`, `ssh_command = ssh azureuser@158.23.84.126`, `resource_group_name = devops-k3s-rg`.

En la VM (`ssh azureuser@158.23.84.126`):
- `cloud-init status --wait` → `status: done`.
- `k3s kubectl get nodes -o wide` → `k3s-lab   Ready   control-plane   v1.35.2+k3s1   Ubuntu 24.04.5 LTS`.
- `k3s kubectl get pods -A`:
  - `coredns`, `local-path-provisioner`, `traefik` y `metrics-server` en `1/1 Running`.
  - `svclb-traefik` en `2/2 Running`.
  - `helm-install-traefik*` en `Completed`.

**Comprobación.** `get nodes` confirma la disponibilidad del nodo (`Ready`). `get pods -A` permite observar los componentes que operan dentro del clúster.

## 4–6. Manifiestos, Secret y despliegue

- En `k8s/06-api-deployment.yaml` se fijó `image: ghcr.io/jechev1/lab5-devops:sha-8f18d5a76e3e8e5c4aff7ea875fdb27df0b611a3`. Los manifiestos se copiaron con `scp -r k8s azureuser@IP:~/`.
- Se aplicó `01-namespace.yaml` y se creó el Secret `postgres-credentials` (clave `password`) en `devops-demo`. La contraseña no se mostró en la terminal y no se versionó.
  - `get secret` → `postgres-credentials   Opaque   1`.
- `kubectl apply -f ~/k8s/`:

```
pod/demo-api-5db4cf9bc-qkgtx   1/1   Running
pod/postgres-55c96cbbc-5rddx   1/1   Running
deployment.apps/demo-api   1/1     deployment.apps/postgres   1/1
service/demo-api   ClusterIP   80/TCP      service/postgres   ClusterIP   5432/TCP
postgres-data   Bound   1Gi   RWO   local-path
ingress demo-api   traefik   *   10.30.1.4   80
```

En el primer arranque `demo-api` se reinició 3 veces mientras PostgreSQL todavía no aceptaba conexiones. Esto ocurre porque `server.js` ejecuta `initializeDatabase` antes de `listen`. Después quedó estable.

**Comprobación (diagnóstico).**
- **Configuración ausente:** el Pod queda en `CreateContainerConfigError` y el evento dice `secret "postgres-credentials" not found`. El contenedor no llega a arrancar.
- **Fallo de la aplicación:** el estado es `Error`/`CrashLoopBackOff` con `RESTARTS` crecientes, y la causa aparece en `kubectl logs --previous` (p. ej. `ECONNREFUSED postgres:5432`).
- **Descarga de imagen:** el estado es `ErrImagePull`/`ImagePullBackOff` con eventos `not found`/`unauthorized` (etiqueta errónea o paquete privado).

## 7. Pruebas desde el equipo local

| Solicitud | Respuesta |
|---|---|
| `GET /health` | `HTTP/1.1 200 OK` `{"status":"ok","component":"api"}` |
| `GET /notes` | `HTTP/1.1 200 OK` `{"notes":[]}` |
| `POST /notes` `{"text":"Nota de prueba en Kubernetes"}` | `HTTP/1.1 201 Created` `{"id":"1","text":"Nota de prueba en Kubernetes","created_at":"2026-11-06T16:07:32.693Z"}` |
| `GET /notes` | `HTTP/1.1 200 OK` `{"notes":[{"id":"1","text":"Nota de prueba en Kubernetes",...}]}` |

**Comprobación.** `/health` responde sin consultar la base de datos. Solo demuestra que la petición atravesó NSG → Traefik → Service → Express. El funcionamiento de PostgreSQL lo demuestran `POST /notes` (201) y el `GET` posterior que devuelve la nota.

## 8. Persistencia ante el reemplazo del Pod

1. `kubectl delete pod -n devops-demo -l app=postgres` eliminó `postgres-55c96cbbc-5rddx`.
2. El Deployment creó `postgres-55c96cbbc-6n9gd` (`1/1 Running`).
3. El PVC `postgres-data` siguió `Bound` al mismo volumen `pvc-347f5ff1-…`.

Durante unos segundos `GET /notes` devolvió `503 no available server`. Express perdió la conexión con PostgreSQL al eliminarse el Pod y su proceso terminó; mientras se reiniciaba no había Pod `Ready` detrás del Service. Cuando `demo-api` volvió a `Ready`:

```
GET /notes → HTTP/1.1 200 OK
{"notes":[{"id":"1","text":"Nota de prueba en Kubernetes","created_at":"2026-11-06T16:07:32.693Z"}]}
```

La nota creada antes del reemplazo se conservó.

**Análisis.** Persiste el PVC `postgres-data`, junto con el PV `local-path`, que es un directorio en el disco de la VM; el Pod nuevo lo vuelve a montar. Esto no garantiza la recuperación si se pierde el disco de la VM: el volumen vive en ese único disco del único nodo, sin réplica, respaldo ni copia externa.

## 9. Destrucción de la infraestructura

| Comando | Resultado |
|---|---|
| `terraform output resource_group_name` (antes de destruir) | `"devops-k3s-rg"` |
| `terraform plan -destroy` | `Plan: 0 to add, 0 to change, 8 to destroy.` (solo los 8 recursos del laboratorio) |
| `terraform destroy` | `Destroy complete! Resources: 8 destroyed.` |
| `az group exists --name devops-k3s-rg` | `false` |
| `terraform state list` | vacío |

La infraestructura se eliminó por completo y no queda consumo en la suscripción.

## Preguntas finales

1. **¿Qué operaciones ejecuta Terraform que no corresponden a un Deployment de Kubernetes?**
   Terraform crea, modifica y destruye la infraestructura de Azure mediante AzureRM:
   - grupo de recursos, VNet, subred, IP pública, NSG (regla SSH limitada a la IP /32 y HTTP), NIC y asociación NIC–NSG;
   - la VM Ubuntu, a la que además entrega la clave pública SSH y el `cloud-init` (`custom_data`) que instala K3s.

   También mantiene el archivo de estado, calcula planes y ejecuta `destroy`. Un Deployment solo declara réplicas y la plantilla de Pods dentro de un clúster que ya existe.

2. **¿Qué dependencia faltaría si ambos contenedores informaran `secret "postgres-credentials" not found`?**
   Faltaría el Secret `postgres-credentials`, con la clave `password`, en el namespace `devops-demo`. De él salen `POSTGRES_PASSWORD` para PostgreSQL y `DB_PASSWORD` para Express. Es una dependencia de configuración (`CreateContainerConfigError`), no un problema de imagen ni de código.

3. **¿Por qué no es suficiente comprobar exclusivamente `/health`?**
   Porque `/health` no consulta PostgreSQL, y la readiness probe usa ese mismo endpoint. Por eso el Pod puede estar "listo" y la ruta pública responder 200 aunque la base de datos no funcione. Hay que ejercitar `POST /notes` y `GET /notes` para demostrar escritura y lectura en PostgreSQL.

4. **¿Qué propiedad se comprobó al recuperar una nota después de recrear PostgreSQL y qué fallo seguiría sin estar cubierto?**
   Se comprobó la persistencia de los datos ante el reemplazo de un Pod: el PVC sobrevive al Pod y se vuelve a montar. No se cubre la pérdida o destrucción del nodo o de su disco, porque no hay replicación, alta disponibilidad, copias de seguridad ni recuperación ante desastres.

5. **¿Qué responsabilidades adicionales supone ejecutar K3s en una VM frente a utilizar un servicio gestionado de Kubernetes?**
   - Instalar y actualizar K3s y Kubernetes, y parchear el sistema operativo.
   - Mantener la disponibilidad y los respaldos del plano de control y de su datastore.
   - Asegurar el acceso a la API y al servidor (SSH, NSG, certificados).
   - Operar el Ingress (Traefik) y el almacenamiento (`local-path`).
   - Monitorear, escalar agregando nodos, y diseñar y probar la recuperación de la plataforma.

   En AKS, EKS o GKE el proveedor opera total o parcialmente el plano de control y ofrece integraciones de red, identidad, almacenamiento y actualización. El equipo sigue respondiendo por los permisos, las versiones, los workloads y los datos.
