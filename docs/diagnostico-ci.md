
# Laboratorio de integración continua

## 1. Enlaces

| Qué | Enlace |
|---|---|
| Repositorio en el tag `lab-ci` | https://github.com/rzambrana98/api-tareas/tree/lab-ci |
| Pull request que el check bloqueó (prueba rota a propósito) | https://github.com/rzambrana98/api-tareas/pull/9 |

## 2. Diagnóstico de los tres pipelines rotos

### Caso 1

**Síntoma** (el paso que falló primero y su mensaje, copiado del log como texto):

```text
/opt/hostedtoolcache/Python/3.12.16/x64/bin/python: No module named pytest
Process completed with exit code 1.
```

**Causa:** En `.github/workflows/ci.yml`, el paso de instalación de dependencias utilizaba `requirements.txt` en lugar de `requirements-dev.txt`. Esto provocó que `pytest` no estuviera instalado y que las pruebas automatizadas no pudieran ejecutarse.

**Corrección:** Se modificó el workflow para instalar las dependencias desde `requirements-dev.txt`. La corrección se validó exitosamente en el [PR #12](https://github.com/rzambrana98/api-tareas/pull/12). · Commit: https://github.com/rzambrana98/api-tareas/commit/a2ad2ea

**Prevención:** Verificar que el pipeline instale las dependencias necesarias para las pruebas y mantener los controles obligatorios de CI antes de integrar cambios a `main`.

### Caso 2

**Síntoma** (el paso que falló primero y su mensaje, copiado del log como texto):

```text
No files were found with the provided path: reporte/. No artifacts will be uploaded.
```

**Causa:** En `.github/workflows/ci.yml`, la publicación del artefacto utilizaba la ruta `reporte/`, mientras que las pruebas generaban el archivo `junit.xml` dentro del directorio `reportes/`.

**Corrección:** Se cambió la ruta del artefacto a `reportes/`. La corrección se validó en el [PR #13](https://github.com/rzambrana98/api-tareas/pull/13), donde el pipeline finalizó exitosamente y se confirmó la publicación del artefacto `reporte-pruebas` de 606 bytes. · Commit: https://github.com/rzambrana98/api-tareas/commit/0e622aa

**Prevención:** Comprobar que las rutas de los artefactos coincidan con los directorios utilizados por los scripts de pruebas. Mantener `if-no-files-found: error` para detectar automáticamente archivos faltantes.

### Caso 3

**Síntoma** (el paso que falló primero y su mensaje, copiado del log como texto):

```text
FileNotFoundError: [Errno 2] No such file or directory: '/home/runner/work/api-tareas/api-tareas/tests/datos/tareas_ejemplo.json'
ERROR tests/test_datos_ejemplo.py::test_carga_las_tareas_de_ejemplo - FileNotFoundError
22 passed, 1 error in 0.25s
Process completed with exit code 1.
```

**Causa:** La regla `*.json` del archivo `.gitignore` excluía `tests/datos/tareas_ejemplo.json`. Aunque el archivo existía localmente, Git no lo incorporaba al repositorio. Por esa razón, GitHub Actions no pudo encontrarlo durante la ejecución de las pruebas.

**Corrección:** Se agregó la excepción `!tests/datos/tareas_ejemplo.json` en `.gitignore` y se incorporó el archivo JSON al repositorio. La corrección se validó en el [PR #14](https://github.com/rzambrana98/api-tareas/pull/14), donde las 23 pruebas finalizaron exitosamente. · Commit: https://github.com/rzambrana98/api-tareas/commit/a6a8ae1

**Prevención:** Revisar las reglas de `.gitignore` y comprobar que los archivos necesarios para las pruebas estén incluidos en el repositorio. Validar su disponibilidad durante la ejecución de GitHub Actions.

## 3. Uso de IA

Se hace uso de IA como apoyo para interpretar los logs de GitHub Actions y redactar de manera mas presentable este informe
