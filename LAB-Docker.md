# Lab de Docker — Contenedores en la práctica

> Este laboratorio refuerza los conceptos de la primera parte del temario (contenedores, imágenes, volúmenes, variables de entorno) y te familiariza con los **comandos de Docker**.
>
> Te conectarás a un entorno **Ubuntu** que corre en un clúster de Kubernetes y que tiene **Docker dentro** (Docker-in-Docker). Cada alumno tiene su propio entorno aislado: `entorno-formacion-<N>` (tu instructor te asigna el número `<N>`).
>
> **Convención de comandos.** Usamos la **sintaxis canónica** (`docker container ...`, `docker image ...`) y, al lado, el **alias corto** equivalente que se usa en el día a día.

---

## 0. Antes de empezar — conexión al entorno

Los comandos de esta sección se ejecutan **desde tu Windows** (PowerShell). El cliente que usamos es **`oc`** (el CLI de OpenShift, un superconjunto de `kubectl`).

### 0.1. Descargar `oc` (cliente de línea de comandos)

Descarga el cliente para Windows desde el GitHub de OKD (versión probada):

```
https://github.com/okd-project/okd/releases/download/4.22.0-okd-scos.3/openshift-client-windows-4.22.0-okd-scos.3.zip
```

1. Descomprime el ZIP. Dentro hay un único ejecutable: **`oc.exe`**.
2. Copia `oc.exe` a una carpeta de trabajo, por ejemplo `C:\formacion\`.
3. Abre **PowerShell** y sitúate en esa carpeta:

```powershell
cd C:\formacion
.\oc.exe version --client
```

Deberías ver algo como `Client Version: 4.22.0-okd-scos.3`.

### 0.2. Configurar el acceso al clúster

Copia el fichero **`kubeconfig-remoto.yaml`** (te lo facilita el instructor) a `C:\formacion\`. Indícale a `oc` que lo use mediante la variable de entorno `KUBECONFIG` (solo para esta ventana de PowerShell):

```powershell
$env:KUBECONFIG = "C:\formacion\kubeconfig-remoto.yaml"
```

Comprueba que llegas al clúster (sustituye el número por el tuyo):

```powershell
.\oc.exe get pods -n entorno-formacion
```

> Alternativa sin variable de entorno: añade `--kubeconfig C:\formacion\kubeconfig-remoto.yaml` a cada comando.

### 0.3. Entrar en tu entorno

Abre una shell `bash` **dentro** de tu entorno (cambia `11` por tu número `<N>`):

```powershell
.\oc.exe exec -it -n entorno-formacion entorno-formacion-11 -c dind -- bash
```

A partir de aquí, el *prompt* cambia (`root@entorno-formacion-11:/#`): **ya estás dentro de tu Ubuntu con Docker**. Todos los comandos siguientes se ejecutan aquí.

Comprueba el entorno:

```bash
cat /etc/os-release | grep PRETTY        # Ubuntu
docker version                            # cliente + servidor (daemon) Docker
docker compose version                    # plugin de Docker Compose
```

> **Para salir** del entorno: escribe `exit`. Para volver a entrar, repite el comando `oc exec` de 0.3.

> **Imágenes sin problemas.** Para evitar límites de descarga de Docker Hub, en este lab usamos siempre el espejo de Google: **`mirror.gcr.io/library/<imagen>`** (p. ej. `mirror.gcr.io/library/nginx:alpine`).

---

## 1. Básicos de imágenes y contenedores

Una **imagen** es el paquete (software ya instalado); un **contenedor** es una ejecución de esa imagen. Vamos a manejarlas con tres imágenes: **nginx**, **mariadb** y **ubuntu**.

### 1.1. Descargar imágenes (`pull`)

```bash
docker image pull mirror.gcr.io/library/nginx:alpine
docker image pull mirror.gcr.io/library/mariadb:11
docker image pull mirror.gcr.io/library/ubuntu:24.04
```

