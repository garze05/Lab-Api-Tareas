# Laboratorio de integración continua

Autor: Gabriel Rodríguez Ovares · SOFT-756 · Temas 2 y 3

## 1. Enlaces

| Qué | Enlace |
|---|---|
| Repositorio en el tag `lab-ci` | https://github.com/garze05/Lab-Api-Tareas/tree/lab-ci |
| Pull request que el check bloqueó (prueba rota a propósito) | https://github.com/garze05/Lab-Api-Tareas/pull/4 |

**Comprobación del bloqueo:** en el PR #4 se cambió `TITULO_MAXIMO = 120` por `100`,
como indica la Parte 2 de la guía. Falló `test_titulo_en_el_limite_se_acepta` con
`AssertionError: assert 400 == 201`: 1 prueba fallida y 21 aprobadas.
Mientras el PR estaba abierto, GitHub informó `mergeStateStatus: BLOCKED` y el check
`CI` en `FAILURE`. Se conservó la [respuesta de GitHub](evidencias/pr-bloqueado.json)
antes de cerrarlo sin merge. El [ruleset proteger-main](https://github.com/garze05/Lab-Api-Tareas/rules/27171230)
está activo para la rama por defecto, exige PR y el check `CI`, y no tiene excepciones
para saltarse las reglas. Su [configuración verificada](evidencias/ruleset-main.json)
queda incluida en esta entrega.

La [ejecución fallida](https://github.com/garze05/Lab-Api-Tareas/actions/runs/44646007342) publicó el artefacto
`reporte-pruebas` aunque pytest terminó con código 1. Se descargó y se verificó en
`junit.xml` el elemento `failure` de la misma prueba. El PR #2 ya mostraba un fallo
intencional anterior; el #4 comprueba también la conservación del reporte del Tema 3.

## 2. Diagnóstico de los tres pipelines rotos

### Caso 1

**Síntoma** (el paso que falló primero y su mensaje, copiado del log como texto):

Paso `Pruebas`, [ejecución fallida](https://github.com/garze05/Lab-Api-Tareas/actions/runs/44646016100), [PR #5](https://github.com/garze05/Lab-Api-Tareas/pull/5):

```text
/opt/hostedtoolcache/Python/3.12.16/x64/bin/python: No module named pytest
Process completed with exit code 1.
```

**Causa:** en `.github/workflows/ci.yml`, `Instalar dependencias` instala solamente
`requirements.txt` (Flask y gunicorn). `pytest` está declarado en
`requirements-dev.txt` y falta en el Python limpio del runner. La caché de pip
conserva descargas, pero no instala las dependencias del entorno.

Después también falla `Publicar reporte de pruebas`:

```text
No files were found with the provided path: reportes/. No artifacts will be uploaded.
```

Este segundo error es consecuencia del primero: pytest no arrancó, por lo que no
creó `reportes/junit.xml`. `if: always()` ejecuta la publicación; no crea el reporte.

**Corrección:** instalar `requirements-dev.txt`, que incluye `requirements.txt` y
las herramientas de prueba. No se cambió el código de la API.
Commit: https://github.com/garze05/Lab-Api-Tareas/pull/5/commits/b8145c13b388c1501cecf3392c338fcc1f47e3e6

La [ejecución corregida](https://github.com/garze05/Lab-Api-Tareas/actions/runs/44646139625) pasó el check `CI` y
publicó `reporte-pruebas` con 22 pruebas aprobadas.

**Prevención:** ejecutar siempre el script de pruebas en CI desde un entorno limpio
instalando `requirements-dev.txt`; revisar juntos los cambios al workflow y a las
listas de dependencias. El check obligatorio del ruleset impide integrar un fallo
como este y evita depender de paquetes instalados manualmente en la computadora local.

### Caso 2

**Síntoma** (el paso que falló primero y su mensaje, copiado del log como texto):

Paso `Publicar reporte de pruebas`, [ejecución fallida](https://github.com/garze05/Lab-Api-Tareas/actions/runs/44646023465), [PR #6](https://github.com/garze05/Lab-Api-Tareas/pull/6):

```text
No files were found with the provided path: reporte/. No artifacts will be uploaded.
```

**Causa:** `.github/workflows/ci.yml` busca `reporte/` (singular), pero
`scripts/pruebas.sh` escribe `reportes/junit.xml` (plural). El build y las 22 pruebas
pasan; falla la publicación por la ruta equivocada y `if-no-files-found: error`.

**Corrección:** cambiar únicamente `path: reporte/` por `path: reportes/`.
Commit: https://github.com/garze05/Lab-Api-Tareas/pull/6/commits/6f302de830c9ccc19ce4db50e27a5531548e5a71

La [ejecución corregida](https://github.com/garze05/Lab-Api-Tareas/actions/runs/44646143564) pasó el check `CI` y
publicó `reporte-pruebas` con 22 pruebas aprobadas.

**Prevención:** revisar que la ruta generada por el script coincida con la del paso
de publicación y descargar el artefacto al cambiar cualquiera de ellas. Mantener
`if-no-files-found: error` y el check obligatorio: las pruebas aprobadas por sí solas
no garantizan que se haya conservado la evidencia.

### Caso 3

**Síntoma** (el paso que falló primero y su mensaje, copiado del log como texto):

Paso `Pruebas`, [ejecución fallida](https://github.com/garze05/Lab-Api-Tareas/actions/runs/44646031153), [PR #7](https://github.com/garze05/Lab-Api-Tareas/pull/7):

```text
E       FileNotFoundError: [Errno 2] No such file or directory: '/home/runner/work/Lab-Api-Tareas/Lab-Api-Tareas/tests/datos/tareas_ejemplo.json'
ERROR tests/test_datos_ejemplo.py::test_carga_las_tareas_de_ejemplo - FileNotFoundError: [Errno 2] No such file or directory: '/home/runner/work/Lab-Api-Tareas/Lab-Api-Tareas/tests/datos/tareas_ejemplo.json'
========================= 22 passed, 1 error in 0.25s ==========================
```

**Causa:** `.gitignore` contiene `*.json`, una regla demasiado amplia para ignorar
reportes de cobertura. Al copiar los archivos y hacer `git add -A`, el archivo
`tests/datos/tareas_ejemplo.json` queda en el disco local pero no entra al commit.
El runner obtiene el contenido versionado y la fixture `tareas_ejemplo` falla al
leer el JSON. Se comprobó la regla responsable con `git check-ignore -v`:

```text
.gitignore:13:*.json	tests/datos/tareas_ejemplo.json
```

Es un error al preparar la fixture, antes del assert de esa prueba. En este caso
`Publicar reporte de pruebas` sí termina correctamente y conserva el JUnit del error.

**Corrección:** reemplazar `*.json` por `coverage.json`, limitado al reporte que se
quería excluir, y agregar `tests/datos/tareas_ejemplo.json` al commit. No se modificó
la prueba para ocultar el archivo faltante.
Commit: https://github.com/garze05/Lab-Api-Tareas/pull/7/commits/c915b6e9345ad75fc85bcb486c8251d49be4a609

La [ejecución corregida](https://github.com/garze05/Lab-Api-Tareas/actions/runs/44646149385) pasó el check `CI` y
publicó `reporte-pruebas` con 23 pruebas aprobadas.

**Prevención:** evitar reglas globales por extensión para archivos que forman parte
de las pruebas; revisar `git status --ignored` y `git ls-files tests/datos` al
agregar fixtures. Ejecutar CI sobre el checkout limpio y exigir su check detecta
archivos presentes solamente en la computadora local.

Los PR #5, #6 y #7 se corrigieron en sus mismas ramas hasta obtener CI en verde y
se cerraron sin merge. Los archivos de los casos no forman parte de `main`.

## 3. Uso de IA

Se utilizó OpenAI Codex para interpretar las consignas, configurar y verificar CI,
reproducir y corregir los tres casos en GitHub Actions, y redactar este diagnóstico
con los logs observados y sus enlaces de evidencia.

## 4. Verificación de la entrega

- El [PR #3](https://github.com/garze05/Lab-Api-Tareas/pull/3) incorpora build, caché de pip y reporte JUnit.
- El workflow ejecuta CI en `push` a `main` y en `pull_request` hacia `main`, con
  Python 3.12, dependencias con versiones fijas y permisos `contents: read`.
- `Validar (build)` compila `app` e importa y crea la aplicación sin levantar un servidor.
- Las pruebas usan `bash scripts/pruebas.sh`; el reporte se publica con
  `if: always()` y `if-no-files-found: error` como `reporte-pruebas`.
- El tag `lab-ci` identifica el estado final integrado a `main`; se comprueba el CI
  verde de ese commit y su artefacto antes de descargar el ZIP del tag.
- Archivo para Moodle: **api-tareas-lab-ci.zip**. La plataforma define la fecha y
  hora de cierre; estas consignas no incluyen una fecha concreta.

El mapa en `docs/mapa-cicd.md` describe la estrategia completa. Este laboratorio
implementa integración continua; staging, aprobación de producción y rollback
pertenecen a las siguientes etapas del curso.
