# 2️ Creación de una pipeline CI/CD
# Explicación del diseño de la pipeline
Para comprender como se crea un pipeline, comienzo leyendo la siguiente web:
https://apuntes.de/gitlab-integracion-continua/construir-nuestro-primer-pipeline/#gsc.tab=0
Esta lectura me ayudó a entender la filosofía de **Integración Continua (CI)** y **Entrega Continua (CD)**, donde cada commit al repositorio puede desencadenar automáticamente:
- La compilación del proyecto
- La ejecución de tests
- El análisis de seguridad
- Y, finalmente, el despliegue de la aplicación

## Automatizar el proceso de construcción, pruebas y despliegue de la aplicación en Gitlab
Para automatizar el proceso creamos un pipeline primeramente con 4 etapas: build, test, package y deploy. 
Así, cada etapa realiza una parte del proceso, compila, ejecuta empaqueta y por último despliega el proyecto
Cada una realiza una función concreta:

| Etapa | Descripción |
|--------|--------------|
| **build** | Compila el proyecto Java con Maven (`mvn clean compile`) |
| **test** | Ejecuta las pruebas unitarias (`mvn test`) y guarda los reportes |
| **package** | Empaqueta el código fuente en un archivo `.jar` listo para despliegue |
| **deploy** | Crea una imagen Docker y lanza la aplicación PetClinic en el puerto 5005 |

Esto automatiza completamente el ciclo de vida de la aplicación: **compilación → pruebas → empaquetado → ejecución.**

Posteriormente añadí una **nueva etapa llamada `security`**, que integra análisis de seguridad automáticos dentro del flujo DevSecOps.

### 📘 Estructura general

```yaml
stages:
  - build
  - test
  - security
  - excel
  - package
  - deploy
```

---

### 🔹 Escaneo de código (SAST)

He añadido un job específico `sast` basado en **Semgrep**, que revisa el código fuente buscando vulnerabilidades:

```yaml
sast:
  stage: security
  image: returntocorp/semgrep
  script:
    - semgrep ci --json > gl-sast-report.json
  artifacts:
    reports:
      sast: gl-sast-report.json
```

**Propósito:**  
Detecta vulnerabilidades en el código fuente (por ejemplo, exposición de endpoints en Spring Boot Actuator) y falla el pipeline si encuentra alguna **crítica o alta**.

---

### 🔹 Escaneo de dependencias (SCA)

El escaneo de dependencias revisa librerías vulnerables en el archivo `pom.xml`.  
Para ello, GitLab ofrece una plantilla preconfigurada que incluí en mi pipeline:

```yaml
include:
  - template: Security/Dependency-Scanning.gitlab-ci.yml
```

**Propósito:**  
Identificar versiones inseguras de dependencias declaradas en Maven, por ejemplo vulnerabilidades conocidas en `spring-webmvc` o `jackson-databind`.

---

### 🔹 Escaneo de imágenes Docker (Trivy)

Para el análisis de contenedores utilicé **Trivy**, que inspecciona las capas de la imagen generada:

```yaml
container_scanning:
  stage: security
  image: docker:24-cli
  services: []
  variables:
    CS_IMAGE: "$IMAGE_TAG"
  script:
    - docker build -t $CS_IMAGE .
    - apk add --no-cache curl
    - curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh
    - ./bin/trivy image --format json --output gl-container-scanning-report.json --exit-code 0 --severity CRITICAL,HIGH $CS_IMAGE
```

**Propósito:**  
Revisar vulnerabilidades en el sistema base (`eclipse-temurin:25-jdk`) o dependencias empaquetadas.  

---

### 🔹 Generación de reportes de seguridad

GitLab genera automáticamente reportes visuales de las etapas `sast`, `dependency-scanning` y `container-scanning`, visibles desde la interfaz del pipeline.

Ejemplo de configuración:

```yaml
artifacts:
  reports:
    sast: gl-sast-report.json
    container_scanning: gl-container-scanning-report.json
```


## Implementar notificaciones y logging en Gitlab:
Añadí echos por todo el código para poder ver por terminal qué estaba sucediendo en el pipeline.
Para capturar logs detallados de cada etapa utilicé redirección de salida con `tee` y almacenamiento como artefactos:

```yaml
test:
  script:
    - mvn test | tee test-logs.txt
  artifacts:
    paths:
      - test-logs.txt
```

Además:
- Añadí logs específicos (`build.log`, `deploy.log`, `container.log`).
- GitLab conserva automáticamente los logs en la interfaz web para auditoría.
- Todos los registros se archivan como artefactos, facilitando su revisión posterior.


