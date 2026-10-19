Proyecto Kubernetes 202630 — Jobs parametrizados desde línea de comandos

Herramienta de consola en Python que lanza, monitorea y limpia **Jobs de Kubernetes** parametrizados. En vez de editar un YAML por cada ejecución, el usuario elige en un menú el **lenguaje**, la **tarea** y la **complejidad**, y el programa crea el `Job` con el cliente oficial de Kubernetes para Python (sin invocar `kubectl`).

- **Integrantes:** `<Juan Alvarez>`, `<Sebastian Salgado>`, `<Jesús Marquéz>`
- **Entorno:** Minikube (runtime containerd) + Python 3 + librería `kubernetes`

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
| `scripts/preparar-imagenes.ps1` | Script del enunciado para construir las imágenes (asume el runtime Docker; ver sección 3). |
| `entrega/gestor_jobs.py` | **Programa del grupo**: el gestor de Jobs. |
| `entrega/requirements.txt` | Dependencias de Python. |

## 3. Preparación y ejecución

Desde la carpeta raíz del proyecto:

```powershell
minikube start
kubectl apply -f .\k8s\00-namespace.yaml
pip install -r .\entrega\requirements.txt
```

### Construcción de las imágenes

Las imágenes se construyen **dentro de Minikube** y los Jobs usan `imagePullPolicy: Never`, de modo que solo se ejecutan imágenes ya cargadas localmente. Nuestro Minikube usa el runtime **containerd**, y con él `minikube docker-env` (que utiliza `scripts/preparar-imagenes.ps1`) falla. Por eso construimos cada imagen con `minikube image build`:

```powershell
minikube image build -t kubernates-202630-python:1.0 .\imagenes\python
minikube image build -t kubernates-202630-java:1.0 .\imagenes\java
minikube image build -t kubernates-202630-c:1.0 .\imagenes\c
```

Para comprobar que quedaron cargadas:

```powershell
minikube image ls | findstr kubernates
```

> **Importante:** hay que reconstruir la imagen cada vez que cambie el programa de un contenedor. Si Kubernetes sigue ejecutando una versión anterior, se elimina la imagen y se vuelve a construir:
> `minikube image rm kubernates-202630-python:1.0`

### Ejecución

```powershell
python .\entrega\gestor_jobs.py
```

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
| 2 | Lista todos los Jobs del namespace con su estado: `pendiente`, `en ejecución`, `completado` o `fallido`, con el motivo cuando Kubernetes lo reporta (por ejemplo `fallido (OOMKilled)`). |
| 3 | Permite elegir un Job y muestra la salida de su contenedor. |
| 4 | Elimina los Jobs terminados (completados o fallidos) tras pedir confirmación; no toca los que siguen corriendo. |
| 5 | Sale del programa. |

## 5. Implementación

### 5.1 Clientes de Kubernetes

- `cliente_batch()` carga el kubeconfig y devuelve un `BatchV1Api`, con el que se crean, listan y eliminan Jobs.
- `cliente_core()` hace lo mismo con `CoreV1Api`, necesario para consultar los **Pods** (estado y logs).

### 5.2 Selección: lenguaje, tarea y complejidad

- `elegir_opcion(titulo, opciones)` es el selector genérico: muestra una lista numerada, valida la entrada y devuelve el texto elegido (o `None` si el usuario escribe `0` o deja la entrada vacía para volver).
- `seleccionar(catalogo)` lo usa tres veces: lenguaje (claves del catálogo), tarea (`definicion["tasks"]`) y complejidad. Devuelve `(lenguaje, tarea, complejidad, imagen, argumentos, recursos)`.
- `argumentos` es la lista que recibirá el contenedor, por ejemplo `["fib", "30"]`.

### 5.3 La complejidad fija dos cosas a la vez

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
| `primos` | 100000 | 500000 | 1000000 |

`requests` es lo que Kubernetes **reserva** para programar el Pod; `limits` es el **tope**. Si el contenedor supera el límite de memoria, Kubernetes lo mata con `OOMKilled`; si supera el de CPU, lo frena (*throttling*).

### 5.4 Creación del Job: del YAML al cliente Python

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

### 5.5 Estado de las tareas

`describir_estado(status)` traduce el `V1JobStatus` a una frase corta. Los contadores valen `None` mientras Kubernetes no los fija, de ahí el `or 0`. El orden de las comprobaciones es:

1. `active > 0` → **en ejecución**
2. `succeeded > 0` → **completado**
3. `failed > 0` → **fallido**
4. En otro caso → **pendiente**

Esta primera lectura tiene una limitación: `status.active` también cuenta los Pods que todavía no arrancaron (por ejemplo, por una imagen que no está cargada). Para no mostrar "en ejecución" cuando en realidad el Pod está bloqueado, `listar_jobs()` refina el resultado consultando el Pod:

