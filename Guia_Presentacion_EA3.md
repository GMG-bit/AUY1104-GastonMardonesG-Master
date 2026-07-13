# Guía de Defensa y Presentación en Vivo - EA3 (TechMarket)

Este documento sirve como apoyo visual y guión técnico durante la defensa en vivo de la evaluación **EA3**. Mapea cada indicador de la pauta de evaluación directamente con el código vivo del repositorio.

---

## 1. Mapeo de Indicadores de la Pauta de Evaluación

### Indicador 1: Diseño de Plantillas Reutilizables (8%)
*   **Qué mostrar en pantalla:** El archivo [.github/workflows/deploy-api.yaml](file:///.github/workflows/deploy-api.yaml) en GitHub.
*   **Qué decir (Argumento):**
    > *"Diseñamos una plantilla reutilizable centralizada bajo el evento `workflow_call`. Esto abstrae toda la complejidad de Kubernetes y Docker para los equipos de desarrollo. Los desarrolladores no necesitan saber cómo configurar SSH, compilar imágenes u orquestar rollbacks en K3s/EKS; simplemente invocan esta plantilla pasando 3 parámetros básicos."*

### Indicador 2: Integración de Componentes Externos (8%)
*   **Qué mostrar en pantalla:** Bloque de `steps` en [.github/workflows/deploy-api.yaml](file:///.github/workflows/deploy-api.yaml):
    *   `actions/checkout@v4` (Línea 55)
    *   `docker/login-action@v2` (Línea 71)
    *   `webfactory/ssh-agent@v0.9.0` (Línea 93)
*   **Qué decir (Argumento):**
    > *"Utilizamos componentes oficiales y mantenidos por la comunidad para agilizar el desarrollo y garantizar la seguridad. Por ejemplo, `webfactory/ssh-agent` maneja las llaves privadas SSH en memoria volátil del runner de forma enmascarada sin escribir archivos de texto plano en disco. Usar versiones fijas (ej. `@v4` y `@v2`) nos protege de cambios disruptivos ('breaking changes') y vulnerabilidades inyectadas en actualizaciones automáticas de terceros."*

### Indicador 3: Parametrización y Reutilización (8%)
*   **Qué mostrar en pantalla:** La cabecera `on.workflow_call` (Líneas 11-33) en [deploy-api.yaml](file:///.github/workflows/deploy-api.yaml).
*   **Qué decir (Argumento):**
    > *"El flujo está 100% parametrizado. Recibe dinámicamente `image-name`, `image-tag` y la IP pública del servidor. Esto permite que el mismo pipeline sea reutilizado para desplegar en cualquier namespace, máquina EC2 o clúster de EKS diferente simplemente cambiando las variables de llamada, sin modificar una sola línea del código de despliegue central."*

### Indicador 4: Aporte al Negocio e Impacto Operativo (8%)
*   **Qué mostrar en pantalla:** El [README.md técnico](file:///README.md) en la sección *"Fundamentación Técnica"*.
*   **Qué decir (Argumento):**
    > *"Esta arquitectura aporta valor directo al negocio al reducir el Time-to-Market de nuevas características de forma segura. Al automatizar la validación de salud y la remediación, mitigamos pérdidas financieras causadas por caídas del servicio 'Orders', garantizando que los clientes de TechMarket nunca experimenten interrupciones en sus compras."*

### Indicador 5, 6 y 7: Estrategias de Despliegue (24% en total)
*   **Qué mostrar en pantalla:** Los manifiestos de demostración en [k8s/demo-blue-green/](file:///k8s/demo-blue-green/).
*   **Qué decir (Argumento):**
    > *"Para el microservicio transaccional 'Orders', seleccionamos la estrategia **Blue-Green**. A diferencia de un RollingUpdate simple que reemplaza pods gradualmente y puede propagar fallas a mitad de camino, o Recreate que causa downtime inmediato, Blue-Green mantiene dos entornos aislados de producción al mismo tiempo. El tráfico es conmutado al 100% modificando el selector del Service. Defendemos esta estrategia para 'Orders' porque, aunque consume temporalmente el doble de recursos (CPU/Memoria), prioriza el Uptime y permite un Rollback instantáneo sin pérdida de transacciones."*

---

## 2. Guión para la Demostración en Vivo (LIVE DEMO - 10%)

Sigue esta secuencia para demostrar el pipeline funcionando:

1.  **Muestra el repositorio Cliente** en la pestaña **Actions**.
2.  **Muestra un run exitoso** y navega por los logs en vivo:
    *   Señala el paso **`Deploy to k3s`**:
        > *"Aquí vemos cómo el pipeline aplica los manifiestos YAML y ejecuta el monitoreo `rollout status` esperando a que los pods de la nueva versión pasen las sondas de disponibilidad."*
    *   Muestra el resultado: `deployment "demo-api" successfully rolled out`.

---

## 3. Guión para la Simulación del Incidente (18% en total)

Si el docente introduce un fallo en vivo (o te pide que simules un error):

### A. Diagnóstico del Incidente (Mantener la Calma)
*   **Qué decir:**
    > *"La pipeline ha fallado en el paso `Deploy to k3s`. Esto se debe a que el comando `rollout status` ha superado el tiempo de espera máximo de 60 segundos debido a que las sondas de salud (readinessProbe) fallaron. Los nuevos pods están en estado no listos (0/1) o arrojando error de ejecución."*

### B. Auto-Healing en Acción
*   **Qué mostrar en pantalla:** El log del paso **`Rollback automático`** ejecutándose a continuación.
*   **Qué decir:**
    > *"Gracias a la condicional `if: failure()`, el fallo fue interceptado tempranamente. El pipeline inició la remediación de forma autónoma ejecutando `rollout undo`. Kubernetes eliminó los pods corruptos y restauró las réplicas de la versión estable anterior."*
*   **Verificación Final:** Muestra el log del paso `Verificar pods` demostrando que el pod estable original está corriendo y respondiendo peticiones.
    > *"Podemos verificar que el microservicio crítico de Orders está a salvo y el clúster volvió a su estado 100% estable de forma automática."*
