
# Instalación y securización de Gitlab
## Medidas de seguridad aplicadas y su impacto

### Instalar Gitlab en Docker y asegurarte de que los  datos persistan(volúmenes).

Comienzo viendo este vídeo para poder comenzar con la tarea:
https://www.bing.com/videos/riverview/relatedvideo?q=1.+Instalar+Gitlab+en+Docker+y+asegurarte+de+que+los+datos+persistan(vol%c3%bamenes).&mid=4E490F7FFD777513ECF54E490F7FFD777513ECF5&FORM=VIRE

El primer paso es escribir el fichero docker-compose.yml en el que añadiremos volúmenes. Con este fragmento sería suficiente para esta primera tarea:

services:
  gitlab:
    image: gitlab/gitlab-ce:17.6.2-ce.0
    container_name: gitlab
    restart: always
    ports:
      - "8000:80"
    volumes:
      - /home/isabel/gitlab/config:/etc/gitlab
      - /home/isabel/gitlab/logs:/var/log/gitlab
      - /home/isabel/gitlab/data:/var/opt/gitlab

A continuación, levanto el contenedor Gitlab:
    
    docker-compose up -d

Comprobamos, en el buscador copio http://localhost:8000 y automaticamente me sale ya el sing in de Gitlab

Por último, para iniciar sesión debemos poner en username la palabra root y la contraseña, en mi caso, habría caducado o no existía el comando ya que obtenía esto:

isabel@c11-ewta8kcoh5l:~/gitlab$ docker exec -it gitlab grep 'Passwor
d:' /etc/gitlab/initial_root_password
grep: /etc/gitlab/initial_root_password: No such file or directory

Por ello, miré como podía establecer una contraseña de root y seguí los pasos del siguiente enlace: [text](https://docs.gitlab.com/security/reset_user_password/?utm_source=chatgpt.com)
Consiguiendo así entrar con la contraseña que he elegido.
![alt text](<Captura de pantalla 2025-10-20 100018-1.png>)

### Configurar la seguridad de Gitlab, aplicando medidas de mejores prácticas y seguridad. 
- Se ha desplegado la versión gitlab/gitlab-ce:17.6.2-ce.0, que incluye los últimos parches de seguridad de octubre de 2025
- Deshabilitar el registro público y fuerza visibilidad privada
    - La ruta es Admin → Settings → General
    - Desactivo Allow new users to sign up.
    - Visibility and access controls: Default project visibility → Private; Default group visibility → Private.
- Obligar 2FA
    - La ruta es Admin → Settings → General → Sign-in restrictions
    - Activo Require two-factor authentication con Tiempo de gracia sugerido: 24–72 h
- Política de contraseñas
    - La ruta es Admin → Settings → General → Account and limit settings
    - Establezco Password length mínima: 12 y el Bloqueo por intentos fallidos: 10

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

  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    container_name: gitlab-runner
    restart: unless-stopped
    depends_on:
      - gitlab
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /home/isabel/gitlab/runner-config.toml:/etc/gitlab-runner/config.toml
   
Una vez añadido, guardamos y volvemos a ejectar docker compose up
Comprobamos con docker ps si ambos contenedores estan activos (tanto gitlab como gitlab-runner)                                                       
Ejecutamos en la terminal el siguiente comando:

docker exec -it gitlab-runner gitlab-runner register

Y vamos respondiendo a las preguntas:

Enter the GitLab instance URL (for example, https://gitlab.com/):
http://gitlab:80
Enter the registration token:
yoTCscy4PyAvBzbBCACp
Enter a description for the runner:
[1d58baf2ca92]: runner-docker
Enter tags for the runner (comma-separated):

Enter optional maintenance note for the runner:
Registering runner... succeeded                     correlation_id=01K80M9E3MVXJGFF7EP35NTWC7 runner=yoTCscy4P
Enter an executor: ssh, docker, docker-windows, docker-autoscaler, custom, shell, parallels, virtualbox, docker+machine, kubernetes, instance:
docker
Enter the default Docker image (for example, ruby:3.3):
cimg/base:stable
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!

y efectivamente nuestro runner queda registrado con éxito.
Por último, vamos a http://localhost:8000/ y entramos en admin area -> CI/CD -> runners y debe salir ahí:
![alt text](<Captura de pantalla 2025-10-20 130950-1.png>)

### Problemas encontrados y cómo los solucionaste
Durante la instalación y configuración del GitLab Runner surgieron varios problemas relacionados principalmente con la conexión entre los contenedores, la configuración de volúmenes y la validación del registro automático. A continuación se detallan los principales errores y cómo se resolvieron.

---

#### 1. El contenedor del runner se reiniciaba continuamente

Inicialmente, he tenido que crear el contener gitlab un total de 3 veces ya que ha habido muchos errores relacionados con los volúmenes.

- Advertencia por uso de atributo version en Docker Compose:
Docker Compose informó que el atributo version está obsoleto y recomendó eliminarlo para evitar confusiones. Simplemente lo eliminé del archivo docker-compose.yml y no afectó la funcionalidad.

Al empezar a hacer el contenedor `gitlab-runner` entraba en un bucle de reinicio con el mensaje de error:

FATAL: Service run failed error=decoding configuration file: read /etc/gitlab-runner/config.toml: is a directory

Este fallo se debía a que el volumen `/home/isabel/gitlab/runner-config.toml` se había creado como **carpeta en lugar de archivo**.  
La solución fue eliminar el directorio y crear un archivo vacío con el mismo nombre:

rm -rf /home/isabel/gitlab/runner-config.toml
touch /home/isabel/gitlab/runner-config.toml

Tras reiniciar los contenedores, el runner comenzó a iniciarse correctamente.


- Problemas con la validación del archivo docker-compose.yml
En un momento dado, Docker arrojaba el error:

Additional property gitlab-runner is not allowed

Esto se debía a un error de indentación en el archivo YAML.
Se corrigió ajustando los espacios y añadiendo correctamente la línea services: al inicio del archivo.
Una vez corregido el formato, el archivo se validó con docker compose config y se levantó sin errores.

- GitLab Runner no aparecía en la interfaz web
Aunque el contenedor estaba activo y sin errores, el runner no aparecía en la sección de Runners del panel de administración.
El motivo era que el runner no se había registrado aún con un token válido.

La solución fue registrar manualmente el runner desde el contenedor con el comando:

docker exec -it gitlab-runner gitlab-runner register
usando como URL interna http://gitlab:80 y el token de registro obtenido desde la interfaz de GitLab.


- Registro del runner desde la web y conexión final
Finalmente, al crear el runner desde la interfaz web, GitLab lo mostraba como “Ocioso / Nunca contactado”.
Esto indicaba que el runner estaba creado pero no conectado.
El problema se resolvió copiando el token individual del runner  y repitiendo el registro manual dentro del contenedor.
Una vez completado, el runner apareció como Online (verde) y comenzó a ejecutar pipelines correctamente.


