# Mapa de estrategia CI/CD - API de tareas

> Autor o autora: Gabriel Rodríguez Ovares
> Punto de partida: `docs/DESPLIEGUE_MANUAL.md`, el proceso manual que el pipeline va a reemplazar.
> Cómo se ve un mapa terminado: `docs/ejemplo-mapa-cicd-sitio.md` (otro caso, mismo método).
> Sintaxis de los diagramas: la Guía rápida de Mermaid (Moodle, material de apoyo).

## 1. Flujo de commit a producción

```mermaid
flowchart TD
    subgraph local[Ambiente local]
        A[Cambio en una rama y pruebas locales]
    end
    A --> B[Commit, push y Pull Request]
    B --> C[Instalar dependencias con versiones fijas]
    C --> D[Validar build y crear la aplicación]
    D --> E[Pruebas automatizadas y reporte JUnit]
    E --> F{¿CI en verde?}
    F -- No --> X[Detener integración y corregir en la rama]
    X --> A
    F -- Sí --> G[Merge protegido a main]
    G --> H[Revalidar y generar artefacto identificado por SHA]
    subgraph staging[Ambiente staging]
        I[Desplegar artefacto probado]
        I --> J{¿Smoke test OK?}
    end
    H --> I
    J -- No --> Y[Notificar y detener promoción]
    J -- Sí --> K{¿Aprobación humana?}
    K -- No --> Y
    subgraph prod[Ambiente producción]
        L[Desplegar el mismo artefacto]
        L --> M{¿Validación OK?}
        M -- No --> N[Restaurar artefacto anterior y validar]
        M -- Sí --> O[Despliegue validado]
    end
    K -- Sí --> L
```

Este diagrama es la estrategia objetivo. En el laboratorio `lab-ci` se implementan CI, reportes
JUnit y protección de `main`; el empaquetado y los despliegues quedan para las siguientes etapas.


## 2. Ambientes

| Ambiente | Para qué sirve | Cómo llega un cambio | Quién decide que llegue |
|---|---|---|---|
| Local | Desarrollo del API y pruebas locales. | Quien desarrolla hace commits a una rama. | Quien desarrolla. |
| Staging | Verificar que los cambios introducidos no rompen funcionalidad existente, y que cumplen los criterios de aceptación. | Automático, después del merge a `main`. | Nadie: el pipeline. |
| Producción | La API que pueden utilizar otros aplicativos o clientes externos. | El mismo paquete que paso por staging. | Aprobador humano + pipeline. |

## 3. Riesgos del proceso manual y controles

| Paso | Riesgo | Qué puede pasar | Control del pipeline que lo atiende | Etapa del flujo |
|---|---|---|---|---|
| 1 | Entrar al servidor por SSH como `root` | Un atacante podría vulnerar el servidor de producción porque se tiene habilitado el acceso `root` | Usar una cuenta de despliegue sin acceso root, con permisos mínimos y credenciales gestionadas como secretos | Despliegue |
| 2 | Traer últimos cambios con git pull, si hay conflicto borrar la carpeta y volver a clonar | Posible perdida de artefactos generados en el servidor; Confiar en que la última versión de Git ya fue probada por el equipo de desarrollo; Posible desincronización al querer hacer rollback. | Generar un paquete identificado por el SHA del commit y desplegar el mismo paquete probado en staging. | Build y Despliegue |
| 3 | Instalar dependencias sin versiones fijas | Una actualización incompatible puede impedir que la API arranque o alterar su comportamiento. | Fijar las versiones de las dependencias y construir el paquete una sola vez para promoverlo entre ambientes. | Build |
| 4 | Configuración del token manual | Exponer el token o intentar arrancar el API sin él | Gestionar el token como secreto, inyectarlo en el ambiente de ejecución y comprobar que esté configurado sin mostrar su valor. | Despliegue |
| 5 | Detener el servicio con comando `kill` | La API deja de responder mientras se inicia la nueva versión; pueden perderse solicitudes activas. | Probar el arranque en staging y usar un reinicio controlado del servicio, con verificación de salud y rollback al artefacto anterior; evitar kill -9 | Despliegue |

## 4. Criterios de éxito

Qué tiene que ser verdad para decir que un despliegue salió bien, de forma que una máquina lo
pueda comprobar.

- `GET /tareas` devuelve HTTP 200 y una lista JSON en la URL desplegada.
- La API responde HTTP 200 y `{"estado": "ok"}` al realizar la petición GET al endpoint `/health` en la URL del API desplegado.
- La API responde la versión de la app actual junto al `APP_ENV` producción al realizar la petición get al endpoint `/version`
- La API crea una nueva tarea de forma exitosa, con código 201 al mandar una petición POST al endpoint `/tareas` con el formato `{"titulo": "..."}`, título de menos de 120 carácteres y el encabezado `Authorization: Bearer <token>` con el `API_TOKEN` inyectado.
- La tarea creada se consulta con `GET /tareas/<id>`: HTTP 200 y los mismos campos.
- Un `POST /tareas` sin token o con token incorrecto devuelve HTTP 401 cuando `API_TOKEN` está configurado.
- Si falla la validación en producción, se restaura el artefacto anterior y `/health` y `/version` confirman la recuperación. Los datos en memoria no se recuperan al reiniciar.

## 5. Decisiones y preguntas abiertas

- La aprobación humana va después de que el smoke test del entorno staging pasó, y antes de llegar a producción. El humano revisa lo que salió del pipeline, no lo que estaba en la computadora del desarrollador.
- Pregunta abierta: ¿En que horarios se despliega para evitar una posible afectación de usuarios?

Uso de IA: se utilizó para preguntas de aprendizaje sobre riesgos y controles; OpenAI Codex ayudó a revisar el flujo, explicitar las pruebas antes del merge y ajustar controles y criterios a la API de tareas.
