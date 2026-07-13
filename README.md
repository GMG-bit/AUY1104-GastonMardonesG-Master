# Recetas Reutilizables de CI/CD - TechMarket (Gobernanza DevOps)

Este repositorio contiene los flujos de trabajo reutilizables (Shared Workflows) de GitHub Actions implementados para centralizar la gobernanza de CI/CD del microservicio crítico **"Orders"** (representado por nuestra API académica `demo-api`) en la infraestructura de la organización **TechMarket**.

La centralización de estas recetas permite estandarizar las fases de construcción y despliegue, disminuyendo la duplicación de código en pipelines y asegurando la resiliencia en entornos productivos.

---

## 1. Contexto de Arquitectura y Homologación (EKS/ECR → K3s/Docker Hub)

Aunque el caso de negocio define el uso de **Amazon EKS** (Elastic Kubernetes Service) y **Amazon ECR** (Elastic Container Registry) en entornos corporativos reales, el clúster local **K3s** (Kubernetes ligero) en AWS Learners Lab implementa la misma API estándar de Kubernetes. 

Por lo tanto, la lógica de automatización diseñada en estas plantillas es 100% compatible y homologable entre K3s y EKS.
*   **Construcción y Registro:** La imagen es empaquetada e impulsada a Docker Hub (actuando como homólogo de Amazon ECR).
*   **Orquestación y Despliegue:** El clúster K3s en AWS ejecuta los manifiestos de Kubernetes de forma idéntica a un nodo trabajador de Amazon EKS.

---

## 2. Estructura de Recetas Centralizadas

El repositorio se compone de los siguientes archivos de configuración:

*   [deploy-api.yaml](file:///.github/workflows/deploy-api.yaml): Receta principal reutilizable de CI/CD. Realiza pruebas, compilación Docker, publicación en registro de contenedores y despliegue automatizado con verificación y rollback en el clúster.
*   [ea2-lab-dispatch-main.yaml](file:///.github/workflows/ea2-lab-dispatch-main.yaml): Automatización para el aprovisionamiento manual del clúster de Kubernetes en AWS Learners Lab (Sandbox).
*   [holamundo.yaml](file:///holamundo.yaml): Flujo de validación inicial básico para certificar llamadas entre repositorios (workflow_call).

---

## 3. Especificación Técnica de la Receta Reutilizable (`deploy-api.yaml`)

El flujo de despliegue se activa de forma remota desde el repositorio del microservicio mediante el evento `workflow_call`.

### Parámetros de Entrada (`inputs`)
| Variable | Tipo | Obligatorio | Descripción |
| :--- | :--- | :--- | :--- |
| `image-name` | `string` | Sí | Nombre asignado al contenedor en el registro (ej. `demo-api`). |
| `image-tag` | `string` | Sí | Etiqueta de versión asignada (comúnmente el tag de Git o SHA del commit). |
| `k3s-server-public-ip` | `string` | Sí | IP pública del servidor del clúster (EKS/K3s) para la conexión SSH. |

### Secretos Requeridos (`secrets`)
| Secreto | Obligatorio | Descripción |
| :--- | :--- | :--- |
| `DOCKER_USERNAME` | Sí | Credencial de usuario para el registro de imágenes. |
| `DOCKER_PASSWORD` | Sí | Token de acceso seguro o contraseña para el registro. |
| `EA2_SSH_PRIVATE_KEY` | Sí | Llave privada SSH para la autenticación en el servidor de AWS. |

---

## 4. Mecanismo de Resiliencia: Validación de Salud y Rollback

El trabajo de despliegue (`deploy-to-k8s`) implementa la estrategia de resiliencia requerida por la organización para evitar caídas de servicio:

```yaml
      - name: Deploy to k3s
        id: deploy
        run: |
          ssh -o StrictHostKeyChecking=accept-new "ubuntu@${{ inputs.k3s-server-public-ip }}" \
            "sudo k3s kubectl apply -f /tmp/k8s-deploy/ && sudo k3s kubectl rollout status deployment/${{ inputs.image-name }} --timeout=60s"
```
*   **Validación de Salud (Rollout Status):** Tras aplicar los manifiestos, el pipeline ejecuta `kubectl rollout status` con un `--timeout=60s`. Este comando monitorea que los pods pasen sus sondas de salud locales (`readinessProbe`). Si las sondas fallan, el rollout se detiene y el comando finaliza con un código de error (falla el paso).

```yaml
      - name: Rollback automático
        id: rollback
        if: failure()
        run: |
          ssh -o StrictHostKeyChecking=accept-new "ubuntu@${{ inputs.k3s-server-public-ip }}" \
            "sudo k3s kubectl rollout undo deployment/${{ inputs.image-name }} && sudo k3s kubectl rollout status deployment/${{ inputs.image-name }} --timeout=60s"
```
*   **Remediación Automática (Rollback Undo):** Configurado con la condición `if: failure()`. Si la validación de salud de Kubernetes expira por timeout (falla el deploy), este paso se activa de inmediato ejecutando `kubectl rollout undo`. Esto revierte el estado del clúster a la última versión estable documentada en la historia de Kubernetes sin intervención manual.

---

## 5. Declaración de Uso de Inteligencia Artificial

De acuerdo con las políticas éticas y académicas establecidas por Duoc UC, se declara que este proyecto ha sido desarrollado con el apoyo y co-programación de una inteligencia artificial generativa (**Antigravity, por Google DeepMind**), la cual asistió en la estructuración de la documentación técnica y en la optimización de los flujos de automatización de bash/SSH.

---

## 6. Referencias Bibliográficas (Norma APA)

*   Docker. (2024). *Docker documentation: Best practices for writing Dockerfiles*. Docker Docs. https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
*   Kubernetes. (2023). *Kubernetes Documentation: Deployments, Rollouts, and Rollback*. Kubernetes IO. https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
*   GitHub. (2024). *GitHub Actions documentation: Reusing workflows*. GitHub Docs. https://docs.github.com/en/actions/sharing-automations/reusing-workflows
