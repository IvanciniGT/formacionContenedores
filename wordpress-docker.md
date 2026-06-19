Perfecto. Aquí lo tienes en **formato Markdown**, pensado como **documento de entrega**, no como respuesta rápida.

````markdown
# Despliegue de WordPress y MariaDB con Docker

## 1. Objetivo

El objetivo de este documento es desplegar una aplicación WordPress con una base de datos MariaDB utilizando comandos Docker tradicionales.

El despliegue se basa en una arquitectura equivalente a la que se podría definir en Kubernetes mediante `Deployment`, `Service`, `Secret`, `PersistentVolumeClaim` e `Ingress`, pero adaptada a un entorno Docker simple.

En este caso no se utilizarán volúmenes Docker nombrados. La persistencia se realizará mediante **bind mounts**, es decir, montando carpetas locales del sistema anfitrión dentro de los contenedores.

---

## 2. Arquitectura del despliegue

El despliegue estará compuesto por dos contenedores:

- Un contenedor MariaDB.
- Un contenedor WordPress.

Ambos contenedores estarán conectados a una red Docker común. WordPress se comunicará con MariaDB utilizando el nombre del contenedor de base de datos como nombre DNS interno.

La base de datos no se expondrá hacia el exterior. Solamente se expondrá el servicio web de WordPress.

---

## 3. Equivalencia conceptual con Kubernetes

| Kubernetes | Docker |
|---|---|
| Namespace | Red Docker y nombres de contenedor |
| Secret | Variables de entorno |
| Deployment MariaDB | Contenedor MariaDB |
| Deployment WordPress | Contenedor WordPress |
| PersistentVolumeClaim | Bind mount a carpeta local |
| Service MariaDB | Resolución DNS interna en red Docker |
| Service WordPress | Publicación de puerto con `-p` |
| Ingress | Acceso directo por puerto o reverse proxy externo |

---

## 4. Imágenes utilizadas

Para este despliegue se utilizarán las siguientes imágenes:

```bash
mariadb:11.4
wordpress:php8.3-apache
```

---

## 5. Datos de configuración

Los datos de configuración de la base de datos serán los siguientes:

| Parámetro | Valor |
|---|---|
| Base de datos | `wordpress` |
| Usuario | `wordpress` |
| Contraseña usuario | `wp-curso-2026` |
| Contraseña root | `root-curso-2026` |
| Host de base de datos | `mariadb` |
| Puerto WordPress en host | `8080` |
| Puerto WordPress en contenedor | `80` |

---

## 6. Crear estructura de directorios

Antes de crear los contenedores, se preparan las carpetas locales que se utilizarán como almacenamiento persistente.

```bash
mkdir -p ./curso-ivan/mariadb-data
mkdir -p ./curso-ivan/wordpress-data
```

Estas carpetas se montarán dentro de los contenedores:

| Carpeta local | Ruta dentro del contenedor | Uso |
|---|---|---|
| `./curso-ivan/mariadb-data` | `/var/lib/mysql` | Datos de MariaDB |
| `./curso-ivan/wordpress-data` | `/var/www/html` | Ficheros de WordPress |

---

## 7. Crear la red Docker

Se crea una red Docker dedicada para que los contenedores puedan comunicarse entre sí.

```bash
docker network create curso-ivan-net
# alias:
# docker network create curso-ivan-net
```

En Docker, los contenedores conectados a la misma red pueden resolverse entre sí usando su nombre.

Por ejemplo, si el contenedor de base de datos se llama `mariadb`, WordPress podrá conectarse usando:

```text
mariadb
```

---

## 8. Crear el contenedor de MariaDB

Se crea el contenedor de MariaDB con las variables necesarias para inicializar la base de datos.

```bash
docker run -d \
  --name mariadb \
  --network curso-ivan-net \
  -e MARIADB_ROOT_PASSWORD="root-curso-2026" \
  -e MARIADB_DATABASE="wordpress" \
  -e MARIADB_USER="wordpress" \
  -e MARIADB_PASSWORD="wp-curso-2026" \
  -v "$(pwd)/curso-ivan/mariadb-data:/var/lib/mysql" \
  mariadb:11.4

# alias:
# docker container run -d \
#   --name mariadb \
#   --network curso-ivan-net \
#   -e MARIADB_ROOT_PASSWORD="root-curso-2026" \
#   -e MARIADB_DATABASE="wordpress" \
#   -e MARIADB_USER="wordpress" \
#   -e MARIADB_PASSWORD="wp-curso-2026" \
#   -v "$(pwd)/curso-ivan/mariadb-data:/var/lib/mysql" \
#   mariadb:11.4
```

### Explicación del comando