## Diferenciar entre Continuous Deployment y Continuous Delivery en la pipeline
o    No estarán activas al mismo tiempo, pero debes poder cambiar entre ellas
o    ¿Cuál es mejor para este proyecto y por qué?
En la etapa deploy, se usa:

when: manual

Esto significa que el despliegue no se ejecuta automáticamente, sino que necesita una aprobación manual. Es lo que se llama Continuous Delivery. Para activar 
Continuous Delivery es más apropiado porque permite revisar el resultado antes de desplegar.
Recordar que es un entorno de pruebas/demostración (PetClinic)  en el que no hay un servidor productivo real conectado.
| Concepto | Continuous Delivery | Continuous Deployment |
|-----------|--------------------|------------------------|
| **Descripción** | El código pasa por build, test y análisis, quedando **listo para desplegar**, pero el despliegue requiere **aprobación manual**. | El pipeline **despliega automáticamente** en producción tras pasar todas las validaciones. |
| **Ejecución** | `when: manual` en el job de deploy. | `when: always` o ejecución automática tras merge. |
| **Control** | Permite revisar resultados antes del despliegue. | Requiere absoluta confianza en los tests y controles automáticos. |
| **Adecuación al proyecto** | Ideal para entornos de **pruebas o demostración**. | Ideal para proyectos en **producción continua**. |

En este proyecto, el despliegue usa:

```yaml
deploy_local:
  stage: deploy
  when: manual
```
Esto implementa **Continuous Delivery**, ya que PetClinic es un entorno de pruebas sin servidor productivo real conectado.

## Usar Gitlab credentials para almacenar información sensible de forma segura
No se ponen contraseñas directamente en el YAML. 
En su lugar, usamos Variables de entorno seguras en GitLab:

Ir a tu repositorio → Settings > CI/CD > Variables

Añadir claves como:

DOCKER_USER
DOCKER_PASSWORD
DEPLOY_KEY

script:
  - docker login -u "$DOCKER_USER" -p "$DOCKER_PASSWORD"

Esto mantiene la información sensible protegida y fuera del código.

Entrega y documentación en Markdown específico:
•         Comandos utilizados y su propósito
•         Resultados de los análisis de seguridad en formato excel o CSV(mejora opcional)
•         Diferencias entre Continuous Deployment y Continuous Delivery

## Problemas encontrados y cómo los solucionaste
Durante la construcción del pipeline y su ejecución en GitLab CI/CD, se presentaron diversos errores tanto de configuración como de compatibilidad. A continuación, se documentan de forma detallada, junto con las acciones tomadas para resolverlos.

- Error de despliegue en `localhost:5005` 
El pipeline finalizaba correctamente, pero al acceder a `http://localhost:5005` aparecía el error:  `ERR_EMPTY_RESPONSE — localhost didn’t send any data.`
El contenedor Docker ejecutaba la aplicación en el puerto **8080**, pero el `docker run` del job usaba el puerto **5005**. Para solucionarlo actualizo el `Dockerfile` y el job `deploy_local` para mapear el puerto correcto:  
docker run -d --name petclinic -p 8080:8080 $IMAGE_TAG

- Durante el despliegue, el pipeline fallaba con:
Conflict. The container name "/petclinic" is already in use by container ...
El contenedor anterior no se eliminaba entre ejecuciones. Añadir una línea para eliminar el contenedor si existe antes de crear uno nuevo: docker rm -f petclinic || true
Esto permite relanzar el despliegue sin conflictos.

- Error en el escaneo con Trivy: formato no válido
El job container_scanning fallaba con: invalid argument "gitlab" for --format flag
La versión reciente de Trivy (0.67.2) ya no soporta el formato gitlab para la salida.
Sustituir el comando por un formato compatible:
./bin/trivy image --format json --output gl-container-scanning-report.json --exit-code 0 --severity CRITICAL,HIGH $CS_IMAGE

# Conclusiones:
- **CI/CD completo**: compilación, test, empaquetado y despliegue automatizado.  
- **Seguridad integrada (DevSecOps)**: análisis SAST, SCA y Container Scanning.  
- **Reporting opcional**: exportación a Excel/CSV de vulnerabilidades.  
- **Gestión segura de credenciales** mediante variables protegidas.  
- **Entrega continua (Continuous Delivery)** con aprobación manual.  
- **Logs completos** y trazabilidad de cada fase.

La aplicación **Spring PetClinic** se despliega correctamente en Docker (`localhost:5005`),  
y los reportes de seguridad se generan sin errores, consolidándose en un flujo **DevSecOps profesional y reproducible**.

