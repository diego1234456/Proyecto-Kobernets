# Proyecto-Kobernets


Este proyecto implementa una herramienta en consola desarrollada en Python que se comunica directamente con el API Server de Kubernetes para orquestar la ejecución, monitoreo y limpieza de tareas computacionales (Jobs).


# Arquitectura y Diseño de la Solución

La aplicación está estructurada en el script `entrega/gestor_jobs.py` utilizando la librería oficial `kubernetes` (`kubernetes-client` de Python). La arquitectura sigue un patrón modular en el que cada función cumple una responsabilidad única:

# 1. Autenticación y Conexión con el Clúster
- **`cliente_batch()`**: Inicializa el cliente `batch_v1_api.BatchV1Api` cargando la configuración local (`config.load_kube_config()`). Permite manipular el recurso `Job` de Kubernetes en el namespace configurado (`estudiantes-202630`).
- **`cliente_core()`**: Inicializa el cliente `core_v1_api.CoreV1Api`, necesario para interactuar directamente con los `Pods` (obtención de logs y detalles de ejecución).

##  Explicación Detallada de Funciones e Implementación

# 1. `seleccionar(coleccion)`
- **Propósito**: Mantiene una interfaz limpia e interactiva en la terminal.
- **Implementación**: Recibe una lista o diccionario y numera dinámicamente sus elementos. Permite al usuario elegir mediante índice numérico con validación de entradas.

# 2. `crear_job(...)`
- **Propósito**: Define y envía el manifiesto del `Job` a Kubernetes.
- **Implementación de Recursos y Configuración**:
  - Construye la estructura `V1Job` con especificación de metadatos (nombre único mediante timestamp) y plantilla de Pod (`V1PodTemplateSpec`).
  - **Límites de Recursos**: Asigna dinámicamente `requests` y `limits` de CPU y Memoria según la complejidad seleccionada (`baja`, `media`, `alta`).
  - **Política de Reinicio**: Se establece `restart_policy="Never"` para que, ante un fallo del contenedor, Kubernetes mantenga el estado final para inspección.
  - **Pull Policy**: Se define `image_pull_policy="Never"` para forzar el uso de las imágenes precargadas localmente en el entorno Minikube.

# 3. `describir_estado(status)`
- **Propósito**: Traducir el objeto `V1JobStatus` devuelto por Kubernetes a un estado comprensible en español.
- **Lógica**:
  - `status.succeeded > 0`: La tarea finalizó correctamente (**completado**).
  - `status.failed > 0` y sin activos: La tarea falló (**fallido**).
  - `status.active > 0`: La tarea se encuentra actualmente corriendo (**en ejecución**).
  - De lo contrario: Se clasifica como **pendiente**.

# 4. `listar_jobs()`
- **Propósito**: Consultar los trabajos existentes en el namespace.
- **Implementación**: Invoca `list_namespaced_job` para recuperar todos los Jobs dentro del namespace `estudiantes-202630` y genera una lista formateada de parejas `(nombre, estado_legible)`.

# 5. `consultar_logs(job_nombre)`
- **Propósito**: Obtener la salida estándar generada por la tarea.
- **Implementación**:
  - Primero realiza una búsqueda de los Pods asociados usando el selector de etiquetas `job-name=<job_nombre>` vía `list_namespaced_pod`.
  - Extrae el nombre del Pod y solicita sus logs con `read_namespaced_pod_log`.
  - Captura y maneja excepciones (`ApiException`) en caso de que el Pod aún no se haya inicializado.

# 6. `jobs_terminados()` y `eliminar_job(job_nombre)`
- **Propósito**: Gestión del ciclo de vida y limpieza de recursos.
- **Implementación**:
  - `jobs_terminados()` filtra y retorna únicamente los Jobs cuyo recuento `active == 0` y que tengan al menos 1 ejecución exitosa o fallida.
  - `eliminar_job()` utiliza `delete_namespaced_job` especificando la política de propagación `propagation_policy="Background"`. Esto asegura que al eliminar el recurso Job, Kubernetes elimine en cascada todos los Pods huérfanos creados por dicho trabajo.

---

##  Decisiones Técnicas y Sincronización con Minikube

1. **Gestión de Imágenes con containerd**:
   Para evitar errores de tipo `ErrImageNeverPull`, las imágenes de los contenedores (`C`, `Java`, `Python`) se compilan y cargan explícitamente dentro del registro interno de Minikube mediante el comando:
   ```powershell
   minikube image load kubernetes-202630-<lenguaje>:1.0