| Opción | Descripción |
|---|---|
| `docker run -d` | Crea y arranca el contenedor en segundo plano |
| `--name mariadb` | Asigna el nombre `mariadb` al contenedor |
| `--network curso-ivan-net` | Conecta el contenedor a la red Docker creada |
| `-e MARIADB_ROOT_PASSWORD` | Define la contraseña del usuario root de MariaDB |
| `-e MARIADB_DATABASE` | Crea una base de datos inicial |
| `-e MARIADB_USER` | Crea un usuario de base de datos |
| `-e MARIADB_PASSWORD` | Define la contraseña del usuario |
| `-v ...:/var/lib/mysql` | Monta una carpeta local para persistir los datos |
| `mariadb:11.4` | Imagen utilizada para el contenedor |

---

## 9. Comprobar el estado de MariaDB

Para comprobar que el contenedor está en ejecución:

```bash
docker ps
# alias:
# docker container ls
```

Para consultar los logs del contenedor:

```bash
docker logs mariadb
# alias:
# docker container logs mariadb
```

Si MariaDB ha arrancado correctamente, se podrá acceder a la base de datos con:

```bash
docker exec -it mariadb mariadb -uwordpress -pwp-curso-2026 wordpress

# alias:
# docker container exec -it mariadb mariadb -uwordpress -pwp-curso-2026 wordpress
```

Dentro de MariaDB se puede ejecutar:

```sql
SHOW DATABASES;
EXIT;
```

---

## 10. Crear el contenedor de WordPress

Una vez creada la base de datos, se crea el contenedor de WordPress.

```bash
docker run -d \
  --name wordpress \
  --network curso-ivan-net \
  -p 8080:80 \
  -e WORDPRESS_DB_HOST="mariadb" \
  -e WORDPRESS_DB_NAME="wordpress" \
  -e WORDPRESS_DB_USER="wordpress" \
  -e WORDPRESS_DB_PASSWORD="wp-curso-2026" \
  -v "$(pwd)/curso-ivan/wordpress-data:/var/www/html" \
  wordpress:php8.3-apache

# alias:
# docker container run -d \
#   --name wordpress \
#   --network curso-ivan-net \
#   -p 8080:80 \
#   -e WORDPRESS_DB_HOST="mariadb" \
#   -e WORDPRESS_DB_NAME="wordpress" \
#   -e WORDPRESS_DB_USER="wordpress" \
#   -e WORDPRESS_DB_PASSWORD="wp-curso-2026" \
#   -v "$(pwd)/curso-ivan/wordpress-data:/var/www/html" \
#   wordpress:php8.3-apache
```

### Explicación del comando

| Opción | Descripción |
|---|---|
| `docker run -d` | Crea y arranca el contenedor en segundo plano |
| `--name wordpress` | Asigna el nombre `wordpress` al contenedor |
| `--network curso-ivan-net` | Conecta WordPress a la misma red que MariaDB |
| `-p 8080:80` | Publica el puerto 80 del contenedor en el puerto 8080 del host |
| `-e WORDPRESS_DB_HOST` | Indica el host de la base de datos |
| `-e WORDPRESS_DB_NAME` | Indica el nombre de la base de datos |
| `-e WORDPRESS_DB_USER` | Indica el usuario de la base de datos |
| `-e WORDPRESS_DB_PASSWORD` | Indica la contraseña de la base de datos |
| `-v ...:/var/www/html` | Monta una carpeta local para persistir los ficheros de WordPress |
| `wordpress:php8.3-apache` | Imagen utilizada para WordPress |

---

## 11. Acceder a WordPress

Una vez creado el contenedor, se puede acceder a WordPress desde el navegador mediante la siguiente URL:

```text
http://localhost:8080
```

Si Docker se está ejecutando en una máquina remota, se deberá utilizar la IP o nombre DNS de dicha máquina:

```text
http://IP_DEL_SERVIDOR:8080
```

---

## 12. Comprobar el estado de WordPress

Para comprobar que el contenedor está en ejecución:

```bash
docker ps
# alias:
# docker container ls
```

Para ver los logs del contenedor:

```bash
docker logs wordpress
# alias:
# docker container logs wordpress
```

---

## 13. Comprobar la resolución interna entre contenedores

WordPress se conecta a MariaDB usando el nombre `mariadb`.

Para comprobar que WordPress puede resolver ese nombre:

```bash
docker exec -it wordpress bash
# alias:
# docker container exec -it wordpress bash
```

Dentro del contenedor:

```bash
getent hosts mariadb
```

La salida debería mostrar una IP interna de Docker asociada al nombre `mariadb`.

Ejemplo:

```text
172.18.0.2 mariadb
```

Esto confirma que ambos contenedores están en la misma red Docker y que la resolución DNS interna funciona correctamente.

---

## 14. Ver información detallada de los contenedores

Para inspeccionar el contenedor de MariaDB:

```bash
docker inspect mariadb
# alias:
# docker container inspect mariadb
```

Para inspeccionar el contenedor de WordPress:

```bash
docker inspect wordpress
# alias:
# docker container inspect wordpress
```

---

## 15. Entrar en los contenedores

Para entrar en el contenedor de MariaDB:

```bash
docker exec -it mariadb bash
# alias:
# docker container exec -it mariadb bash
```

Para entrar en el contenedor de WordPress:

```bash
docker exec -it wordpress bash
# alias:
# docker container exec -it wordpress bash
```

