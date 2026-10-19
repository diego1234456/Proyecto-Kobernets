# Proyecto Kubernetes 202630 — Gestor de Jobs desde línea de comandos

Herramienta de consola en Python que lanza, monitorea y limpia **Jobs de Kubernetes** parametrizados. En vez de editar un YAML por cada ejecución, el usuario elige en un menú el **lenguaje**, la **tarea** y la **complejidad**, y el programa crea el `Job` con el cliente oficial de Kubernetes para Python (sin invocar `kubectl`).

- **Integrantes:** `<Juan Diego Alvarez>`, `<Jesús Marquéz>`, `<Sebastián Salgado>`
- **Video de demostración:** `<LINK DE YOUTUBE>`
- **Entorno:** Minikube + Python 3 + librería `kubernetes`

---

## 1. Recorrido completo

```
gestor_jobs.py -> Job de Kubernetes -> Pod -> contenedor con argumentos -> logs
```

1. El usuario elige lenguaje, tarea y complejidad en el menú.
2. El gestor construye un objeto `V1Job` y lo envía al API Server (`create_namespaced_job`).
3. El controlador de Jobs de Kubernetes crea un **Pod** a partir de la plantilla del Job.
4. El Pod ejecuta el contenedor de la imagen elegida con los argumentos `["<tarea>", "<N>"]`.
5. El contenedor escribe sus logs en stdout y termina con un JSON de una línea.
6. El gestor lee el estado del Job y los logs del Pod a través del API Server.

## 2. Estructura del repositorio

| Carpeta / archivo | Contenido |
| --- | --- |
| `imagenes/` | Imágenes de Python, Java y C. Reciben `<tarea> <N>`, escriben logs en stdout y cierran con un JSON de una línea. |
| `catalogo/tareas.json` | Catálogo de lenguajes, imágenes y tareas admitidas. |
| `k8s/00-namespace.yaml` | Namespace dedicado `estudiantes-202630`. |
| `scripts/preparar-imagenes.ps1` | Construye las imágenes contra el demonio Docker de Minikube. |
| `entrega/gestor_jobs.py` | **Programa del grupo**: el gestor de Jobs. |
| `entrega/requirements.txt` | Dependencias de Python. |

## 3. Preparación y ejecución

Desde la carpeta raíz del proyecto:

```powershell
minikube start
kubectl apply -f .\k8s\00-namespace.yaml
.\scripts\preparar-imagenes.ps1
pip install -r .\entrega\requirements.txt
```

Ejecución (sin argumentos):

```powershell
python .\entrega\gestor_jobs.py
```

> **Importante:** `preparar-imagenes.ps1` debe volver a ejecutarse cada vez que cambie el programa de un contenedor. No se usa `minikube image load` para actualizar una imagen, porque no reemplaza un tag ya existente y el clúster seguiría ejecutando la versión anterior.

## 4. El menú

```
=== Gestor de Jobs - namespace estudiantes-202630 ===
  1. Ejecutar una tarea nueva
  2. Revisar el estado de las tareas
  3. Ver los logs de una tarea
  4. Limpiar las tareas terminadas
  5. Salir
```

| Opción | Qué hace |
| --- | --- |
| 1 | Pide lenguaje, tarea y complejidad (el menú de complejidad muestra el `N` y la memoria de cada nivel), crea el Job y muestra su nombre, imagen, argumentos y recursos. |
| 2 | Lista todos los Jobs del namespace con su estado: `pendiente`, `en ejecución`, `completado` o `fallido`. |
| 3 | Permite elegir un Job y muestra la salida de su contenedor. |
| 4 | Elimina los Jobs terminados (completados o fallidos) tras pedir confirmación; no toca los que siguen corriendo. |
| 5 | Sale del programa. |

## 5. Implementación

### 5.1 Selección: lenguaje, tarea y complejidad

- `elegir_opcion(titulo, opciones)` es el selector genérico: muestra una lista numerada, valida la entrada y devuelve el texto elegido (o `None` si el usuario escribe `0` o deja la entrada vacía para volver).
- `seleccionar(catalogo)` lo usa tres veces: lenguaje (claves del catálogo), tarea (`definicion["tasks"]`) y complejidad. Devuelve `(lenguaje, tarea, complejidad, imagen, argumentos, recursos)`.
- `argumentos` es la lista que recibirá el contenedor, por ejemplo `["fib", "30"]`.

### 5.2 La complejidad fija dos cosas a la vez

La complejidad elegida determina **los recursos** del contenedor (`NIVELES`) y **el tamaño del trabajo**, el parámetro `N` (`TAMANOS`). Por eso un nivel alto no solo tarda más: también reserva más CPU y permite más memoria.

| Nivel | CPU request | CPU limit | Mem request | Mem limit |
| --- | --- | --- | --- | --- |
| baja | 100m | 500m | 64Mi | 128Mi |
| media | 250m | 1000m | 128Mi | 512Mi |
| alta | 500m | 2000m | 256Mi | 1Gi |

| Tarea | baja | media | alta |
| --- | --- | --- | --- |
| `hola` | 0 | 0 | 0 |
| `ordenar` | 100000 | 1000000 | 3000000 |
| `fib` | 25 | 30 | 35 |
| `matriz` | 60 | 120 | 250 |
| `<TAREA_ADICIONAL>` | `<N>` | `<N>` | `<N>` |

