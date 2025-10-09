
# Instalación y securización de Gitlab
## Medidas de seguridad aplicadas y su impacto

### Instalar Gitlab en Docker y asegurarte de que los  datos persistan(volúmenes).

Comienzo viendo este vídeo para poder comenzar con la tarea:
https://www.bing.com/videos/riverview/relatedvideo?q=1.+Instalar+Gitlab+en+Docker+y+asegurarte+de+que+los+datos+persistan(vol%c3%bamenes).&mid=4E490F7FFD777513ECF54E490F7FFD777513ECF5&FORM=VIRE

El primer paso es escribir el fichero docker-compose.yml en el que añadiremos volúmenes y límites.
A continuación, levanto el contenedor Gitlab:
    
    docker-compose up -d

Para lograr que funcione gitlab.local como hostname, he añadido la siguiente línea al archivo /etc/hosts:  127.0.0.1 gitlab.local
Esto permite que el sistema pueda resolver correctamente el nombre de dominio gitlab.local en la máquina local.

Por último y a modo de comprobación, en el buscador copio https://127.0.0.1 y automaticamente me sale ya el sing in de Gitlab

### Configurar la seguridad de Gitlab, aplicando medidas de mejores prácticas y seguridad. 
| Medida de Seguridad                                                   | Código                                                                                                                                                                                                    | Fuente                                                                                   |
| --------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| **Habilitar HTTPS con certificados personalizados (TLS)**          | `<br>external_url 'https://gitlab.local'<br>nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.local.crt"<br>nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.local.key"<br>nginx['redirect_http_to_https'] = true<br>` | [GitLab Docs - Enable HTTPS](https://docs.gitlab.com/omnibus/settings/nginx.html#enable-https)               |
| **Deshabilitar registro de usuarios nuevos**                       | `<br>gitlab_rails['gitlab_signup_enabled'] = false<br>`                                                                                                                                                                       | [GitLab Docs - Disable user signup](https://docs.gitlab.com/ee/administration/settings/signup.html)          |
| **Protección contra ataques de fuerza bruta con Rack Attack**      | `<br>gitlab_rails['rack_attack_git_basic_auth'] = {<br>  'enabled' => true,<br>  'maxretry' => 3,<br>  'findtime' => 60,<br>  'bantime' => 3600<br>}<br>`                                                                     | [GitLab Docs - Rack Attack](https://docs.gitlab.com/omnibus/settings/rack_attack.html#basic-auth-throttling) |
| **Cambiar el puerto SSH para evitar escaneos automatizados**       | `<br>gitlab_rails['gitlab_shell_ssh_port'] = 2222<br>ports:<br>  - "2222:22"<br>`                                                                                                                                             | [GitLab Docs - SSH Port](https://docs.gitlab.com/omnibus/settings/ssh.html#change-the-ssh-port)              |
| **Uso de volúmenes persistentes para configuración, datos y logs** | `<br>volumes:<br>  - gitlab-config:/etc/gitlab<br>  - gitlab-logs:/var/log/gitlab<br>  - gitlab-data:/var/opt/gitlab<br>`                                                                                                     | [GitLab Docs - Docker Volumes](https://docs.gitlab.com/omnibus/docker/#data-volumes)                         |
| **Redirección forzada de HTTP a HTTPS**                            | `<br>nginx['redirect_http_to_https'] = true<br>`                                                                                                                                                                              | [GitLab Docs - Redirect HTTP](https://docs.gitlab.com/omnibus/settings/nginx.html#redirect-http-to-https)    |

# Impacto de las medidas de seguridad aplicadas

- El uso de HTTPS asegura que toda la comunicación esté cifrada, protegiendo credenciales y datos.
- La redirección automática de HTTP a HTTPS evita accesos inseguros por error.
- Desactivar el registro de usuarios garantiza que solo administradores puedan crear cuentas, evitando registros no autorizados.
- La protección contra fuerza bruta limita los intentos de acceso fallidos, bloqueando IPs sospechosas automáticamente.
- Cambiar el puerto SSH a 2222 reduce ataques automatizados que escanean el puerto 22 por defecto.
- Usar volúmenes persistentes garantiza que los datos no se pierdan tras reiniciar o actualizar el contenedor.
- Definir un hostname local personalizado facilita el acceso seguro mediante HTTPS en entornos de prueba.
- Estas medidas fortalecen la seguridad sin complicar la experiencia del usuario, manteniendo un entorno controlado, fiable y privado.

Estas medidas fortalecen la seguridad sin complicar la experiencia del usuario, manteniendo un entorno controlado, fiable y privado.

### Configurar un runner en Gitlab para ejecutar las pipelines:
Para configurar un runner necesitamos añadir el siguiente código a docker-compose.yml:                 

        ```bash
        gitlab-runner:
            image: gitlab/gitlab-runner:latest
            container_name: gitlab-runner
            restart: always
            volumes:
            - /var/run/docker.sock:/var/run/docker.sock
            - ./gitlab-runner/config:/etc/gitlab-runner
                                                                 
Básicamente, con esto estamos dando permisos para que el runner pueda comunicarse con Docker y crear contenedores cuando sea necesario para ejecutar las tareas del pipeline.

La línea /var/run/docker.sock:/var/run/docker.sock conecta el runner con el motor de Docker del sistema, permitiéndole crear y controlar otros contenedores de forma segura.

La carpeta ./gitlab-runner/config es donde guardamos la configuración del runner para que no se pierda cuando se apaga o reinicia.

Esto garantiza que el runner pueda funcionar correctamente y ejecutar las tareas de integración continua que definamos en GitLab.

### Problemas encontrados y cómo los solucionaste
- Error al levantar el contenedor por nombre duplicado:
Al intentar levantar el contenedor GitLab con Docker, apareció un error porque ya existía un contenedor con el nombre gitlab. Para solucionarlo, eliminé el contenedor antiguo con docker rm -f gitlab y luego volví a crear el contenedor con Docker Compose.

- No se resolvía el hostname gitlab.local:
Al intentar acceder por https://gitlab.local o conectarme vía SSH con ese hostname, recibía errores de DNS porque el sistema no sabía cómo resolver ese nombre. Lo solucioné accediendo directamente a través de la IP 127.0.0.1 y puerto correspondiente (https://127.0.0.1), evitando la necesidad de configurar DNS o editar el archivo hosts.

- Error en docker-compose por configuración incorrecta:
Al añadir el servicio del runner, puse su configuración dentro de la sección networks, lo que generó un error. La solución fue mover la definición del runner a la sección correcta services, manteniendo las redes separadas.

- Advertencia por uso de atributo version en Docker Compose:
Docker Compose informó que el atributo version está obsoleto y recomendó eliminarlo para evitar confusiones. Simplemente lo eliminé del archivo docker-compose.yml y no afectó la funcionalidad.