- `estado_pod(nombre)` devuelve la fase del Pod y, si el contenedor está esperando, el motivo (`state.waiting.reason`, por ejemplo `ErrImageNeverPull`). Si la fase es `Pending`, el estado se muestra como `pendiente (<motivo>)`.
- `motivo_fallo(nombre)` lee `state.terminated.reason` del contenedor. Para un Job fallido muestra la causa, por ejemplo `fallido (OOMKilled)` o `fallido (Error)`.

`listar_jobs()` usa `list_namespaced_job` y devuelve parejas `(nombre, estado)`. `consultar_estado(nombre)` lee un Job concreto con `read_namespaced_job_status`.

### 5.6 Logs

Los logs pertenecen al **Pod**, no al Job. Kubernetes etiqueta cada Pod creado por un Job con `job-name=<nombre del Job>`, así que `consultar_logs(nombre)`:

1. busca el Pod con `list_namespaced_pod(label_selector="job-name=<nombre>")`,
2. si no hay Pods, avisa que el Job todavía no tiene ninguno,
3. si el Pod está en `Pending`, avisa que aún no arranca (no hay logs que leer),
4. en otro caso pide su salida con `read_namespaced_pod_log`.

### 5.7 Limpieza

- `jobs_terminados()` devuelve los Jobs con `active == 0` y al menos un Pod correcto o fallido. Un Job en ejecución **nunca** aparece, así que la limpieza no cancela trabajo en curso.
- `eliminar_job(nombre)` usa `delete_namespaced_job` con `propagation_policy="Background"`, de modo que Kubernetes borra también, en cascada, el Pod que creó el Job.
- `accion_limpiar()` muestra qué se va a eliminar y pide confirmación (`s/N`) antes de borrar.

### 5.8 Manejo de errores

El bucle del `menu()` captura `ApiException` (errores del API Server: nombre repetido, namespace inexistente, permisos insuficientes) y cualquier otra excepción, para que el menú no se caiga.

## 6. Tarea adicional: `primos`

- **Qué hace:** cuenta cuántos números primos hay hasta `N`, probando la divisibilidad de cada número hasta su raíz cuadrada (división de prueba). Es una tarea intensiva en CPU y casi sin uso de memoria. `N` lo fija la complejidad: 100000 (baja), 500000 (media) y 1000000 (alta).
- **Lenguaje elegido:** Python.
- **Resultado esperado** (sirve para verificarla): hasta 100000 hay 9592 primos (el último es 99991), hasta 500000 hay 41538 y hasta 1000000 hay 78498.
- **Archivos modificados:**
  - `imagenes/python/tareas.py`: función `primos(n)` y su registro en el diccionario `TAREAS`. Respeta el contrato: recibe `<tarea> <N>`, escribe logs en stdout y cierra con un JSON de una línea.
  - `catalogo/tareas.json`: se agregó `primos` a la lista de tareas de Python.
  - `entrega/gestor_jobs.py`: se agregó su fila en `TAMANOS`.
- Se reconstruyó la imagen de Python con `minikube image build`.

## 7. Pruebas y evidencia

Cada prueba se contrastó con lo que reporta el clúster:

```powershell
kubectl get jobs,pods -n estudiantes-202630
kubectl logs -n estudiantes-202630 job/<nombre-del-job>
```

| Prueba | Resultado esperado | Evidencia |
| --- | --- | --- |
| `hola` en Python, Java y C | Completado; el arranque de la JVM se nota frente a Python y C | `<<img width="494" height="525" alt="image" src="https://github.com/user-attachments/assets/4618a987-0f78-4954-94ce-9e9668fe8883" />

| `fib` / `matriz` / `ordenar` en niveles baja, media, alta | Mayor tiempo y mayores recursos a mayor nivel | `<<img width="373" height="511" alt="image" src="https://github.com/user-attachments/assets/3618f2e4-8656-41ac-8db5-93712c2c4ee9" /> 

| `primos` en Python (baja, media, alta) | Completado, con 9592 / 41538 / 78498 primos en el JSON final | `<<img width="493" height="528" alt="image" src="https://github.com/user-attachments/assets/478c7696-38b5-4a85-9771-d0010317e937" />
>`    

| Limpieza (opción 4) | Elimina terminados, respeta los que corren | `<<img width="435" height="287" alt="image" src="https://github.com/user-attachments/assets/7ff9a529-86e0-420a-bc97-136f4b3bda3d" />
>` 

## 8. Problemas encontrados y aprendizajes

- **`docker-env` no funciona con containerd.** 
- **Un Pod que no arranca cuenta como "activo".** Un Job cuya imagen no estaba cargada aparecía como "en ejecución" aunque su Pod seguía en `Pending`. Para distinguirlo, el gestor consulta el Pod y muestra `pendiente (<motivo>)`.
- **Los logs son del Pod, no del Job.** Por eso hay que buscar el Pod con la etiqueta `job-name` antes de pedir los logs.

