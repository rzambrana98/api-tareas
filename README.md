# API de tareas

Aplicación de laboratorio del curso SOFT-756 (Implementación de CI/CD y Prácticas DevOps). Es una
API pequeña en Python (Flask) que guarda tareas en memoria. El código de la aplicación casi no
cambia durante el curso: lo que crece es el **pipeline** que la construye, la prueba, la empaqueta
y la despliega.

## Endpoints

| Método | Ruta | Respuesta |
|---|---|---|
| GET | `/health` | `{"estado": "ok"}` si la API está viva |
| GET | `/version` | Versión desplegada y ambiente (`APP_VERSION`, `APP_ENV`) |
| GET | `/tareas` | Lista de tareas |
| GET | `/tareas/<id>` | Una tarea, o 404 |
| POST | `/tareas` | Crea una tarea con `{"titulo": "..."}`; 201, o 400 si el título no es válido |

Si la variable `API_TOKEN` tiene valor, `POST /tareas` exige el encabezado
`Authorization: Bearer <token>`. Las lecturas no lo exigen.

## Configuración

| Variable | Para qué sirve | Valor por defecto |
|---|---|---|
| `APP_ENV` | Nombre del ambiente (desarrollo, staging, produccion) | `desarrollo` |
| `APP_VERSION` | Versión que reporta `/version` | `dev` |
| `API_TOKEN` | Token que protege `POST /tareas` | sin valor (no se exige) |

## Correr en local

Requisitos: Python 3.10 o posterior, Git y una terminal Bash (en Windows, **Git Bash**).

```bash
python -m venv .venv
source .venv/bin/activate          # en Windows (Git Bash): source .venv/Scripts/activate
python -m pip install -r requirements-dev.txt
bash scripts/pruebas.sh            # corre las pruebas
python -m flask --app app run --port 8000
```

Con la API en marcha, en otra terminal:

```bash
bash scripts/smoke.sh http://localhost:8000
```

Resultado esperado:

```text
Smoke test contra http://localhost:8000
OK /health
OK /version -> {"ambiente":"desarrollo","version":"dev"}
OK /tareas
Smoke test superado
```

## Estructura

```text
app/            código de la API (config.py lee las variables de entorno)
tests/          pruebas con pytest
scripts/        pruebas.sh y smoke.sh: los mismos comandos en local y en el pipeline
docs/           documentación del flujo de entrega
requirements.txt       dependencias de la aplicación, con versión fija
requirements-dev.txt   dependencias para desarrollar y probar
```

Prueba

## Despliegue y rollback

### Comparación de imágenes Docker

Se construyeron dos imágenes de la aplicación utilizando las mismas dependencias y configuración, modificando únicamente la imagen base de Python.

| Imagen base | Tamaño |
|---|---:|
| `python:3.12` | 1.63 GB |
| `python:3.12-slim` | 200.47 MB |

Se eligió `python:3.12-slim` porque permite reducir el tamaño de la imagen aproximadamente un 87.67%. Esto disminuye el espacio de almacenamiento requerido, facilita la transferencia de imágenes y reduce la cantidad de componentes innecesarios dentro del contenedor.

### Justificación del archivo .dockerignore

El archivo `.dockerignore` permite excluir archivos y directorios que no son necesarios para construir la imagen de la aplicación.

| Elemento | Justificación |
|---|---|
| `.git` | Excluye el historial y los metadatos del repositorio. |
| `.github` | Excluye las configuraciones de GitHub Actions. |
| `.venv` | Excluye el entorno virtual local de Python. |
| `venv` | Excluye otros entornos virtuales locales. |
| `**/__pycache__` | Excluye los directorios de caché generados por Python. |
| `**/*.pyc` | Excluye los archivos de bytecode de Python. |
| `.pytest_cache` | Excluye la caché generada por pytest. |
| `.ruff_cache` | Excluye la caché de la herramienta de análisis Ruff. |
| `.coverage` | Excluye los datos de cobertura de las pruebas. |
| `htmlcov` | Excluye los informes HTML de cobertura. |
| `reportes` | Excluye los reportes generados durante las pruebas. |
| `tests` | Excluye las pruebas, ya que no son necesarias durante la ejecución de la aplicación. |
| `docs` | Excluye la documentación del proyecto. |
| `.env` | Evita incluir configuraciones locales y posibles credenciales sensibles en el contexto de construcción. |

### Identificación del commit de una imagen

Las imágenes generadas por el pipeline CI/CD se publican en GitHub Container Registry (GHCR) utilizando el SHA completo del commit como etiqueta.

También se incorpora el SHA mediante la variable `APP_VERSION` y la etiqueta OCI `org.opencontainers.image.revision`, lo que permite identificar el código fuente utilizado para construir la imagen.

El endpoint `/version` permite consultar la versión desplegada y el ambiente en el que se está ejecutando la aplicación.

### Proceso de despliegue

El pipeline ejecuta las pruebas de integración continua, construye la imagen Docker y la publica en GHCR. Posteriormente, despliega el contenedor en staging y ejecuta un Smoke test para verificar los endpoints `/health`, `/version` y `/tareas`.

Si las comprobaciones son exitosas, el despliegue en producción queda pendiente de aprobación manual. Si staging falla, producción no se ejecuta.

### Proceso de rollback

El rollback se utiliza cuando una nueva versión presenta errores que afectan el funcionamiento de la aplicación y es necesario recuperar rápidamente una versión funcional anterior.

Para ejecutar el rollback:

1. Ingresar a la sección Actions del repositorio en GitHub.
2. Seleccionar el workflow Rollback.
3. Presionar Run workflow.
4. Seleccionar la rama `main`.
5. Introducir el SHA completo de un commit funcional cuya imagen esté disponible en GHCR.
6. Ejecutar el workflow y verificar que el Smoke test finalice correctamente.

Durante el laboratorio se recuperó exitosamente la imagen correspondiente al commit `dd770e247ca5a91745728475e90814858d5a91e1`.

Este procedimiento permite recuperar una versión anterior en el entorno de staging del laboratorio sin construir una nueva imagen ni modificar el historial del repositorio.

Posteriormente, se utilizó `git revert` para corregir el cambio defectuoso en el Dockerfile, restaurando el puerto 8000 y conservando el historial de Git. La versión corregida fue desplegada exitosamente en producción.