---

## 16. Parar los contenedores

Para detener WordPress y MariaDB:

```bash
docker stop wordpress
docker stop mariadb

# alias:
# docker container stop wordpress
# docker container stop mariadb
```

---

## 17. Arrancar los contenedores de nuevo

Para arrancar los contenedores de nuevo:

```bash
docker start mariadb
docker start wordpress

# alias:
# docker container start mariadb
# docker container start wordpress
```

Es recomendable arrancar primero MariaDB y después WordPress, ya que WordPress necesita conectarse a la base de datos.

---

## 18. Eliminar los contenedores

Primero se detienen los contenedores:

```bash
docker stop wordpress
docker stop mariadb

# alias:
# docker container stop wordpress
# docker container stop mariadb
```

Después se eliminan:

```bash
docker rm wordpress
docker rm mariadb

# alias:
# docker container rm wordpress
# docker container rm mariadb
```

Este proceso elimina los contenedores, pero no elimina los datos, porque los datos están almacenados en carpetas locales mediante bind mounts.

---

## 19. Eliminar la red Docker

Para eliminar la red:

```bash
docker network rm curso-ivan-net
# alias:
# docker network rm curso-ivan-net
```

---

## 20. Eliminar también los datos persistentes

Si se desea eliminar completamente la instalación, incluyendo los datos de MariaDB y los ficheros de WordPress, se pueden borrar las carpetas locales:

```bash
rm -rf ./curso-ivan
```

Esta operación elimina de forma definitiva:

- Los datos de la base de datos.
- Los ficheros de WordPress.
- Plugins, temas y contenidos almacenados localmente.

---

## 21. Script completo de despliegue

El siguiente bloque resume todo el proceso de despliegue:

```bash
mkdir -p ./curso-ivan/mariadb-data
mkdir -p ./curso-ivan/wordpress-data

docker network create curso-ivan-net

docker run -d \
  --name mariadb \
  --network curso-ivan-net \
  -e MARIADB_ROOT_PASSWORD="root-curso-2026" \
  -e MARIADB_DATABASE="wordpress" \
  -e MARIADB_USER="wordpress" \
  -e MARIADB_PASSWORD="wp-curso-2026" \
  -v "$(pwd)/curso-ivan/mariadb-data:/var/lib/mysql" \
  mariadb:11.4

docker run -d \
  --name wordpress \
  --network curso-ivan-net \
  -p 8080:80 \
  -e WORDPRESS_DB_HOST="mariadb" \
  -e WORDPRESS_DB_NAME="wordpress" \
  -e WORDPRESS_DB_USER="wordpress" \
  -e WORDPRESS_DB_PASSWORD="wp-curso-2026" \
  -v "$(pwd)/curso-ivan/wordpress-data:/var/www/html" \
  wordpress:php8.3-apache
```

---

## 22. Script completo de parada

```bash
docker stop wordpress
docker stop mariadb
```

---

## 23. Script completo de arranque

```bash
docker start mariadb
docker start wordpress
```

---

## 24. Script completo de borrado sin eliminar datos

```bash
docker stop wordpress
docker stop mariadb

docker rm wordpress
docker rm mariadb

docker network rm curso-ivan-net
```

---

## 25. Script completo de borrado total

```bash
docker stop wordpress
docker stop mariadb

docker rm wordpress
docker rm mariadb

docker network rm curso-ivan-net

rm -rf ./curso-ivan
```

---

## 26. Consideraciones importantes

### Persistencia

La persistencia se consigue mediante bind mounts:

```bash
-v "$(pwd)/curso-ivan/mariadb-data:/var/lib/mysql"
-v "$(pwd)/curso-ivan/wordpress-data:/var/www/html"
```

Esto significa que los datos no dependen del ciclo de vida del contenedor.

Si se elimina el contenedor, los datos permanecen en el sistema anfitrión.

---

### Comunicación interna

WordPress se conecta a MariaDB usando:

```bash
WORDPRESS_DB_HOST="mariadb"
```

Esto funciona porque ambos contenedores están conectados a la misma red Docker:

```bash
--network curso-ivan-net
```

Docker proporciona resolución DNS interna dentro de las redes definidas por el usuario.

---

### Exposición de servicios

MariaDB no se publica hacia el exterior.

WordPress sí se publica:

```bash
-p 8080:80
```

Esto permite acceder desde el host mediante:

```text
http://localhost:8080
```

---

### Diferencia con Kubernetes

En Kubernetes, el acceso externo se podría resolver mediante un `Ingress` con TLS.

En este despliegue Docker simple no se configura TLS ni un nombre de dominio. Para conseguirlo sería necesario añadir un reverse proxy externo, por ejemplo:

- Nginx
- Apache HTTPD
- Traefik
- Caddy

---

## 27. Resultado final

Al finalizar el despliegue, se obtiene una instalación funcional de WordPress conectada a MariaDB, con persistencia local basada en bind mounts y acceso web mediante el puerto `8080` del host.

La URL de acceso será:

```text
http://localhost:8080
```
````