| Canónico | Alias corto |
|----------|-------------|
| `docker image pull <img>` | `docker pull <img>` |

### 1.2. Listar imágenes

```bash
docker image ls          # canónico
docker images            # alias
```

### 1.3. Ejecutar un contenedor que hace una tarea y termina

La imagen **ubuntu** trae solo una base del sistema. La usamos para ejecutar comandos sueltos:

```bash
docker container run --rm mirror.gcr.io/library/ubuntu:24.04 echo "Hola desde un contenedor Ubuntu"
docker container run --rm mirror.gcr.io/library/ubuntu:24.04 cat /etc/os-release
```

- `--rm` borra el contenedor automáticamente al terminar.

| Canónico | Alias corto |
|----------|-------------|
| `docker container run ...` | `docker run ...` |

### 1.4. Ejecutar un contenedor que se queda corriendo (servicio)

```bash
docker container run -d --name prueba-ubuntu mirror.gcr.io/library/ubuntu:24.04 sleep 3600
```

- `-d` = *detached* (segundo plano). `--name` le da un nombre.

### 1.5. Ver contenedores

```bash
docker container ls          # solo los que están corriendo  (alias: docker ps)
docker container ls -a       # todos, incluidos los parados    (alias: docker ps -a)
```

### 1.6. Entrar en un contenedor en marcha y ver sus logs

```bash
docker container exec -it prueba-ubuntu bash    # abre una shell dentro (alias: docker exec)
# (dentro:)  ls /   ;   exit
docker container logs prueba-ubuntu             # salida estándar del proceso (alias: docker logs)
```

### 1.7. Parar y borrar

```bash
docker container stop prueba-ubuntu             # alias: docker stop
docker container rm   prueba-ubuntu             # alias: docker rm   (con -f fuerza si está corriendo)
docker image rm mirror.gcr.io/library/ubuntu:24.04   # borra la imagen (alias: docker rmi)
```

---

## 2. Servir una web con nginx

### 2.1. Prueba de concepto

Arranca un nginx publicando su puerto 80 en el 8080 del entorno:

```bash
docker container run -d --name web1 -p 8080:80 mirror.gcr.io/library/nginx:alpine
docker container ls
curl -s localhost:8080 | grep -i title        # -> "Welcome to nginx!"
```

- `-p 8080:80` = *publicar* el puerto: `HOST:CONTENEDOR`.

### 2.2. Crear una web de ejemplo

Crea una carpeta con una página y dos enlaces (copia y pega el bloque entero):

```bash
mkdir -p /root/web
cat > /root/web/index.html <<'HTML'
<!DOCTYPE html>
<html lang="es">
<head><meta charset="utf-8"><title>Mi web</title></head>
<body>
  <h1>Hola desde mi web custom</h1>
  <ul>
    <li><a href="pagina1.html">Página 1</a></li>
    <li><a href="pagina2.html">Página 2</a></li>
  </ul>
</body>
</html>
HTML

cat > /root/web/pagina1.html <<'HTML'
<!DOCTYPE html><html lang="es"><head><meta charset="utf-8"><title>Página 1</title></head>
<body><h1>Esto es la página 1</h1><a href="index.html">Volver al inicio</a></body></html>
HTML

cat > /root/web/pagina2.html <<'HTML'
<!DOCTYPE html><html lang="es"><head><meta charset="utf-8"><title>Página 2</title></head>
<body><h1>Esto es la página 2</h1><a href="index.html">Volver al inicio</a></body></html>
HTML
```

### 2.3. Inyectar la web con `docker cp`

`docker cp` copia ficheros del entorno **al contenedor** (una foto puntual):

```bash
docker container cp /root/web/index.html   web1:/usr/share/nginx/html/index.html
docker container cp /root/web/pagina1.html web1:/usr/share/nginx/html/pagina1.html
docker container cp /root/web/pagina2.html web1:/usr/share/nginx/html/pagina2.html

curl -s localhost:8080                 # ahora muestra TU index.html
curl -s localhost:8080/pagina1.html    # y las páginas enlazadas
```