`requests` es lo que Kubernetes **reserva** para programar el Pod; `limits` es el **tope**. Si el contenedor supera el límite de memoria, Kubernetes lo mata con `OOMKilled`; si supera el de CPU, lo frena (*throttling*).

### 5.3 Creación del Job: del YAML al cliente Python

`crear_job(nombre, imagen, argumentos, recursos)` arma el equivalente exacto del siguiente YAML con las clases del cliente de Kubernetes:

| YAML (camelCase) | Cliente Python (snake_case) | Para qué sirve |
| --- | --- | --- |
| `metadata.name` / `namespace` | `V1ObjectMeta(name=..., namespace=...)` | Nombre único (con *timestamp*) dentro del namespace `estudiantes-202630`. |
| `spec.backoffLimit: 0` | `V1JobSpec(backoff_limit=0)` | Si el Pod falla, no se reintenta: el fallo queda visible. |
| `restartPolicy: Never` | `V1PodSpec(restart_policy="Never")` | El contenedor no se reinicia; el Pod conserva su estado final para inspeccionarlo. |
| `imagePullPolicy: Never` | `V1Container(image_pull_policy="Never")` | Usa solo las imágenes cargadas en Minikube; no intenta descargarlas. |
| `args: [...]` | `V1Container(args=argumentos)` | Pasa `<tarea> <N>` al contenedor. |
| `resources.requests` / `limits` | `V1ResourceRequirements(requests=..., limits=...)` | CPU y memoria según el nivel elegido. |

El nombre del Job se construye en `nombre_job(...)` como `lenguaje-tarea-complejidad-<timestamp>`: solo minúsculas, dígitos y guiones, máximo 63 caracteres. El sufijo de tiempo permite repetir la misma tarea sin chocar con un Job anterior.

### 5.4 Estado de las tareas

`describir_estado(status)` traduce el `V1JobStatus` a una frase corta. Los contadores valen `None` mientras Kubernetes no los fija, de ahí el `or 0`. El orden de las comprobaciones es:

1. `active > 0` → **en ejecución**
2. `succeeded > 0` → **completado**
3. `failed > 0` → **fallido**
4. En otro caso → **pendiente**

`listar_jobs()` usa `list_namespaced_job` y devuelve parejas `(nombre, estado)`. `consultar_estado(nombre)` lee un Job concreto con `read_namespaced_job_status`.

### 5.5 Logs

Los logs pertenecen al **Pod**, no al Job. Kubernetes etiqueta cada Pod creado por un Job con `job-name=<nombre del Job>`, así que `consultar_logs(nombre)`:

1. busca el Pod con `list_namespaced_pod(label_selector="job-name=<nombre>")`,
2. toma su nombre y
3. pide su salida con `read_namespaced_pod_log`.

### 5.6 Limpieza

- `jobs_terminados()` devuelve los Jobs con `active == 0` y al menos un Pod correcto o fallido. Un Job en ejecución **nunca** aparece, así que la limpieza no cancela trabajo en curso.
- `eliminar_job(nombre)` usa `delete_namespaced_job` con `propagation_policy="Background"`, de modo que Kubernetes borra también, en cascada, el Pod que creó el Job.
- `accion_limpiar()` muestra qué se va a eliminar y pide confirmación (`s/N`) antes de borrar.

### 5.7 Manejo de errores

El bucle del `menu()` captura `ApiException` (errores del API Server: nombre repetido, namespace inexistente, permisos insuficientes) y cualquier otra excepción, para que el menú no se caiga.

## 6. Tarea adicional: `<NOMBRE_TAREA>`

- **Qué hace:** `<descripción de la tarea y del parámetro N>`
- **Lenguaje elegido:** `<Python / Java / C>`
- **Archivos modificados:**
  - `imagenes/<lenguaje>/...`: se implementó la tarea respetando el contrato (`<tarea> <N>`, logs en stdout, JSON de una línea).
  - `catalogo/tareas.json`: se agregó `<NOMBRE_TAREA>` a la lista de tareas del lenguaje.
  - `entrega/gestor_jobs.py`: se agregó su fila en `TAMANOS`.
- Se reconstruyó la imagen con `preparar-imagenes.ps1`.

## 7. Pruebas y evidencia

Cada prueba se contrastó con lo que reporta el clúster:

```powershell
kubectl get jobs,pods -n estudiantes-202630
kubectl logs -n estudiantes-202630 job/<nombre-del-job>
```

| Prueba | Resultado esperado | Evidencia |
| --- | --- | --- |
| `hola` en Python, Java y C | Completado; el arranque de la JVM se nota frente a Python y C | `<captura>` |
| `fib` / `matriz` / `ordenar` en niveles baja, media, alta | Mayor tiempo y mayores recursos a mayor nivel | `<captura>` |
| Job fallido por memoria (`OOMKilled`) | Estado `fallido` y motivo `OOMKilled` en el Pod | `<captura>` |
| Tarea adicional | Completado con su JSON final | `<captura>` |
| Limpieza (opción 4) | Elimina terminados, respeta los que corren | `<captura>` |

## 8. Problemas encontrados y aprendizajes

- `<Ejemplo: ErrImageNeverPull por no recargar la imagen tras un cambio>`
- `<Ejemplo: diferencia entre requests y limits observada con OOMKilled>`
- `<Ejemplo: los logs son del Pod y no del Job>`

## 9. Demo

Video de demostración: `<LINK DE YOUTUBE>`