> **Concepto clave:** con `docker cp` los ficheros quedan **dentro del contenedor**. Si lo borras, se pierden. Y si editas el fichero del entorno, el contenedor **no** lo ve (hay que volver a copiarlo).

Limpia este contenedor antes de la siguiente prueba:

```bash
docker container rm -f web1
```

### 2.4. Servir la web desde un VOLUMEN (bind mount)

Ahora montamos la carpeta del entorno **dentro** del contenedor. El contenedor "ve en vivo" lo que haya en la carpeta:

```bash
docker container run -d --name web2 -p 8080:80 \
  -v /root/web:/usr/share/nginx/html:ro \
  mirror.gcr.io/library/nginx:alpine

curl -s localhost:8080 | grep -i h1     # -> "Hola desde mi web custom"
```

- `-v /root/web:/usr/share/nginx/html:ro` = montar carpeta `HOST:CONTENEDOR`, `:ro` solo lectura.

Edita el fichero en el entorno y recarga **sin tocar el contenedor**:

```bash
sed -i 's/mi web custom/mi web EDITADA EN VIVO/' /root/web/index.html
curl -s localhost:8080 | grep -i h1     # -> "Hola desde mi web EDITADA EN VIVO"
```

> **`docker cp` vs volumen:** `cp` es una copia puntual hacia dentro del contenedor; el **volumen** mantiene los datos fuera del contenedor y vivos (los cambios se reflejan al instante, y sobreviven si borras el contenedor).

Limpia:

```bash
docker container rm -f web2
```

---

## 3. Base de datos: variables de entorno y volúmenes (con `docker run`)

Las imágenes se **parametrizan** con variables de entorno (`-e`). La base de datos guarda sus ficheros en un **volumen con nombre** para que **persistan** aunque borremos el contenedor.

### 3.1. Arrancar MariaDB

```bash
docker container run -d --name db \
  -e MARIADB_ROOT_PASSWORD=raiz123 \
  -e MARIADB_DATABASE=tienda \
  -e MARIADB_USER=ivan \
  -e MARIADB_PASSWORD=ivan123 \
  -v datos-db:/var/lib/mysql \
  mirror.gcr.io/library/mariadb:11
```

- `-e VAR=valor` = variable de entorno (configura usuario, contraseña y BBDD).
- `-v datos-db:/var/lib/mysql` = **volumen con nombre** `datos-db` montado donde MariaDB guarda los datos.

Espera unos segundos a que inicialice y comprueba los logs:

```bash
docker container logs db | tail -5
docker volume ls                 # aparece el volumen "datos-db"
```

### 3.2. Crear datos

```bash
docker container exec db mariadb -uivan -pivan123 tienda \
  -e "CREATE TABLE productos(id INT, nombre VARCHAR(50)); INSERT INTO productos VALUES (1,'teclado'),(2,'raton');"

docker container exec db mariadb -uivan -pivan123 tienda -e "SELECT * FROM productos;"
```

Salida esperada:

```
id      nombre
1       teclado
2       raton
```

### 3.3. Comprobar la PERSISTENCIA

Destruimos el contenedor y creamos **uno nuevo** apuntando al **mismo volumen**:

```bash
docker container rm -f db

docker container run -d --name db \
  -e MARIADB_ROOT_PASSWORD=raiz123 \
  -v datos-db:/var/lib/mysql \
  mirror.gcr.io/library/mariadb:11

# (espera unos segundos) y consulta de nuevo:
docker container exec db mariadb -uroot -praiz123 tienda -e "SELECT * FROM productos;"
```

> **Los datos siguen ahí** (`teclado`, `raton`): vivían en el volumen, no en el contenedor. Esa es la razón de ser de los volúmenes.

Limpia todo:

```bash
docker container rm -f db
docker volume rm datos-db
```

---

## 4. La misma MariaDB, pero con Docker Compose

Compose describe lo mismo de forma **declarativa** en un fichero, en lugar de un `docker run` largo. Crea la carpeta y el fichero:

```bash
mkdir -p /root/db && cd /root/db
cat > compose.yaml <<'YAML'
services:
  db:
    image: mirror.gcr.io/library/mariadb:11
    environment:
      MARIADB_ROOT_PASSWORD: raiz123
      MARIADB_DATABASE: tienda
      MARIADB_USER: ivan
      MARIADB_PASSWORD: ivan123
    ports:
      - "3306:3306"
    volumes:
      - datos-db:/var/lib/mysql
volumes:
  datos-db:
YAML
```

Levanta, consulta y derriba la pila:

```bash
docker compose up -d                 # crea red, volumen y contenedor
docker compose ps                    # estado de los servicios
docker compose logs db | tail -5

# (cuando termines)
docker compose down                  # borra contenedor y red (conserva el volumen)
docker compose down -v               # idem, y ADEMÁS borra el volumen
```

> Compara: lo que en la sección 3 era un `docker run` con muchos `-e` y `-v`, aquí es un fichero legible y versionable. Mismo resultado, forma declarativa.

---

## 5. WordPress completo con Docker Compose

Montamos **dos servicios** que se hablan entre sí: **WordPress** (imagen con Apache + PHP integrados) y **MariaDB**. Es el ejemplo clásico de aplicación multicontenedor.

```bash
mkdir -p /root/wordpress && cd /root/wordpress
cat > compose.yaml <<'YAML'
services:
  db:
    image: mirror.gcr.io/library/mariadb:11
    environment:
      MARIADB_ROOT_PASSWORD: raiz123
      MARIADB_DATABASE: wordpress
      MARIADB_USER: wp
      MARIADB_PASSWORD: wp123
    volumes:
      - datos-db:/var/lib/mysql

  wordpress:
    image: mirror.gcr.io/library/wordpress:latest   # Apache + PHP + WordPress
    depends_on:
      - db
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db                # <-- el NOMBRE del servicio db (DNS interno)
      WORDPRESS_DB_USER: wp
      WORDPRESS_DB_PASSWORD: wp123
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - datos-wp:/var/www/html

volumes:
  datos-db:
  datos-wp:
YAML
```

Levanta la pila:

```bash
docker compose up -d
docker compose ps                              # db y wordpress "running"
curl -s -o /dev/null -w "HTTP %{http_code}\n" localhost:8080   # 302 (redirige al instalador)
```

> **Concepto clave:** WordPress se conecta a la base de datos usando `WORDPRESS_DB_HOST: db`, es decir, **el nombre del otro servicio**, no una IP. Docker Compose crea una red interna donde cada servicio es resoluble por su nombre. Es la misma idea que un `Service` en Kubernetes.


```bash
docker compose down -v        # borra contenedores, red y volúmenes
```

---

## Resumen de comandos (chuleta)

| Acción | Canónico | Alias |
|--------|----------|-------|
| Descargar imagen | `docker image pull <img>` | `docker pull <img>` |
| Listar imágenes | `docker image ls` | `docker images` |
| Borrar imagen | `docker image rm <img>` | `docker rmi <img>` |
| Ejecutar contenedor | `docker container run ...` | `docker run ...` |
| Listar contenedores | `docker container ls -a` | `docker ps -a` |
| Logs | `docker container logs <c>` | `docker logs <c>` |
| Entrar (shell) | `docker container exec -it <c> bash` | `docker exec -it <c> bash` |
| Copiar fichero | `docker container cp <orig> <c>:<dest>` | `docker cp ...` |
| Parar / borrar | `docker container stop/rm <c>` | `docker stop/rm <c>` |
| Compose arriba/abajo | `docker compose up -d` / `down -v` | — |

**Opciones de `run` más usadas:** `-d` (segundo plano) · `--name` (nombre) · `-p HOST:CONT` (publicar puerto) · `-e VAR=valor` (variable de entorno) · `-v origen:destino[:ro]` (volumen) · `--rm` (autoborrar al salir).
