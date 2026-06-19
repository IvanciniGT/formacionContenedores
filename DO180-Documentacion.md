# DO180 — Contenedores, Kubernetes y OpenShift

> Documentación del curso. Aunque la certificación de referencia (DO180) es de **OpenShift**, la mayor parte del temario se centra en los **conceptos y objetos básicos de Kubernetes**, que son la base común. La parte específica de OpenShift se trata en un bloque propio al final del documento.

**Definición de trabajo que se usará durante todo el curso:**

> Kubernetes es una herramienta que permite **desplegar y operar entornos de producción basados en contenedores de forma automatizada**, mediante la **definición declarativa** de cómo debe ser dicho entorno.

Cada apartado de este documento sigue la misma pauta: primero el **concepto**, después uno o varios **ejemplos trabajados** o **diagramas**, y al final un bloque de **comandos y recetas operativas** con `kubectl` (y, donde proceda, `oc`).

---

## Índice

1. [Formas de desplegar software](#1-formas-de-desplegar-software)
2. [Contenedores](#2-contenedores)
3. [Imágenes de contenedor](#3-imágenes-de-contenedor)
4. [El sistema de archivos de un contenedor: capas y volúmenes](#4-el-sistema-de-archivos-de-un-contenedor-capas-y-volúmenes)
5. [Gestores de contenedores y redes](#5-gestores-de-contenedores-y-redes)
6. [Introducción a Kubernetes](#6-introducción-a-kubernetes)
7. [Entornos de producción: alta disponibilidad y escalabilidad](#7-entornos-de-producción-alta-disponibilidad-y-escalabilidad)
8. [La analogía del supermercado](#8-la-analogía-del-supermercado)
9. [Objetos de Kubernetes: visión general](#9-objetos-de-kubernetes-visión-general)
10. [Workload: del contenedor al Pod, y del Pod a la plantilla](#10-workload-del-contenedor-al-pod-y-del-pod-a-la-plantilla)
11. [Objetos generadores de Pods: Deployment, StatefulSet, DaemonSet](#11-objetos-generadores-de-pods-deployment-statefulset-daemonset)
12. [Recursos: requests y limits](#12-recursos-requests-y-limits)
13. [Sondas de salud (probes)](#13-sondas-de-salud-probes)
14. [Afinidades y antiafinidades](#14-afinidades-y-antiafinidades)
15. [Tareas: Job y CronJob](#15-tareas-job-y-cronjob)
16. [Escalado automático y fiabilidad (HorizontalPodAutoscaler)](#16-escalado-automático-y-fiabilidad-horizontalpodautoscaler)
17. [Gestión de actualizaciones: rollouts y estrategias](#17-gestión-de-actualizaciones-rollouts-y-estrategias)
18. [Comunicaciones: la red en Kubernetes](#18-comunicaciones-la-red-en-kubernetes)
19. [Service: ClusterIP, NodePort y LoadBalancer](#19-service-clusterip-nodeport-y-loadbalancer)
20. [Ingress e Ingress Controller](#20-ingress-e-ingress-controller)
21. [NetworkPolicy](#21-networkpolicy)
22. [Almacenamiento en Kubernetes](#22-almacenamiento-en-kubernetes)
23. [Configuración: ConfigMap, Secret y variables de entorno](#23-configuración-configmap-secret-y-variables-de-entorno)
24. [Gobierno del clúster: Namespace, LimitRange y ResourceQuota](#24-gobierno-del-clúster-namespace-limitrange-y-resourcequota)
25. [Caso integrador: despliegue completo de WordPress](#25-caso-integrador-despliegue-completo-de-wordpress)
26. [El modelo de API y las CLIs: kubectl / oc](#26-el-modelo-de-api-y-las-clis-kubectl--oc)
27. [Bloque OpenShift](#27-bloque-openshift)
28. [Glosario y tabla de equivalencias](#28-glosario-y-tabla-de-equivalencias)

---

## 1. Formas de desplegar software

Para construir software se emplea un lenguaje de programación (Java, C, C++, Python…) que después se traduce al lenguaje que entiende el sistema operativo, bien por **compilación** (traducción previa a la ejecución), bien por **interpretación** (traducción en tiempo real mediante un intérprete: la JVM para Java, CPython para Python…). El intérprete sí depende del sistema operativo.

A lo largo del tiempo se han usado tres métodos para instalar y ejecutar ese software. Conviene compararlos porque justifican la existencia de los contenedores.

### 1.1. Método clásico: todo sobre el mismo sistema operativo

```
        App 1   +   App 2   +   App 3
   ------------------------------------------
              Sistema Operativo
   ------------------------------------------
                  Hardware
```

Problemas:

- **Compartición de recursos.** Si la App 1 tiene un fallo y consume el 100 % de la CPU, deja sin servicio a la App 2 y a la App 3.
- **Incompatibilidad de configuración** del sistema operativo entre aplicaciones.
- **Incompatibilidad de librerías, *frameworks* y dependencias** (la App 1 necesita la versión X de una librería; la App 2, la versión Y).
- **Seguridad**: todas las aplicaciones comparten el mismo entorno y se ven entre sí.

### 1.2. Método basado en máquinas virtuales

```
        App 1     |   App 2  +  App 3
   ------------------------------------------
         SO 1     |        SO 2
   ------------------------------------------
         MV 1     |        MV 2
   ------------------------------------------
      Hipervisor (VMware, Citrix, KVM, Hyper-V, VirtualBox…)
   ------------------------------------------
              Sistema Operativo
   ------------------------------------------
                  Hardware
```

Resuelve el aislamiento, pero introduce otros inconvenientes:

- **Mayor complejidad** de configuración (cada máquina virtual es un sistema completo que administrar).
- **Desperdicio de recursos**: cada máquina virtual ejecuta **su propio kernel** y su propio sistema operativo completo. Ese kernel consume memoria y CPU que no dedicamos a las aplicaciones.
- **Peor rendimiento** de las aplicaciones, por la capa de virtualización.

### 1.3. Método basado en contenedores

```
        App 1     |   App 2  +  App 3
   ------------------------------------------
         C1       |        C2
   ------------------------------------------
      Gestor de contenedores
      (Docker, containerd, CRI-O, Podman)
   ------------------------------------------
      Sistema Operativo (normalmente GNU/Linux)
   ------------------------------------------
           Hardware físico o virtual
```

Se obtiene el aislamiento de las máquinas virtuales **sin** el coste de ejecutar un kernel por aplicación: todos los contenedores comparten el **único** kernel del host. Esa es la diferencia esencial.

> **Regla mnemotécnica.** La frontera de «lo contenedorizable» pasa justo por debajo de las aplicaciones: sistemas operativos, aplicaciones, demonios y servicios se pueden contenedorizar; las librerías y *drivers* no (no son ejecutables por sí mismos); los scripts y comandos se pueden contenedorizar pero **terminan**, por lo que en Kubernetes requieren tratamiento especial (Jobs).

---

## 2. Contenedores

### 2.1. Qué es un contenedor

> Un **contenedor** es un **entorno aislado** dentro de un sistema operativo con kernel Linux en el que se ejecutan procesos.

El aislamiento se refiere a cuatro aspectos:

- **Red propia**: el contenedor tiene su(s) propia(s) dirección(es) IP.
- **Sistema de archivos propio.**
- **Variables de entorno propias.**
- **Limitación de acceso a los recursos físicos del host** (CPU, memoria, GPU…), de forma opcional.

Además, en su configuración se define el **proceso principal** que se ejecutará al arrancar.

**Los contenedores no ejecutan un kernel propio.** Por eso solo pueden ejecutarse sobre un kernel Linux:

- En **Linux**, de forma nativa.
- En **macOS**, se levanta una máquina virtual con Linux que aloja los contenedores.
- En **Windows**, mediante WSL2 (antiguamente, Hyper-V).

Dado que Linux es el kernel más utilizado del mundo (servidores empresariales, Android…), esta limitación es poco relevante en la práctica.

> **Nota terminológica.** «Linux» no es un sistema operativo, sino un **kernel**. En las empresas se utilizan sistemas operativos **GNU/Linux**, donde el kernel Linux (que cumple los estándares UNIX®: POSIX + SUS) se acompaña del conjunto de programas GNU (`ls`, `ps`, `mkdir`, `bash`…) y de las herramientas que añade cada distribución: RHEL (`yum`/`dnf`), Debian/Ubuntu (`apt`), SUSE… GNU es el acrónimo recursivo de *GNU is Not Unix*.

### 2.2. Aislamiento, en cuanto a qué

El aislamiento de red implica que **los puertos que abran los procesos del contenedor se abren sobre la IP del contenedor**, no sobre la del host. El aislamiento de sistema de archivos implica que el contenedor «ve» su propia raíz `/` (ver apartado 4). El aislamiento de variables de entorno permite parametrizar cada contenedor por separado.

### 2.3. Ejecutar procesos adicionales dentro de un contenedor

Un contenedor, al iniciarse, arranca **un único proceso principal**, el configurado en su imagen. Ese proceso puede a su vez lanzar procesos hijos. También es posible pedir al sistema operativo que ejecute procesos adicionales dentro de un contenedor en marcha:

```bash
docker exec CONTENEDOR COMANDO
```

> Cuando se ejecuta un comando sin ruta absoluta, el sistema lo busca en los directorios definidos en la variable `PATH`.

### 2.4. Tipos de software (y cuáles encajan en Kubernetes)

| Tipo | Descripción | ¿Contenedorizable? | ¿Encaja en un Pod de Kubernetes? |
|------|-------------|:------------------:|:--------------------------------:|
| **Sistema operativo** | — | (caso especial) | — |
| **Aplicación** | Programa con interfaz gráfica para interactuar con personas (Word, Chrome…) | Sí | — |
| **Demonio** | Programa en segundo plano, en ejecución indefinida | Sí | **Sí** |
| **Servicio** | Demonio que atiende a otros programas, normalmente por un puerto de red (servidor web, FTP…) | Sí | **Sí** |
| **Librería** | Funciones reutilizables por otros programas | No (no es ejecutable) | No |
| **Driver** | Controla un dispositivo físico | No (no es ejecutable) | No |
| **Script** | Ejecuta una serie de tareas y **termina** | Sí | No directamente (ver Jobs) |
| **Comando** | Ejecuta una tarea concreta y **termina** | Sí | No directamente (ver Jobs) |

> **Idea clave para Kubernetes.** Los contenedores de un Pod deben ejecutarse **de forma continua, 24×7**. Si un contenedor termina, Kubernetes lo interpreta como un fallo y lo recrea. Por eso los únicos tipos que encajan de forma natural son **demonios y servicios**. Los scripts y comandos (que terminan) se gestionan con **Jobs** y **CronJobs**, o dentro de un Pod en el bloque `initContainers`.

---

## 3. Imágenes de contenedor

### 3.1. Qué es una imagen de contenedor

Los contenedores se crean **a partir de imágenes de contenedor**.

> Una **imagen de contenedor** es un fichero comprimido (`tar`) que empaqueta una aplicación **ya instalada y preconfigurada**, lista para ejecutarse.

Dentro de una imagen encontramos:

- Uno o varios **programas ya instalados** (por ejemplo, NGINX), en una estructura de carpetas POSIX.
- Las **dependencias** (librerías, *frameworks*) necesarias para ejecutarlos, **ya instaladas**.
- Otros programas de utilidad que pueden venir incluidos (`ls`, `bash`…).

Y, además, información complementaria sobre el software que contiene:

- **Puertos** que usan los programas instalados.
- **Directorios** que usan para guardar datos o leer configuración.
- **Variables de entorno** que emplean, algunas con valores por defecto.
- El **proceso principal** que se ejecuta al arrancar.
- **Metadatos** (autor, etc.) y documentación.

Si descomprimiéramos el `tar`, encontraríamos una estructura POSIX típica:

```
/bin/    Comandos ejecutables
/etc/    Configuraciones
/opt/    Aplicaciones
/usr/    Más programas
/var/    Datos de las aplicaciones
/home/   Carpetas de usuarios
/root/   Carpeta del usuario root
/tmp/    Temporales (se borran al reiniciar)
```

Por convención: la aplicación suele instalarse en `/usr/` o `/opt/`, su configuración en `/etc/` y sus datos se generan en `/var/`.

> Hoy prácticamente **todo el software empresarial se distribuye como imágenes de contenedor**. Se ha convertido en un estándar de facto, respaldado por la **Cloud Native Computing Foundation (CNCF)**. Esto implica dos cosas: (1) una imagen funciona en **cualquier** ejecutor de contenedores, con independencia de la herramienta con que se haya creado; y (2) la forma de **operar** con el software contenido también está estandarizada.

### 3.2. Comparación con la instalación tradicional

Instalar, por ejemplo, un SQL Server de forma tradicional exige: preparar el sistema operativo con sus dependencias, conseguir un instalador, ejecutarlo y configurarlo. La imagen de contenedor **comprime ese resultado final** (el software ya instalado y configurado) en un único artefacto distribuible:

```
Instalación tradicional        →   Imagen de contenedor
(SO + instalador + pasos +         (resultado ya empaquetado,
 configuración manual)              listo para ejecutar)
```

### 3.3. Registries de imágenes

Las imágenes se obtienen de **registries** (registros de repositorios de imágenes):

- El **registry propio de la empresa**.
- **Docker Hub** (registry oficial de imágenes Docker).
- **Quay.io** (de Red Hat).

### 3.4. Identificación de una imagen: repositorio y tag

```
REGISTRY/REPOSITORIO:TAG

docker.io/nginx:1.21
miempresa.com/mirepo:mitag
```

- **REGISTRY**: a menudo se omite y se toma el configurado por defecto (por ejemplo, `docker.io`).
- **REPOSITORIO**: identifica el software (`nginx`).
- **TAG**: es lo que propiamente identifica una **imagen concreta**.

Internamente, cada imagen tiene un **ID** calculado como una huella (`sha256:…`). Un **tag** es una etiqueta legible que apunta a esa huella. Se usan tags en lugar de IDs porque son más legibles y porque **una misma imagen puede tener varios tags** a la vez.

> Si no se indica tag, el gestor **no descarga la última versión**, sino la etiquetada como **`latest`**. Si ninguna imagen tiene ese tag, la descarga fallará indicando que la imagen no existe.

**Versionado del software** (`MAYOR.MINOR.MICRO`, p. ej. `3.4.5`):

| Componente | Se incrementa cuando… |
|-----------|------------------------|
| **MAYOR** (3) | Hay un rediseño que **puede romper la retrocompatibilidad** |
| **MINOR** (4) | Se **añade** nueva funcionalidad |
| **MICRO** (5) | Se **corrige** un *bug* |

**Qué tag elegir en producción.** Supongamos que el desarrollador publica una imagen y la etiqueta como `1.21.7`, `1.21`, `1` y `latest`, todas apuntando a la misma huella:

| Tag | ¿Recomendable? | Motivo |
|-----|:--------------:|--------|
| `nginx:latest` | **No** | Mañana ese tag puede apuntar a la versión 2.x y todo dejaría de funcionar |
| `nginx:1.21.7` | No | Demasiado restrictivo: no recoge correcciones de *bugs* posteriores (`1.21.8`…) |
| `nginx:1` | No | Puede introducir versiones *minor* con nueva funcionalidad y **nuevos *bugs*** que no necesitamos |
| **`nginx:1.21`** | **Sí** | Fija la funcionalidad que necesito y recoge el **mayor MICRO posible** (todas las correcciones de *bugs*) |

> Usar `latest` es cómodo pero es una **mala práctica** en producción: impide saber con certeza qué versión se está desplegando, algo inadmisible en un entorno productivo.

El tag suele incluir además información sobre la **imagen base** (`ubuntu`, `debian`, `alpine`, `oraclelinux`…) o configuraciones adicionales.

### 3.5. Imágenes base

Las **imágenes base** apenas «hacen» nada por sí mismas: aportan una estructura de carpetas POSIX y un conjunto de comandos. Se usan como punto de partida para construir imágenes propias.

| Imagen base | Gestor de paquetes | Características |
|-------------|--------------------|----------------|
| **Ubuntu** / Debian | `apt` | Completa, muy usada |
| **Fedora** / CentOS / RHEL | `yum` / `dnf` | Familia Red Hat |
| **Alpine** | `apk` | Muy ligera (pocos MB). **No** incluye `bash` ni comandos extra |

Comparativa de contenidos (orientativa):

```
ubuntu/        alpine/        fedora/
  bin/ ls        bin/ ls        bin/ ls
      mkdir          mkdir          mkdir
      bash           sh             bash
      apt            wget           yum / dnf
  etc/ var/ ...  etc/ var/ ...  etc/ var/ ...
```

> **Construcción de una imagen propia** (aplicación bancaria Java sobre WebLogic): (1) partir de `oraclelinux`; (2) instalar WebLogic encima; (3) configurar en WebLogic el `.war` de la aplicación; (4) exportar la nueva imagen. De esa imagen se crearán los contenedores.

---

## 4. El sistema de archivos de un contenedor: capas y volúmenes

### 4.1. Dónde reside y el truco de `chroot`

Cada contenedor tiene su propio sistema de archivos, **que reside dentro del sistema de archivos del host**. El sistema operativo «engaña» a los procesos del contenedor: cuando preguntan por la raíz `/`, no se les lleva a la raíz del host, sino a la raíz del sistema de archivos del contenedor.

Este mecanismo existe en UNIX desde hace décadas. El comando que cambia la raíz aparente de un proceso es **`chroot`** (*change root*):

```
chmod   → cambia los permisos de un fichero
chown   → cambia el propietario
chroot  → cambia la raíz (/) del sistema de archivos para un proceso
```

```
   FS del HOST                                  El proceso del contenedor "minginx"
   /                          (1)  raíz real    cree que su raíz "/" es (2),
     bin/  ls  mkdir  bash                       no (1). Es un engaño de chroot.
     usr/  bin/docker
     var/
       lib/
         docker/
           containers/
             minginx/         (3)  CAPA DEL CONTENEDOR
               var/log/nginx/nginx.log   ← cambios y datos propios del contenedor
           images/
             nginx/           (2)  CAPA BASE (imagen), de SOLO LECTURA
               bin/  ls  mkdir  bash
               etc/nginx/nginx.conf      ← configuración de la imagen
               opt/nginx/                ← binarios de nginx
               var/nginx/web/            ← web por defecto
```

### 4.2. El sistema de archivos por capas

El sistema de archivos de un contenedor es el resultado de **superponer varias capas**:

```
   ┌──────────────────────────────────────────────────────────────┐
   │  VOLÚMENES (capa n)   Puntos de montaje adicionales hacia      │
   │                       almacenamiento interno o externo.        │
   │                       OPACAN las rutas equivalentes de las     │
   │                       capas inferiores.                        │
   ├──────────────────────────────────────────────────────────────┤
   │  CAPA DEL CONTENEDOR  Recoge TODOS los cambios sobre el FS.    │
   │                       Si se borra el contenedor, se elimina.   │
   ├──────────────────────────────────────────────────────────────┤
   │  CAPA BASE (imagen)   La capa de la imagen. INALTERABLE:       │
   │                       nadie tiene permiso de escritura.        │
   └──────────────────────────────────────────────────────────────┘
```

Cuando un proceso escribe o modifica un archivo, el cambio se registra **en la capa del contenedor**, nunca en la de la imagen (de solo lectura). Por eso pasar de `nginx:1.21.1` a `nginx:1.21.2` consiste, simplemente, en **borrar el contenedor antiguo y crear uno nuevo** desde la imagen nueva.

### 4.3. Qué es un volumen y para qué sirve

> Un **volumen** es un **punto de montaje** en una carpeta o archivo del sistema de archivos del contenedor que, en realidad, guarda los datos **fuera** de ese sistema de archivos.

Soporte físico posible de un volumen: un disco (HDD/SSD) del host, un almacenamiento en red (NFS, Samba, iSCSI), una LUN de una cabina, un trozo de RAM usado como disco, o un *cloud*.

**Usos de los volúmenes:**

1. **Persistencia de datos** tras la eliminación de un contenedor. Borrar contenedores es **habitual**: al escalar (de 3 a 10 réplicas y de vuelta), al actualizar el software, o cuando Kubernetes decide mover un contenedor de un host a otro (lo borra en uno y lo crea en otro). Sin volúmenes, los datos se perderían en cada operación.
2. **Compartir archivos entre contenedores**: dos contenedores con el mismo volumen montado comparten esos datos.
3. **Inyectar archivos o carpetas** dentro de un contenedor: sobrescribir una configuración por defecto, inyectar los ficheros de nuestra aplicación, etc.

---

## 5. Gestores de contenedores y redes

### 5.1. Gestores de contenedores

| Gestor | Notas |
|--------|-------|
| **Docker** | El más popular históricamente. Su licencia es de pago para empresas grandes (>250 empleados); sigue siendo gratuito para empresas pequeñas, estudiantes y proyectos *open source* |
| **Podman** | *Open source* y gratuito. Viene de serie en Red Hat y derivados. Es un gestor de **pods** |
| **CRI-O** | *Open source*. Soportado por Kubernetes |
| **containerd** | *Open source*. Soportado por Kubernetes |

La arquitectura interna de Docker ilustra los niveles de gestión:

```
docker (cliente)  →  dockerd  →  containerd  →  runC
                      │            │             │
                  construir     ciclo de vida  ejecución de
                  imágenes,     de contenedores un contenedor
                  login al      e imágenes      concreto
                  registry      (alternativa: CRI-O)
```

`docker run` es, en realidad, la combinación de varios pasos:

```
docker run  =  docker image pull          (descargar la imagen)
            +  docker container create     (crear el contenedor)
            +  docker container start       (arrancarlo)
            +  docker container attach      (engancharse a su salida)
```

Estructura general del cliente: `docker [TIPO_OBJETO] [VERBO] <args>`

```
docker image  pull / push / list / rm / inspect
docker container  create / start / stop / restart / logs / attach / exec / rm / inspect

# Atajos frecuentes
docker images   ==  docker image list
docker ps       ==  docker container list
docker rmi      ==  docker image rm
```

### 5.2. Redes en contenedores

Al instalar Docker, una de las primeras cosas que hace es crear una **red virtual privada dentro del host**. Por defecto opera en el rango `172.17.0.0/16`, y el host toma la IP `172.17.0.1`. Los contenedores se «pinchan» en esa red y reciben IPs secuenciales (`172.17.0.2`, `172.17.0.3`…).

```
   ─────────────────── Red de la empresa (p. ej. AWS) ───────────────────
        │                                              │
   172.31.16.180  (IvanPC)                        172.31.16.181 (MenchúPC)
        │
        ├─ 127.0.0.1     loopback (procesos internos)
        │
        └─ 172.17.0.1    Docker ── 172.17.0.2 ── Contenedor nginx (abre el puerto 80)
```

> **Interfaz de red**: abstracción del sistema operativo sobre la tarjeta física (NIC) que da acceso a una red. Toda computadora tiene al menos la interfaz de **loopback** (`127.0.0.1` = `localhost`), para comunicar procesos dentro de la propia máquina, además de su interfaz física.

---

## 6. Introducción a Kubernetes

### 6.1. Qué es

> **Kubernetes es un orquestador de gestores de contenedores** (Docker, Podman, CRI-O, containerd).

Es la herramienta que hoy opera los **entornos de producción** de las empresas. Su cometido: **administrar cargas de trabajo y servicios** (escalabilidad y alta disponibilidad) y **gestionar, implementar y escalar aplicaciones contenedorizadas**.

Nuestra misión no es operar el entorno de producción a mano, sino **indicarle a Kubernetes, de forma declarativa, cómo queremos que lo opere**. Ya no instalamos nada nosotros: le pedimos a Kubernetes que despliegue las cosas con alta disponibilidad y escalabilidad.

### 6.2. Origen y distribuciones

Kubernetes es un proyecto *open source* iniciado por Google. Hoy es, en esencia, una **colección de estándares** con muchas implementaciones:

| Distribución | Proveedor / contexto |
|--------------|----------------------|
| **K8s** | La implementación de referencia |
| **K3s** | Ligera, para *edge* / IoT |
| **Minikube** | Clúster «de juguete» para aprendizaje y desarrollo local; trae muchas cosas ya instaladas |
| **OKD / OpenShift** | Red Hat (K8s + funcionalidades adicionales) |
| **Tanzu** | VMware |

> En este curso, los conceptos y objetos son los de Kubernetes; la última parte aborda las particularidades de **OpenShift**, la distribución de Red Hat sobre la que versa la certificación DO180.

### 6.3. Lenguaje declarativo frente a imperativo

Kubernetes se configura con un **lenguaje declarativo**: se describe **el estado deseado**, no los pasos para alcanzarlo.

- **Imperativo**: «Ve al almacén, coge una silla, ponla bajo la ventana». Se detalla el *cómo*.
- **Declarativo**: «Quiero que haya una silla bajo la ventana». Se describe el *qué*; el *cómo* es responsabilidad del sistema.

Este enfoque es común a Kubernetes, OpenShift, Ansible, Puppet, Terraform y Docker Compose. En Kubernetes, esas definiciones se escriben habitualmente en **YAML**.

> **YAML** (*YAML Ain't Markup Language*) es un lenguaje de marcado de información de propósito general, como XML o JSON, pero mucho más legible; hoy ha desplazado a JSON en este tipo de configuraciones. Para escribir definiciones de Kubernetes no basta con conocer YAML: hay que conocer también el **esquema de Kubernetes** (qué campos admite cada tipo de objeto).

### 6.4. Anatomía de la red de un clúster real

Un clúster está formado por varias máquinas. Unas ejecutan las cargas de trabajo (**nodos *worker***) y otras albergan el **plano de control** (los maestros). Todas tienen un gestor de contenedores (CRI-O o containerd):

```
   Cluster de Kubernetes
   ├─ Máquina 1   CRI-O/containerd        (worker)
   ├─ Máquina 2   CRI-O/containerd        (worker)
   ├─ …           CRI-O/containerd
   ├─ Máquina n   CRI-O/containerd        (worker)
   ├─ Maestra 1   CRI-O/containerd + Kubernetes
   ├─ Maestra 2   CRI-O/containerd + Kubernetes
   └─ Maestra 3   CRI-O/containerd + Kubernetes
```

Software de Kubernetes y dónde vive cada pieza:

| Componente | Función | Dónde |
|------------|---------|-------|
| `kubeadm` | Crear/gestionar el clúster, añadir nodos | Sobre el hierro, en todos los nodos |
| `kubelet` | Demonio que llama al gestor de contenedores (análogo a `dockerd`) | Sobre el hierro, en todos los nodos |
| `kubectl` | Cliente de línea de comandos | Donde queramos conectarnos al clúster |
| `kube-apiserver` | Recibe todas las órdenes de los clientes | Plano de control (en contenedores) |
| `etcd` | Base de datos clave-valor con todo el estado del clúster | Plano de control (≥3 instancias) |
| `kube-scheduler` | Decide en qué nodo se despliega cada Pod | Plano de control |
| `kube-proxy` | Mantiene las reglas de red (Netfilter) en cada nodo | DaemonSet (una réplica por nodo) |
| `kube-controller-manager` | Resto de la lógica de control | Plano de control |
| `CoreDNS` | Servidor DNS interno del clúster | Plano de control |

Flujo de una orden:

```
CLIENTE        MAESTRO(s)                   NODO (worker)
kubectl  ───►  kube-apiserver  ───►  kubelet  ───►  containerd  ───►  runC
```

---

## 7. Entornos de producción: alta disponibilidad y escalabilidad

Las empresas suelen tener varios entornos: **desarrollo**, **pruebas/preproducción** y **producción**. Un entorno de producción se caracteriza por dos exigencias: **alta disponibilidad** y **escalabilidad**.

### 7.1. Alta disponibilidad (High Availability, HA)

**A nivel del servicio** consiste en garantizar que esté disponible un porcentaje de tiempo acordado. Las diferencias entre porcentajes son enormes (el tiempo «caído» incluye actualizaciones, mantenimientos y *backups*):

| Disponibilidad | Tiempo caído al año | Tipo de servicio (orientativo) | Coste |
|:--------------:|:-------------------:|--------------------------------|:-----:|
| 90 % | ~36 días | Inaceptable para casi todo | € |
| 99 % | ~3,6 días | Peluquería de barrio | €€ |
| 99,9 % | ~8 horas | Tienda *online* | €€€€€ |
| 99,99 % | ~20 minutos | Hospital, servicio crítico | €€€€€€€€€€ |

La forma de aproximarse a estos objetivos es la **redundancia**, mediante **clústeres** (del inglés *cluster*, «grupo»):

- **Activo/pasivo**: un nodo da servicio y otro espera en *standby*; si el activo falla, el pasivo toma el relevo.

  ```
  clientes ──► BALANCEO ──┬─► Máquina ACTIVA (IP1)   ← da servicio
                          └─► Máquina PASIVA (IP2)   ← en espera
  ```

- **Activo/activo**: varios nodos ofrecen el servicio simultáneamente.

  ```
  clientes ──► BALANCEO ──┬─► Máquina ACTIVA (IP1)
                          ├─► Máquina ACTIVA (IP2)
                          └─► Máquina ACTIVA (IP3)
  ```

El balanceo de carga se consigue con un balanceador (NGINX, HAProxy, Apache `httpd`, casi cualquier proxy reverso) o con IPs virtuales (VIPA).

> **Ventaja de Kubernetes en el modelo activo/pasivo.** Tradicionalmente, mantener una máquina pasiva «encendida» consumía recursos. Como Kubernetes crea una réplica desde una plantilla de Pod **en segundos**, puede permitirse **no** tener la réplica pasiva levantada y crearla solo cuando hace falta, ahorrando recursos.

**A nivel de los datos**, la alta disponibilidad consiste en garantizar que no se pierde información. De nuevo, **redundancia**: cada dato se guarda en **al menos 3 ubicaciones** (estándar en producción, equivalente a un RAID 5).

> La infraestructura y el software son, por sí mismos, un **coste**. Lo único que aporta valor son **los datos**. Por eso su protección es prioritaria.

### 7.2. Escalabilidad

Consiste en que la infraestructura **se adapte a las necesidades de cada momento**. La demanda hoy es muy variable (una web puede pasar de 0 a un millón de peticiones en horas y volver a 0). Frente al antiguo dimensionamiento «al pico», ahora se busca adaptarse:

- Con muchos usuarios, mucha infraestructura.
- Con pocos usuarios, poca infraestructura.
- Sin usuarios, nada de infraestructura.

Esto lo facilitan los **clouds** (infraestructura como servicio, con pago por uso). Pero no es solo cuestión de infraestructura, sino también de los **programas en ejecución** sobre ella, y de operarlos: reiniciar caídos, instalar nuevos, añadirlos/sacarlos del balanceador, reinstalar máquinas… Antes lo hacían los administradores de sistemas a mano; **hoy lo hace Kubernetes**, de forma más barata, más rápida y sin los errores del trabajo manual.

### 7.3. Por qué la redundancia de datos no es gratis: las cuentas del clúster de BBDD

Conviene entender que replicar datos **mejora la disponibilidad pero penaliza el rendimiento**. Veámoslo con números.

**Clúster de 3 máquinas, cada dato replicado ×2:**

```
   MÁQUINA 1  mariaDB1   datos:  2  3
   MÁQUINA 2  mariaDB2   datos:  1  2
   MÁQUINA 3  mariaDB3   datos:  1  3
```

- **¿Hay HA?** Sí: aguanta la caída de **1** máquina (cada dato está en otra). Si caen 2, se pierde acceso a algún dato.
- **¿Mejora el rendimiento?** Una máquina sola grababa 2 datos en 2 instantes; tres máquinas graban 3 datos en 2 instantes. La mejora es solo del **50 %**, no del 200 %, porque cada dato se escribe dos veces.

  ```
  1 máquina   → 100 peticiones/s
  3 máquinas  → 300 peticiones/s en bruto, pero por la replicación ×2
                queda en ~150 peticiones/s efectivas
  ```

**Clúster de 5 máquinas, cada dato replicado ×3:**

```
   MÁQUINA 1  mariaDB1   1 2 4
   MÁQUINA 2  mariaDB2   1 3 4
   MÁQUINA 3  mariaDB3   1 3 5
   MÁQUINA 4  mariaDB4   2 3 5
   MÁQUINA 5  mariaDB5   2 4 5
```

- **¿Hay HA?** Sí: aguanta la caída de **2** máquinas. Si caen 3, problema.
- **Rendimiento**: 5 máquinas grabando 5 datos en 3 instantes frente a 1 dato; la mejora efectiva es de ~**66 %** (de 100 a ~166 peticiones/s), no del 400 %.

> **Conclusión.** Subir réplicas de datos mejora la HA, pero el rendimiento crece mucho menos de lo que el número de máquinas sugiere, porque cada escritura se multiplica por el factor de replicación.

### 7.4. Brain split

Si las instancias de un clúster de datos pierden la comunicación entre sí, cada grupo podría seguir trabajando como si fuera independiente:

```
   1 3                 2 4
   ▼ ▼                 ▼ ▼
  BBDD1     ╱╱╱       BBDD2
  1 2 3    (corte)    1 2 4     ← dos bases de datos divergentes, imposibles de reunificar
```

Para evitarlo, estos sistemas eligen un **nodo maestro por votación**, que **solo ejerce si obtiene mayoría absoluta**. Por eso se despliegan en **número impar** de instancias (3, 5, 7…): si la red se parte, solo el lado con mayoría sigue operando; el lado minoritario se detiene.

> Todos los sistemas de almacenamiento en clúster funcionan así: bases de datos (MariaDB, MySQL, PostgreSQL, Oracle), indexadores (Elasticsearch) y sistemas de mensajería (Kafka).

### 7.5. Metodologías ágiles y DevOps

Las **metodologías ágiles** sustituyen al desarrollo en cascada. Se basan en **entregas incrementales**: en lugar de una única entrega final, se entrega una versión 100 % funcional (aunque cubra poca funcionalidad) cada pocas semanas. Esto resuelve el problema de los requisitos cambiantes (ahora nos enteramos antes de los cambios), pero introduce otro: **hay que instalar en producción y probar mucho más a menudo**. Donde antes había una instalación al final del proyecto, ahora hay una cada 2-3 semanas.

La respuesta es **DevOps**, una cultura de **automatización**:

- **Integración continua**: tener siempre, de forma automática, lo último desarrollado integrado en el entorno de integración y sometido a pruebas automatizadas.
- **Entrega continua**: poner automáticamente en manos del cliente la última versión probada.
- **Despliegue continuo**: instalar automáticamente en producción la última versión.

Todo se desencadena desde una única acción: el desarrollador hace un *commit* de su código en el sistema de control de versiones (Git).

> **El entregable de un desarrollo ha cambiado.** Antes: programa + documentación + procedimiento de instalación/operación. Hoy: **imágenes de contenedor** (el programa ya instalado) **+ ficheros de despliegue en Kubernetes** (la descripción declarativa de cómo operarlo en producción). **De esto trata este curso.** El curso que le sigue, de programación en **Helm**, trata las *plantillas* de esos ficheros de despliegue (los *charts*).

---

## 8. La analogía del supermercado

Un entorno de producción operado por Kubernetes se entiende bien comparándolo con un gran supermercado (un Carrefour o Alcampo). El **gerente** del supermercado se comporta como **Kubernetes**: si un carnicero se pone enfermo, llama a otra persona con el mismo perfil; eso es alta disponibilidad.

| Supermercado | Entorno de producción / Kubernetes |
|--------------|-------------------------------------|
| **Gerente** del supermercado | **Kubernetes** |
| Si un carnicero enferma, llama a otro con el mismo perfil | Si un Pod cae, crea otro desde la **imagen / plantilla de Pod** (cluster activo/pasivo) |
| **Perfil** del empleado (qué sabe hacer) | **Imagen de contenedor** |
| **Carteles** que indican dónde está cada sección | **DNS** del clúster |
| **Sección** de carnicería (un mostrador) | **Service** |
| Número/pantalla que indica «a quién le toca» | **Balanceo de carga** del Service |
| **Puesto de trabajo** (tabla, cuchillos, báscula) | **Máquina** (CPU, RAM, GPU) |
| **Carnicero** trabajando en su puesto | **Proceso → contenedor → Pod** |
| **Neveras/expositores** (frías para datos, rápidas para venta) | **Almacenamiento** (HDD lento para datos, SSD rápido para servicio) |
| **Sección de cajas** con cola única y pantalla «Caja 14» | Otro **Service** con su balanceo |
| **Puerta de clientes / de mercancías / de empleados**, con seguridad | **Ingress Controller / Router** (proxy reverso) |
| Las **reglas** de qué puerta lleva a qué sección | **Ingress / Route** |
| Los **productos** (la carne, el pescado) | Los **datos** (lo único que aporta valor) |
| La tienda de ultramarinos de la esquina | Un único host con **Docker** (sin orquestación) |

---

## 9. Objetos de Kubernetes: visión general

Kubernetes se organiza en **objetos** (también llamados recursos o configuraciones). Esta es la familia que cubre el curso, agrupada por área:

**Estructura y nodos** — `Namespace`, `Node`

**Workload** — `Pod`, `Deployment`, `StatefulSet`, `DaemonSet`, `Job`, `CronJob`, `HorizontalPodAutoscaler`

**Comunicaciones** — `Service` (ClusterIP / NodePort / LoadBalancer), `Ingress`, `NetworkPolicy`

**Almacenamiento** — `PersistentVolume` (PV), `PersistentVolumeClaim` (PVC), `StorageClass`

**Configuración** — `ConfigMap`, `Secret`

**Gobierno** — `LimitRange`, `ResourceQuota` (y, para identidad, `ServiceAccount` / `Role` / `RoleBinding`)

Todo objeto comparte la misma estructura de cabecera en YAML:

```yaml
kind:        Pod          # Tipo de objeto
apiVersion:  v1           # Librería de Kubernetes que define ese tipo
metadata:
  name: nombre-del-objeto # Identificador (único dentro del Namespace)
  labels:                 # Etiquetas arbitrarias (clave de los vínculos en K8s)
    app: ejemplo
spec:                     # Especificación: el estado deseado del objeto
  ...
```

> Las **labels** (etiquetas) son el mecanismo de vinculación más importante de Kubernetes: un Service encuentra «sus» Pods por *label*, un Deployment sabe qué Pods son suyos por *label*, las afinidades se expresan con *labels*… Casi todo se conecta mediante etiquetas, no mediante nombres ni IPs.

---

## 10. Workload: del contenedor al Pod, y del Pod a la plantilla

### 10.1. En Kubernetes no operamos con contenedores, sino con Pods

> Un **Pod** es un conjunto de contenedores (puede ser uno solo) que:
> - **comparten configuración de red** (y por tanto IP): sus contenedores se comunican entre sí mediante `localhost`;
> - **escalan juntos**;
> - **se despliegan en el mismo host**, lo que les permite **compartir volúmenes de almacenamiento local no persistente**.

```
   POD A  ← tiene UNA IP, compartida por todos sus contenedores
   ├── contenedor 1: nginx
   │        │  localhost:3306
   │        ▼
   └── contenedor 2: mariadb (puerto 3306)
```

### 10.2. ¿Mismo contenedor, mismo Pod o Pods distintos?

La decisión se toma respondiendo a tres preguntas:

> ¿Deben **escalar juntos**? ¿Deben estar en el **mismo host**? ¿Necesitan **compartir volúmenes locales** o hablarse por `localhost`?

**Ejemplo 1 — WordPress (servidor web + base de datos).**

| Pregunta | Respuesta |
|----------|-----------|
| ¿En el mismo contenedor? | **No**: se quieren escalar por separado (p. ej. 5 web y 2 BBDD) y actualizar por separado |
| ¿Mismo Pod o Pods distintos? | **Pods distintos**: no deben escalar juntos, no necesitan el mismo host ni compartir volúmenes locales |

```
   POD A: nginx          POD B: mariadb
```

**Ejemplo 2 — Servidor web + recolector de logs (filebeat/fluentd).**

| Pregunta | Respuesta |
|----------|-----------|
| ¿En el mismo contenedor? | **No**: el mantenimiento sería más complejo (actualizar el recolector sin tocar el web) |
| ¿Mismo Pod? | **Sí**: deben escalar juntos (cada web lleva su recolector) y estar en el mismo host para **compartir el volumen local** donde el web escribe el `access.log` y el recolector lo lee |

```
   POD A                      POD B
   ├── C1: nginx              └── C3: mariadb
   └── C2: filebeat  ← contenedor SIDECAR (acompaña al principal)
```

### 10.3. Caso trabajado: la tubería de logs (sidecar + ELK)

Los ficheros de log de un servidor web **no** deben persistirse en el host (si el host cae, se pierden; si crecen sin control, llenan la máquina). El patrón es:

```
   nginx-1  ──► access.log (volumen en RAM) ◄── filebeat (sidecar) ─┐
   nginx-2  ──► access.log (volumen en RAM) ◄── filebeat (sidecar) ─┤
   …                                                                 ├─► Kafka ─► Logstash ─► Elasticsearch ◄─ Kibana
   nginx-50 ──► access.log (volumen en RAM) ◄── filebeat (sidecar) ─┘     (mensajería   (persistencia y
                                                                           asíncrona)     análisis)
```

Decisiones de diseño y su porqué:

1. **El `access.log` va en un volumen `emptyDir` en memoria RAM**, no en disco ni en red. Si fuera en red, con 50 servidores escribiendo se saturaría la red; en RAM es lo más rápido. Se configura NGINX con **dos ficheros rotados** de tamaño limitado (p. ej. 50 KB cada uno) para no consumir más de 100 KB.
2. **El web (escritor) y filebeat (lector) comparten ese volumen** → deben estar en el **mismo Pod** (mismo host, vol. local no persistente).
3. **filebeat envía a Kafka de forma asíncrona** (el receptor no tiene por qué estar presente al momento). Desde Kafka, Logstash lleva los datos a **Elasticsearch**, donde **sí** se persisten y se analizan (con Kibana).

> Para el servidor web, los logs son datos **desechables**; para Elasticsearch, esos mismos datos son **persistentes**, porque a Elasticsearch sí se le pedirán cuentas de ellos en el futuro (cuadros de mando: accesos por hora, por país, por navegador, por página…).

### 10.4. Por qué no creamos Pods directamente

En un clúster de producción **no creamos Pods** (aunque técnicamente se pueda). Si un Pod o un contenedor muere, nos quedamos sin servicio, lo cual es inaceptable.

> En producción **no trabajamos con Pods, sino con plantillas de Pod**.

A Kubernetes le proporcionamos: una **plantilla de Pod**, **cuántas réplicas** deben estar siempre operativas y, opcionalmente, **entre cuántas y cuántas** según la carga. Es Kubernetes quien **crea, vigila y recrea los Pods**.

### 10.5. Qué define una plantilla de Pod

De cada contenedor: la **imagen**, los **puertos**, los **volúmenes** y su **ruta de montaje**, las **variables de entorno**, los **recursos** y las **sondas**. A nivel de Pod: los **volúmenes** que sus contenedores comparten.

### 10.6. Recetas con Pods

```bash
# Crear y consultar
kubectl apply -f pod.yaml
kubectl get pods -o wide                 # IP y nodo de cada Pod
kubectl describe pod nginx-pod           # eventos, estado, montajes
kubectl logs nginx-pod -c nginx          # logs (salida estándar del proceso)
kubectl exec -it nginx-pod -c nginx -- bash   # entrar en el contenedor
kubectl delete pod nginx-pod
```

Salida típica de `kubectl get pods -o wide`:

```
NAME        READY   STATUS    RESTARTS   AGE   IP           NODE
nginx-pod   1/1     Running   0          5s    10.10.0.7    worker-2
```

---

## 11. Objetos generadores de Pods: Deployment, StatefulSet, DaemonSet

| Objeto | Contenido | Comportamiento |
|--------|-----------|----------------|
| **Deployment** | 1 plantilla de Pod + nº de réplicas | Garantiza ese número de Pods. **Todas las réplicas comparten** (si lo hay) el mismo volumen |
| **StatefulSet** | 1 plantilla de Pod + plantilla de PVC (`volumeClaimTemplates`) + nº de réplicas | Igual, pero **cada réplica recibe su propio volumen** y un nombre estable (`nombre-0`, `nombre-1`…) |
| **DaemonSet** | 1 plantilla de Pod | Despliega **una réplica en cada nodo** del clúster |

**Cuándo usar cada uno:**

- **Deployment**: cargas *stateless* o en las que todas las réplicas comparten volumen. Caso típico: **servidores web**.
- **StatefulSet**: cada réplica necesita **su propio almacenamiento** e identidad estable. Caso típico: **bases de datos en clúster** (cada instancia gestiona sus propios ficheros — ver §7.3).
- **DaemonSet**: componentes que deben estar en todos los nodos (recolectores de logs, agentes de monitorización, `kube-proxy`). Lo usan sobre todo los administradores.

> **Nombres estables del StatefulSet.** Con `serviceName: servicio-mariadb` y `replicas: 2`, los Pods se llaman `statefulset-mariadb-0` y `statefulset-mariadb-1`, y son direccionables individualmente como `statefulset-mariadb-0.servicio-mariadb`. Esto es imprescindible para que las instancias de una base de datos se reconozcan entre sí.

El vínculo entre el generador y sus Pods se hace **por labels**:

```yaml
spec:
  selector:
    matchLabels:
      app: mariadb        # ◄── el Deployment gestiona los Pods con esta label
  template:
    metadata:
      labels:
        app: mariadb      # ◄── la plantilla genera Pods con esta misma label
```

**Recetas:**

```bash
kubectl apply -f deployment.yaml
kubectl get deployment
kubectl get pods -l app=mariadb          # filtrar por label
kubectl scale deployment mariadb-deployment --replicas=5   # escalado manual
kubectl rollout status deployment/mariadb-deployment
kubectl rollout undo deployment/mariadb-deployment         # volver atrás
```

---

## 12. Recursos: requests y limits

A cada contenedor se le asocian cuatro valores:

```yaml
resources:
  requests:        # Mínimo que Kubernetes debe GARANTIZAR
    cpu: 2
    memory: 4Gi
  limits:          # Máximo permitido si hay disponibilidad
    cpu: ...       # (CPU: puede dejarse alto o sin fijar)
    memory: 4Gi    # (Memoria: el MISMO valor que en requests)
```

- **`requests`** (mínimos garantizados): los usa el **scheduler** para decidir en qué nodo cabe el Pod. No existe un «mínimo objetivo» universal; son valores **estimados** (y luego afinados con monitorización) para atender a un número determinado de usuarios.
- **`limits`** (máximos):
  - **CPU** (recurso *comprimible*): con poca CPU la aplicación va más lenta, pero funciona. El límite de CPU suele dejarse alto o sin fijar.
  - **Memoria** (recurso *no comprimible*): si falta, la aplicación falla con *Out Of Memory*. **Regla práctica: `limits.memory` = `requests.memory`**, ni un bit más (salvo casos muy excepcionales).

> **¿Cuál manda, CPU o RAM?** Manda la **CPU**: a más cores, más usuarios se atienden y más RAM se necesita. Configurar mucha RAM con poca CPU es desperdiciar memoria. El objetivo es un **buen ratio** entre ambos, que se determina con **monitorización**.

**Unidades:**

- **CPU**: en cores (`1`, `2`) o en **milicores** (`500m` = medio core).
- **Memoria**: potencias de 1024 — `Ki`, `Mi`, `Gi` — o de 1000 — `K`, `M`, `G`. `1 GiB = 1024 MiB`; `1 GB = 1000 MB`. Lo que antiguamente se llamaba «GB» hoy es «GiB».

### 12.1. Caso trabajado: capacidad del clúster

Sin `limits` de memoria, un Pod «se flipa» y toma toda la RAM que pueda, comprometiendo a los demás. Veámoslo:

```
                     TOTAL        SOLICITADO (request)   EN USO         LIBRE
                     CPU   RAM    CPU      RAM            CPU   RAM
  Máquina 1          8     32Gi
    Pod Apache-WP-1            ┐  5 cores  16Gi           5     6Gi
    Pod MySQL-1               ┘  2 cores  12Gi           3     12Gi   →  ~0 CPU, ~1Gi RAM libres
  Máquina 2          8     32Gi
    Pod Apache-WP-2              5 cores  16Gi           2     8Gi    →  ~6 CPU, ~24Gi RAM libres
```

Una base de datos **siempre** querrá más RAM (su objetivo es tener toda la base y cada consulta cacheadas en memoria). Si no se le pone `limits.memory`, irá pidiendo más mientras quede memoria libre, hasta ahogar el nodo. De ahí la regla `limits.memory = requests.memory`.

> Los `requests`/`limits` de Kubernetes son **mecanismos de protección** (para que un proceso no se desboque), no mecanismos de tuning fino: el control real de memoria de una base de datos se hace en **su propio fichero de configuración**.

### 12.2. Escalado

Cuando la CPU (o la RAM) de los Pods alcanza un umbral, se **escala** añadiendo réplicas:

```
  Apache-1   4 cores   2 en uso (50%)  ← aparentemente "sobra" capacidad
  Apache-2   4 cores   2 en uso (50%)

  ¿Está bien al 50%? NO necesariamente: si Apache-2 cae, Apache-1 no puede
  absorber su carga (se iría al 100% y caería también). Para un servicio
  crítico (homebanking) se escala mucho antes (p. ej. al 25%), de modo que
  si caen varias réplicas, el resto asuma la carga sin perder servicio.
```

El límite del escalado se dimensiona para el **pico** de usuarios previsto, añadiendo un margen por seguridad (alta disponibilidad).

---

## 13. Sondas de salud (probes)

Que el proceso principal esté en ejecución **no garantiza** que el servicio esté operativo (puede estar bloqueado). Kubernetes distingue tres situaciones y, para cada una, se define una prueba **obligatoriamente** si queremos que actúe bien:

| Estado | Sonda | Si la prueba falla… |
|--------|-------|---------------------|
| **¿Ha arrancado?** | `startupProbe` | Kubernetes espera; si nunca arranca, **mata y recrea** el Pod |
| **¿Sigue vivo?** | `livenessProbe` | Kubernetes **mata y recrea** el contenedor |
| **¿Está listo para tráfico?** | `readinessProbe` | Kubernetes **lo saca del balanceador** (Service) y espera; **no** lo mata |

Ejemplo con una base de datos (MySQL):

| Fase | Comprobación |
|------|--------------|
| Inicializando | El proceso está operativo → se le deja seguir |
| Vivo (*liveness*) | Conectar a la BBDD con el usuario **administrador** |
| Listo (*readiness*) | Conectar con un usuario **normal** y ejecutar consultas |

> Durante un *backup* en frío, la BBDD bloquea el acceso a todos salvo al administrador: está **viva** pero **no lista**. Kubernetes debe respetarla (no matarla), pero sacarla del balanceador hasta que vuelva a estar lista.

Tipos de comprobación: `httpGet` (petición HTTP a una ruta/puerto), `exec` (ejecutar un comando dentro del contenedor) y `tcpSocket` (abrir un puerto). Parámetros: `initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, `successThreshold`, `failureThreshold`.

```yaml
livenessProbe:
  httpGet: { path: /, port: 80 }
  initialDelaySeconds: 3
  periodSeconds: 3
  failureThreshold: 3
readinessProbe:
  exec:
    command: ["curl", "localhost:80"]
  initialDelaySeconds: 50
  periodSeconds: 5
```

---

## 14. Afinidades y antiafinidades

Por defecto, el **scheduler** decide el nodo según los recursos comprometidos. Las reglas de afinidad permiten influir en esa decisión.

**A nivel de nodo:**

- `nodeName`: fija el Pod a un nodo concreto (se pierde la HA; uso muy desaconsejado).
- `nodeSelector`: el Pod solo va a nodos con una **label** concreta. Útil para nodos con características especiales (arquitectura de hardware, GPU para *machine learning*, ubicación geográfica).
- `affinity.nodeAffinity`: versión más expresiva, con operadores (`In`, `NotIn`, `Exists`, `Gt`, `Lt`…) y modos `requiredDuringScheduling…` (obligatorio) o `preferredDuringScheduling…` (preferente).

**A nivel de Pod:**

- `podAffinity`: desplegar este Pod **cerca** de otros con ciertas labels.
- `podAntiAffinity`: desplegar este Pod **lejos** de otros con ciertas labels. **Es la regla más usada.**

### 14.1. Caso trabajado: antiafinidad consigo mismo por zona

Para una aplicación con varias réplicas (p. ej. WordPress) interesa **no** colocarlas todas en el mismo nodo ni en la misma zona geográfica, para mejorar la HA. Con `replicas: 3` y antiafinidad **por zona**:

```
   Madrid       Máquina 2 → WP-1
   Cataluña     Máquina 3 → WP-2
   Andalucía    Máquina 1 → WP-3
```

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        podAffinityTerm:
          labelSelector:
            matchLabels: { app: wp }   # "lejos de otros Pods con app=wp"
          topologyKey: zona            # la unidad de reparto es la label "zona"
```

> Se usa **preferente** (`preferred…`) y no **obligatoria** (`required…`): con la obligatoria, si no hay nodos/zonas suficientes, los Pods **no llegarían a desplegarse**. La `topologyKey` define la unidad de reparto: `kubernetes.io/hostname` reparte por nodo; una label propia (`zona`) reparte por zona geográfica.

Etiquetar nodos:

```bash
kubectl label nodes ip-172-31-11-196 zona=EUROPA
kubectl get nodes --show-labels
```

---

## 15. Tareas: Job y CronJob

Para Kubernetes, los contenedores de un Pod deben ejecutarse **eternamente**. Si un contenedor termina, lo interpreta como un fallo y lo recrea. Por eso **scripts y comandos** (que terminan) no encajan como contenedores normales.

Para ejecutar tareas que **deben terminar**:

| Objeto | Descripción |
|--------|-------------|
| **Job** | Contenedores que ejecutan una tarea y **terminan** (un *backup*, una migración, una compilación). `restartPolicy: Never` |
| **CronJob** | Plantilla de Job + una programación **cron** (`schedule: "*/5 * * * *"`) |

Igual que con los Pods, en producción **no creamos Jobs sueltos**, sino **plantillas de Job** mediante **CronJob**.

> **`initContainers`.** Dentro de un Pod, además de `containers`, puede haber `initContainers`: contenedores que se ejecutan **secuencialmente y deben terminar antes** de que arranquen los normales. Sirven para «preparar el terreno» (descargar código de Git, compilarlo, esperar a una dependencia). Los `containers` normales arrancan **en paralelo** una vez completados los `initContainers`.

```bash
# CronJob cada 5 minutos
kubectl get cronjob
kubectl get jobs                  # ejecuciones generadas
kubectl logs job/scripts-noches-28....
```

---

## 16. Escalado automático y fiabilidad (HorizontalPodAutoscaler)

Configurar una aplicación para que sea **fiable** en producción descansa en tres pilares que ya hemos visto y uno nuevo:

1. **Réplicas suficientes** (Deployment/StatefulSet) repartidas con **afinidades** (§14) → tolerancia a la caída de Pods y nodos.
2. **Sondas de salud** (§13) → Kubernetes detecta y recrea lo que no está vivo, y saca del balanceador lo que no está listo.
3. **Recursos bien dimensionados** (§12) → el scheduler garantiza los mínimos y evita que un Pod ahogue al nodo.
4. **Escalado automático** según la carga → ni sobran recursos cuando hay pocos usuarios, ni faltan cuando hay muchos.

### 16.1. Escalado manual frente a automático

El escalado **manual** se hace cambiando el número de réplicas:

```bash
kubectl scale deployment kibana --replicas=10
```

Pero el objetivo en producción es **automatizarlo**: que el número de réplicas suba y baje solo, en función de la carga real. De eso se encarga el **HorizontalPodAutoscaler (HPA)**.

> **Horizontal frente a vertical.** Escalado **horizontal** = añadir/quitar **réplicas** (más Pods). Escalado **vertical** = dar **más CPU/RAM** a cada Pod. El HPA hace escalado **horizontal**, que es el modelo natural en Kubernetes (sumar Pods detrás de un Service). El vertical existe (`VerticalPodAutoscaler`) pero es menos habitual.

### 16.2. HorizontalPodAutoscaler

> Un **HorizontalPodAutoscaler** vigila una métrica (típicamente el **% de uso de CPU**) de los Pods de un Deployment/StatefulSet y **ajusta el número de réplicas** entre un mínimo y un máximo para mantener esa métrica en un objetivo.

```yaml
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v1
metadata:
  name: kibana-hpa
spec:
  scaleTargetRef:                 # a quién escala
    apiVersion: apps/v1
    kind: Deployment
    name: kibana
  minReplicas: 1                  # nunca baja de aquí
  maxReplicas: 10                 # nunca sube de aquí
  targetCPUUtilizationPercentage: 50   # objetivo: CPU media al 50%
```

Funcionamiento: si la CPU media de los Pods supera el 50 %, el HPA **añade** réplicas (hasta `maxReplicas`); si baja, las **retira** (hasta `minReplicas`).

> **Dos requisitos imprescindibles.**
> 1. **Métricas disponibles.** El HPA necesita un origen de métricas en el clúster: el **metrics-server** (en OpenShift viene integrado). Sin él, el HPA no tiene datos sobre los que decidir y no funciona.
> 2. **`requests` de CPU definidos.** El porcentaje del HPA se calcula **sobre el `request` de CPU** de cada contenedor. Si no hay `requests.cpu`, el HPA no puede calcular el porcentaje. Por eso §12 (recursos) es prerrequisito de este apartado.

### 16.3. Recetas

```bash
# Crear un HPA de forma imperativa (equivalente al YAML de arriba)
kubectl autoscale deployment kibana --min=1 --max=10 --cpu-percent=50

kubectl get hpa
# NAME         REFERENCE           TARGETS   MINPODS   MAXPODS   REPLICAS
# kibana-hpa   Deployment/kibana   35%/50%   1         10        3

kubectl describe hpa kibana-hpa     # eventos de escalado y motivo
```

> En OpenShift, el mismo objeto se gestiona con `oc autoscale` / `oc get hpa`, y la consola web muestra gráficamente el escalado.

---

## 17. Gestión de actualizaciones: rollouts y estrategias

Actualizar una aplicación significa, casi siempre, **cambiar la imagen del contenedor** (nueva versión del software) o **cambiar su configuración** (plantilla del Pod). Como un Pod es inmutable, «actualizar» consiste en **crear Pods nuevos y eliminar los viejos**. Quién orquesta ese reemplazo, y **cómo**, es lo que veremos aquí.

### 17.1. El ReplicaSet, el objeto intermedio

Un **Deployment** no gestiona los Pods directamente: crea por debajo un **ReplicaSet** (el objeto que realmente garantiza «N réplicas de esta plantilla»). Cuando se actualiza la plantilla del Deployment, este **crea un ReplicaSet nuevo** (con la nueva plantilla) y va **trasvasando** réplicas del viejo al nuevo. Conservar el ReplicaSet antiguo es lo que permite **volver atrás** (rollback).

```
   Deployment  ──┬─► ReplicaSet v1 (imagen 1.21)  → Pods …  (se va vaciando)
                 └─► ReplicaSet v2 (imagen 1.22)  → Pods …  (se va llenando)
```

### 17.2. Estrategias de actualización

| Estrategia | Comportamiento | Cuándo usarla |
|-----------|----------------|---------------|
| **RollingUpdate** (por defecto) | Sustituye los Pods **gradualmente**, unos pocos cada vez, sin cortar el servicio | Caso general. **Sin downtime** |
| **Recreate** | **Elimina todos** los Pods viejos y luego crea los nuevos | Apps que **no toleran** dos versiones conviviendo (p. ej. un cambio de esquema de BBDD incompatible). **Hay downtime** |

En `RollingUpdate` se controla el ritmo con dos parámetros:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # cuántos Pods de menos se toleran durante la actualización
      maxSurge: 1         # cuántos Pods de más se permiten crear temporalmente
```

```
   RollingUpdate (maxSurge=1, maxUnavailable=0), 3 réplicas:

   v1 v1 v1                ← estado inicial
   v1 v1 v1 [v2]           ← crea 1 nuevo (surge)
   v1 v1 [v2]              ← retira 1 viejo cuando el nuevo está LISTO (readinessProbe)
   v1 v1 v2 [v2]
   …                       ← repite hasta
   v2 v2 v2                ← estado final, sin haber bajado nunca de 3 listos
```

> **La `readinessProbe` (§13) es la clave de un rollout sin cortes.** Kubernetes solo mete un Pod nuevo en el balanceador (Service) cuando su sonda de *readiness* dice que está **listo**, y solo entonces retira un Pod viejo. Sin *readiness probe* bien definida, un rolling update puede enviar tráfico a Pods que aún no están operativos.

### 17.3. Disparar y controlar un rollout

```bash
# Cambiar la imagen (dispara un rollout)
kubectl set image deployment/miapp contenedor=miapp:1.22

# (o editar el YAML y volver a aplicarlo)
kubectl apply -f deployment.yaml

# Seguir el progreso
kubectl rollout status deployment/miapp

# Historial de revisiones
kubectl rollout history deployment/miapp
kubectl rollout history deployment/miapp --revision=2

# Volver atrás
kubectl rollout undo deployment/miapp                 # a la revisión anterior
kubectl rollout undo deployment/miapp --to-revision=2 # a una concreta

# Reiniciar todos los Pods (re-rollout sin cambiar nada)
kubectl rollout restart deployment/miapp
```

Cuántas revisiones se conservan para poder volver atrás se controla con `spec.revisionHistoryLimit` (por defecto, 10).

### 17.4. Tags, digests e `imagePullPolicy`

Al actualizar entra en juego **cómo se identifica la imagen**:

- **Tag mutable** (`nginx:1.21`): cómodo, pero el contenido al que apunta **puede cambiar** con el tiempo. Para producción, recuerda elegir el tag adecuado (§3.4).
- **Digest inmutable** (`nginx@sha256:abc123…`): apunta **siempre al mismo contenido exacto**. Es la forma más segura y reproducible de fijar una versión en producción.

`imagePullPolicy` decide cuándo se vuelve a descargar la imagen:

| Valor | Comportamiento |
|-------|----------------|
| `IfNotPresent` | Solo la descarga si no está ya en el nodo (lo habitual con tags versionados) |
| `Always` | La descarga siempre (por defecto cuando el tag es `latest`) |
| `Never` | Nunca la descarga; debe existir ya en el nodo |

### 17.5. En OpenShift: triggers e ImageStreams

En OpenShift, además de los rollouts de Kubernetes, un despliegue puede actualizarse **automáticamente** cuando aparece una nueva versión de la imagen, gracias a los **image triggers** y los **ImageStreams** (§27.4): el despliegue «observa» un ImageStream y, al detectar una imagen nueva, dispara el rollout solo.

```bash
oc rollout status deployment/miapp
oc rollout undo   deployment/miapp
oc set image deployment/miapp contenedor=miapp:1.22
```

---

## 18. Comunicaciones: la red en Kubernetes

### 18.1. El problema: las IPs de los Pods cambian

Un Apache necesita conectarse a su base de datos. ¿Puede usar la **IP del Pod** de la BBDD? Funcionaría, pero **no sirve**, porque:

- No conocemos la IP de antemano (hasta que el Pod no existe, no tiene IP); obligaría a desplegar por partes.
- La IP **cambia**: si el Pod se reinicia, se actualiza (borrar + crear) o Kubernetes lo mueve a otro nodo, recibe una IP nueva.

¿Y registrar la IP del Pod en un DNS? Tampoco: al cambiar la IP habría que actualizar el DNS **y** confiar en que los clientes refresquen su caché de DNS. La única solución sólida es **una IP fija** más un **nombre en el DNS**. Eso es el **Service**.

### 18.2. La red virtual del clúster: recorrido completo

Kubernetes monta una **red virtual** sobre todos los nodos, con rangos separados para Pods y Servicios:

```
   Pods:       10.10.0.0/16
   Servicios:  10.20.0.0/16
```

Cada nodo ejecuta **Netfilter** (componente del kernel de Linux por el que pasan **todos** los paquetes de red; se configura con `iptables`). Recorrido de extremo a extremo de una petición externa a `app1.produccion.es`:

```
                                                app1.produccion.es = 172.30.2.1
                                                        (DNS de la empresa)
   BALANCEADOR EXTERNO  172.30.2.1                          │
     :80 → [172.30.1.1:30100, 172.30.1.2:30100, …]         │   Menchú (172.30.0.178)
   ───────────────────────── Red de la empresa (172.30.0.0/16) ─────────────────
   │  CLÚSTER
   │
   ├─ 172.30.1.1  Máquina 1 ── Netfilter:
   │     10.20.0.2:8080 → [10.10.0.1:80, 10.10.0.2:80]  (apaches, round robin)
   │     10.20.0.1:3306 → [10.10.0.3:3306]              (BBDD)
   │     10.20.0.3:80   → [10.10.0.4:80, 10.10.0.5:80]  (ingress controllers)
   │     172.30.1.1:30100 → 10.20.0.3:80                (NodePort → ClusterIP)
   │     ├─ Pod Apache 10.10.0.1   (config: BBDD = basedatos.produccion.es)
   │     └─ Pod Ingress Controller (nginx) 10.10.0.4   (regla: app1 → apaches)
   │
   ├─ 172.30.1.2  Máquina 2 ── (mismas reglas Netfilter)
   │     ├─ Pod Apache 10.10.0.2
   │     └─ Pod Ingress Controller 10.10.0.5
   │
   ├─ 172.30.1.3  Máquina 3 ── (mismas reglas Netfilter)
   │     └─ Pod MySQL 10.10.0.3:3306
   │
   └─ Servidor DNS de Kubernetes (CoreDNS):
        basedatos.produccion.es → 10.20.0.1   (ClusterIP)
        apaches.produccion.es   → 10.20.0.2   (ClusterIP)
        ingress.micluster.es    → 10.20.0.3   (LoadBalancer → NodePort → ClusterIP)
```

**Mantenimiento automático.** Si un Pod cae, Kubernetes **actualiza inmediatamente las reglas de Netfilter de todos los nodos** para retirar su IP; si aparecen nuevos Pods, las actualiza para incluirlos. El programa encargado en cada nodo es **kube-proxy** (desplegado como DaemonSet).

**Dos flujos resueltos:**

```
Petición EXTERNA a app1.produccion.es
  1. DNS externo → balanceador externo (172.30.2.1)
  2. balanceador → una máquina del clúster (Netfilter, puerto 30100)
  3. Netfilter/CoreDNS → Pod Ingress Controller
  4. Ingress Controller (reglas Ingress) → Service de los Apaches → un Pod Apache

Comunicación INTERNA Apache → BBDD
  1. El Apache usa la palabra "basedatos"
  2. CoreDNS la resuelve → 10.20.0.1 (ClusterIP del Service)
  3. Netfilter intercepta y reescribe → 10.10.0.3 (un Pod MySQL concreto)
```

---

## 19. Service: ClusterIP, NodePort y LoadBalancer

> Un **Service** es una **IP fija de balanceo de carga** (a nivel de clúster) **+ una entrada en el DNS interno**. Balancea entre todos los Pods que tengan unas **labels** (`selector`) dentro del mismo Namespace.

**Siempre que haya que exponer o trabajar con un puerto, se monta un Service.** Es lo que aporta escalabilidad y alta disponibilidad de cara a los consumidores.

Tres tipos, cada uno **construido sobre el anterior**:

| Tipo | Es… + … | Uso |
|------|---------|-----|
| **ClusterIP** | IP de balanceo + entrada en el DNS interno | Comunicaciones **internas** (BBDD, mensajería, microservicios). El tipo por defecto |
| **NodePort** | ClusterIP + **un puerto (>30000) abierto en cada nodo** | Acceso **externo** asumiendo nosotros el balanceador. Apenas se usa (pruebas) |
| **LoadBalancer** | NodePort + **gestión automatizada de un balanceador externo** | Acceso **externo** con balanceador gestionado automáticamente |

**Estimación realista en producción:**

| Tipo | Cantidad típica |
|------|-----------------|
| ClusterIP | **Casi todos** |
| NodePort | Prácticamente ninguno |
| LoadBalancer | **1** — el del **Ingress Controller** |

> **LoadBalancer necesita un balanceador externo compatible con Kubernetes.** En un *cloud* (AWS, Azure, GCP, IBM Cloud), el proveedor lo aporta (previo pago). *On premises*, el habitual es **MetalLB**.

```yaml
spec:
  type: NodePort
  ports:
    - port: 8888         # puerto del Service (en su ClusterIP)
      targetPort: 80     # puerto en el contenedor
      nodePort: 32000    # puerto externo en cada nodo (>30000)
  selector:
    app: nginx           # Pods backend: los que tengan esta label en este Namespace
```

```bash
kubectl get svc
kubectl describe svc nginx-srv        # ver Endpoints (IPs de los Pods detrás)
# Acceso interno:   curl nginx-srv:8888
# Acceso externo:   curl IP_DE_CUALQUIER_NODO:32000
```

---

## 20. Ingress e Ingress Controller

### 20.1. Proxy reverso

Un **proxy reverso** se sitúa delante de uno o varios servidores y, según reglas (nombre de dominio o ruta), redirige cada petición al servidor adecuado, ocultando la identidad de los servidores reales y aportando balanceo y seguridad.

```
   Cliente  ──►  Proxy reverso  ── app1.miempresa.es     ──►  App 1
                 (192.168.3.117) ── miempresa.es          ──►  App 2
                                 ── miempresa.es/servicios ──►  App 3
   DNS externo:  192.168.3.117  app1.miempresa.es
                 192.168.3.117  miempresa.es
```

(Un **proxy normal** protege la identidad del **cliente**; el **proxy reverso** protege la de los **servidores**.) Ejemplos: Apache `httpd`, NGINX, HAProxy, Envoy.

### 20.2. Ingress Controller e Ingress

En Kubernetes, el proxy reverso se llama **Ingress Controller**. Se despliega como un Pod (expuesto con un Service **LoadBalancer**) y es el **único punto de entrada** externo. Se puede usar el que se prefiera (NGINX, HAProxy, Traefik…).

Como cada Ingress Controller se configura distinto, Kubernetes ofrece un objeto neutro:

> Un **Ingress** es un conjunto de **reglas de configuración** (qué nombre de dominio o ruta va a qué Service) que Kubernetes **traduce automáticamente** a la configuración del Ingress Controller de turno.

```yaml
spec:
  rules:
    - host: www.miapp.ad
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nombre-servicio   # un Service ClusterIP
                port: { number: 80 }
  tls:                                    # HTTPS (certificado en un Secret)
    - hosts: [www.miapp.ad]
      secretName: mi-cert-secret
```

> **HTTPS y certificados.** HTTPS añade a HTTP una capa **TLS** que frustra dos ataques: el *man in the middle* (mediante **cifrado**: la comunicación usa cifrado simétrico, cuya clave se intercambia con cifrado asimétrico) y la *suplantación de identidad* (mediante un **certificado** firmado por una entidad certificadora de confianza, que acredita que el servidor es quien dice ser). El certificado se guarda en un `Secret` y se referencia desde el Ingress/Route.

---

## 21. NetworkPolicy

Una **NetworkPolicy** define **reglas de filtrado de tráfico** entre Pods (un cortafuegos a nivel de Pod). Se aplica a nivel de **Namespace** y selecciona los Pods afectados con labels (`podSelector`).

- `policyTypes: Ingress` — controla **quién puede acceder** a los Pods seleccionados.
- `policyTypes: Egress` — controla **a dónde pueden conectar** los Pods seleccionados.

Orígenes/destinos: `podSelector`, `namespaceSelector` o `ipBlock` (CIDR), restringibles por puerto.

```yaml
spec:
  podSelector:
    matchLabels: { app: es-master }   # a qué Pods se aplica
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: { tipo: elasticSearch }
      ports:
        - { port: 9200, protocol: TCP }
        - { port: 9300, protocol: TCP }
```

> Una política con `podSelector: {}` y `policyTypes: [Ingress]` y sin reglas `ingress` **bloquea todo el tráfico entrante** del Namespace (regla «denegar todo» por defecto).

> En clústeres de producción de empresas medianas/grandes, el filtrado de tráfico suele delegarse hoy en **mallas de servicio** (Istio, Linkerd), que además aportan cifrado mutuo (mTLS), control y observabilidad entre servicios.

---

## 22. Almacenamiento en Kubernetes

### 22.1. Concepto y tipos

> En Kubernetes, los volúmenes **no se asignan a un contenedor, sino a un Pod**. Los contenedores del Pod pueden usarlos, y cada uno decide en qué carpeta montarlos.

Dos grandes familias:

**Volúmenes NO persistentes** — los datos no necesitan sobrevivir al Pod; interesa acceso rápido (local o RAM, nunca en red, para no saturarla).

| Tipo | Uso |
|------|-----|
| `emptyDir` | Carpeta vacía en el host (o en RAM) que **se borra con el Pod**. Para compartir datos entre contenedores del mismo Pod |
| `configMap` | Inyectar ficheros de configuración guardados en `etcd` |
| `secret` | Igual, pero para datos sensibles (almacenados cifrados) |
| `hostPath` | Inyectar una carpeta/fichero que **ya existe** en el host. Uso poco frecuente (monitorización: `/proc`, sockets) |

**Volúmenes persistentes** — los datos deben conservarse aunque el Pod desaparezca. Se almacenan **fuera del clúster**: NFS, iSCSI, cabinas, *clouds* (`awsElasticBlockStore`, `azureDisk`, `gcePersistentDisk`, `cephfs`, `glusterfs`, `vsphereVolume`…).

```
  Velocidad orientativa del soporte:
    RAM        muchísimo más rápida   →  emptyDir en memoria (logs efímeros)
    SSD/NVMe   ~0,65 GB/s para mí     →  datos de servicio
    Red 10Gb   ~1 Gbit/s compartido   →  evitar para datos calientes
```

> **Regla de oro.** En producción **nunca** se guardan datos persistentes en el almacenamiento **local** de un nodo: si la máquina cae, los datos se pierden con ella. Los volúmenes persistentes residen **externamente al clúster**.

### 22.2. PersistentVolume (PV) y PersistentVolumeClaim (PVC)

El reparto de responsabilidades es la clave:

- El **desarrollo** escribe la plantilla de Pod y **pide** almacenamiento con unas características, **sin** saber (ni deber saber) dónde se guardan físicamente los datos. Esa petición es un **PersistentVolumeClaim (PVC)**.
- El **administrador / equipo de almacenamiento** decide la ubicación real y registra el espacio como **PersistentVolume (PV)**.

> Un **PV** es una **referencia a un espacio de almacenamiento ya existente**: Kubernetes **no crea** ese espacio, solo lo referencia.
> Un **PVC** es una **petición de volumen** con características: capacidad, rendimiento, cifrado, redundancia, modo de acceso.

Qué pide el desarrollo en un PVC (no una IP ni una ruta, sino **características**):

- Capacidad (p. ej. 50 Gi).
- Rendimiento (rápido / normal / lento — *backups*).
- Cifrado (sí/no).
- Redundancia (×3, ×4…).
- Modo de acceso (`accessModes`): `ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany`.

**Ámbito:** los **PVC** viven dentro de un **Namespace**; los **PV** son **globales**.

```
   DESARROLLO                          SISTEMAS / ADMIN
   ┌──────────────────────┐            ┌──────────────────────────────┐
   │ PVC mariadb          │            │ PV mariadb                   │
   │ storageClass: rápido │  ◄──────►  │ storageClass: rápido         │ ──► VOLUMEN real
   │ 10Gi  ReadWriteOnce  │  Kubernetes│ 18Gi  ReadWriteOnce          │     (NFS, AWS,
   └──────────────────────┘  los vincula└─ hostPath / nfs / awsEBS …  ┘      cabina…)
        (en un Namespace)     por carac-     (global al clúster)
                              terísticas
```

> **Vinculación (*bind*).** Una vez vinculado un PVC con un PV, **ese PV queda asignado (`Bound`) a ese PVC para siempre**: aunque el PVC se elimine, el PV pasa a `Released` y **no se reasigna** a otro PVC. Kubernetes **no valida** la capacidad indicada; solo la usa para el *match* (un PVC de 10Gi puede vincularse a un PV de 18Gi).

**Para qué sirve realmente el PVC** (más allá de pedir un volumen): para **reutilizar el mismo volumen** entre Pods sucesivos. Al actualizar el software de un Pod, se borra el Pod y se crea uno nuevo, pero el **PVC (y su PV) permanecen**, conservando los datos:

```
   POD-1  ─┐
   POD-2  ─┼─► PVC mariadb ──► PV ──► VOLUMEN (200 TB en la cabina)
   POD-3  ─┘     (7Gi)         (10Gi)
   (al borrar pods y PVC, el PV queda RELEASED y ya no se reasigna)
```

### 22.3. StorageClass y provisionadores

Históricamente, conectar PVC y PV exigía trabajo manual (crear el volumen en el *cloud*/cabina y registrar el PV cada vez, vigilando la pantalla por si entraba una PVC). Hoy se automatiza con un **provisionador**:

> Una **StorageClass** describe una clase de almacenamiento asociada a un **provisionador** (un programa instalado en el clúster, que corre como un Deployment con su propia identidad — ServiceAccount + Role + RoleBinding). Cuando un desarrollador lanza un PVC con esa `storageClassName`, el provisionador **crea automáticamente** el volumen y **registra el PV**.

```
  Plantilla Pod ──► PVC ──► (provisionador) ──► PV ──► StorageClass ──► Volumen real
  "un mariadb     "10Gi    "lo creo en AWS    "id de  "rápido y           (AWS / NFS /
   con datos"      rápido   automáticamente"   AWS"    redundante ×4"      cabina)
                   ×4"
```

> **Ampliar un volumen.** Si el sistema de almacenamiento lo permite, se amplía directamente. Si no, hay que crear PVC, PV y volumen nuevos y copiar los datos a mano del antiguo al nuevo.

```bash
kubectl get pvc          # estado: Pending → Bound
kubectl get pv
kubectl get storageclass
kubectl describe pvc mariadb-peticion-volumen
```

---

## 23. Configuración: ConfigMap, Secret y variables de entorno

### 23.1. El problema de los datos sensibles

Quien escribe la plantilla de Pod es el **desarrollador**, que **no** conoce (ni debe conocer) las credenciales de la base de datos de **producción**. Por tanto, esos datos **no se escriben en la plantilla**, sino que se **referencian**; los valores reales los rellenan los administradores en cada entorno. El desarrollador conoce las credenciales de **desarrollo**, quizá las de **preproducción**, pero **nunca** las de producción.

### 23.2. ConfigMap y Secret

| Objeto | Descripción |
|--------|-------------|
| **ConfigMap** | Guarda datos de configuración (pares clave-valor o ficheros completos) en `etcd` |
| **Secret** | Igual, pero para **datos sensibles**: los valores van en **base64** y se almacenan cifrados |

> El base64 **no es cifrado**: es solo una codificación. El cifrado lo aporta Kubernetes al guardarlo en `etcd`. Un Secret puede crearse sin que su valor aparezca en ningún fichero:
> ```bash
> kubectl create secret generic misecreto --from-literal=password=micontrasena
> ```

**Usos del ConfigMap:** centralizar variables de entorno (un único sitio para los valores), tener toda la configuración junta y **compartir configuración** entre Pods. Quién lo rellena depende del dato: la configuración propia de la aplicación la pone el desarrollador; los datos que cambian por entorno (memoria de la JVM, host de la BBDD) los ponen los administradores.

### 23.3. Cómo consumirlos desde un contenedor

Tres formas:

- **`env` + `valueFrom`**: inyectar **una** clave concreta como variable (`configMapKeyRef` o `secretKeyRef`).
- **`envFrom`**: inyectar **todas** las claves como variables. Si hay conflicto, **`env` tiene prioridad sobre `envFrom`**.
- **Montar como volumen** (`configMap`/`secret` en `volumes`): cada clave se materializa como un **fichero** dentro de la ruta de montaje. Ideal para ficheros de configuración completos (un `nginx.conf`, por ejemplo).

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef: { name: secretitos, key: MARIADB_PASSWORD }
envFrom:
  - configMapRef: { name: configuracion }   # todas las claves de golpe
volumeMounts:
  - name: ficheros-configuracion
    mountPath: /config                       # cada clave → un fichero en /config
```

> **Bloques de texto en YAML.** Para guardar ficheros completos en un ConfigMap se usa `|`:
> - `|`  conserva los saltos internos y termina con **un único** salto final.
> - `|+` conserva **todos** los saltos finales.
> - `|-` **no** incluye salto al final.

---

## 24. Gobierno del clúster: Namespace, LimitRange y ResourceQuota

### 24.1. Namespace

> Un **Namespace** es un **espacio de nombres único** dentro del clúster: una agrupación lógica de objetos donde **no puede haber dos objetos del mismo tipo con el mismo nombre**.

Usos:

- **Agrupar** los recursos relacionados de un despliegue (todo el WordPress: Deployment, StatefulSet, Secrets, ConfigMaps, Services, Ingress…).
- **Separar entornos** (`app1-produccion`, `app1-preproduccion`).
- **Limitar el acceso**: los permisos se asignan a nivel de Namespace (un usuario solo gestiona los recursos de su Namespace).
- **Limitar el uso de recursos** físicos (RAM, CPU, almacenamiento) por Namespace.

Namespaces propios de Kubernetes: `default`, `kube-system` (Pods del plano de control), etc.

> **Cuidado con la resolución entre Namespaces.** Un Service se direcciona como `nombre` (mismo Namespace) o `nombre.namespace` (otro Namespace). Que el WAS de **desarrollo** pueda alcanzar `mariadb.produccion` es técnicamente posible pero **debe impedirse** (con NetworkPolicy o permisos): cruzar entornos es un riesgo grave.

### 24.2. LimitRange y ResourceQuota

Mientras el **desarrollador** pide recursos a nivel de cada contenedor, el **administrador** impone restricciones a nivel de **Namespace**:

| Objeto | Ámbito | Qué controla |
|--------|--------|--------------|
| **LimitRange** | Por **contenedor o Pod** del Namespace | Valores por defecto y mínimos/máximos de CPU y memoria por contenedor/Pod |
| **ResourceQuota** | **Total** del Namespace | Suma máxima: CPU, memoria, nº de Pods, Services, PVCs… |

```yaml
# LimitRange: cotas por contenedor
spec:
  limits:
    - type: Container
      defaultRequest: { cpu: 1000m }   # request por defecto si el dev no pide nada
      default:        { cpu: 1500m }   # limit por defecto
      max: { cpu: 2000m, memory: 2Gi } # tope que el dev puede pedir
      min: { cpu: 500m }               # mínimo que el dev puede pedir
```
```yaml
# ResourceQuota: techo agregado del Namespace
spec:
  hard:
    limits.cpu: 4000m
    limits.memory: 2Gi
    pods: 10
    services: 5
    persistentvolumeclaims: 5
```

> **El scheduler y los requests.** El scheduler usa los `requests` para decidir el nodo: solo coloca un Pod donde pueda **garantizar** esos mínimos. Si ningún nodo tiene recursos suficientes, el Pod queda en estado **`Pending`**. Si dos Pods caben mejor reorganizándose, Kubernetes puede **mover** uno (lo borra en un nodo y lo crea en otro).

### 24.3. RBAC: identidad y permisos (introducción)

Algunos componentes (por ejemplo, un **provisionador de volúmenes**) necesitan **identidad propia** y permisos:

- **ServiceAccount**: la «cuenta» de un proceso dentro del clúster.
- **Role**: definición de permisos (operaciones — `get`, `create`, `delete`, `watch`, `update` — sobre tipos de objeto).
- **RoleBinding**: vincula un ServiceAccount con un Role.

> Ejemplo: un provisionador NFS necesita un ServiceAccount con permisos para `create`/`delete` de PV y `get`/`watch`/`update` de PVC, de modo que pueda crear un PV automáticamente cuando alguien lance un PVC.

---

## 25. Caso integrador: despliegue completo de WordPress

Este caso reúne casi todos los objetos. Se trata de desplegar un WordPress (Apache + PHP) con su base de datos MariaDB, en producción.

### 25.1. Diseño

```
   Namespace: wordpress
   ├─ Secret  datos-bbdd            (usuario, contraseñas, nombre de BBDD)
   ├─ Service servicio-mariadb      (ClusterIP)   ← acceso interno a la BBDD
   ├─ Service servicio-wp           (NodePort)    ← acceso externo a la web
   ├─ StatefulSet statefulset-mariadb (réplicas: 1)
   │     volumeClaimTemplates → PVC propio → PV → volumen
   │     contenedor: mariadb:10.9   (env desde el Secret)
   └─ Deployment  deployment-wp     (réplicas: 2)
         antiafinidad consigo mismo por hostname (repartir las 2 web)
         contenedor: wordpress:6.1
           WORDPRESS_DB_HOST = servicio-mariadb   ← el NOMBRE del Service, no una IP
           WORDPRESS_DB_USER / _PASSWORD / _NAME  ← desde el Secret
         volumen pvc-wp (ReadWriteMany) para los ficheros subidos (imágenes, PDF)
```

### 25.2. Por qué cada decisión

- **MariaDB en StatefulSet**, no Deployment: una base de datos necesita su propio volumen e identidad estable (aunque aquí haya 1 réplica, deja la puerta abierta a crecer correctamente).
- **WordPress en Deployment** con 2 réplicas: es *stateless* en cuanto a proceso; los ficheros que sube el usuario van a un **volumen compartido** (`ReadWriteMany`), de modo que si un usuario sube `informe.pdf` por la réplica WEB-4 y otro lo consulta por la WEB-1, el fichero esté disponible para ambas.
- **Acceso a la BBDD por el nombre del Service** (`servicio-mariadb`), nunca por IP: la IP del Pod de MariaDB cambia; el Service es la IP fija + nombre estable.
- **Antiafinidad** en el Deployment: que las 2 réplicas web no caigan en el mismo nodo (HA).
- **Credenciales en un Secret**: el desarrollador referencia las claves; los administradores ponen los valores reales por entorno.
- **Servicio web como NodePort** solo porque en este laboratorio no hay Ingress Controller; en producción sería ClusterIP detrás de un Ingress/Route.

### 25.3. Operación

```bash
kubectl apply -f wordpress.yaml
kubectl get all -n wordpress
kubectl get pods -n wordpress -o wide
# Acceso externo de laboratorio:
curl http://IP_DE_UN_NODO:30100
```

> La diferencia con un único host y Docker es enorme: aquí Kubernetes garantiza HA (recrea Pods caídos), escalabilidad (`kubectl scale`), balanceo (Services) y persistencia (PVC/PV), todo declarado en un fichero.

---

## 26. El modelo de API y las CLIs: kubectl / oc

### 26.1. Kubernetes es una API REST de recursos

Todo lo que hemos visto —Pods, Services, PVCs…— son **recursos** de una **API REST** que expone el `kube-apiserver`. Las CLIs (`kubectl`, `oc`) y la consola web no son más que **clientes** que hablan con esa API. Comprender el modelo de API es lo que permite trabajar con **cualquier** tipo de objeto, incluso los que no conocemos de memoria.

Cada tipo de objeto pertenece a un **grupo de API** y tiene una **versión**, que es justo lo que indica el campo `apiVersion`:

| `apiVersion` | Grupo | Objetos típicos |
|--------------|-------|-----------------|
| `v1` | *core* (grupo «vacío», histórico) | Pod, Service, Namespace, ConfigMap, Secret, PV, PVC, Node |
| `apps/v1` | `apps` | Deployment, StatefulSet, DaemonSet, ReplicaSet |
| `batch/v1` | `batch` | Job, CronJob |
| `networking.k8s.io/v1` | `networking.k8s.io` | Ingress, NetworkPolicy |
| `autoscaling/v1` | `autoscaling` | HorizontalPodAutoscaler |
| `route.openshift.io/v1` | OpenShift | Route |

> **Madurez de las versiones.** El sufijo indica estabilidad: `v1alpha1` (experimental, puede cambiar o desaparecer), `v1beta1` (preestreno, razonablemente estable) y `v1` (estable, con garantías de compatibilidad). En producción se trabaja con versiones estables.

### 26.2. Descubrir la API: `api-resources` y `explain`

No hace falta memorizar el esquema: la propia API es **autodescriptiva**.

```bash
# Listar TODOS los tipos de objeto, su alias, su grupo y si tienen Namespace
kubectl api-resources
# NAME          SHORTNAMES   APIVERSION         NAMESPACED   KIND
# pods          po           v1                 true         Pod
# deployments   deploy       apps/v1            true         Deployment
# services      svc          v1                 true         Service
# nodes         no           v1                 false        Node

kubectl api-versions          # grupos/versiones disponibles en el clúster

# Documentación de CUALQUIER campo, navegando el esquema
kubectl explain pod
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy.rollingUpdate
```

> `kubectl explain` es la mejor referencia mientras se escribe un manifiesto: describe cada campo, su tipo y si es obligatorio, directamente desde la versión de la API que tiene **ese** clúster.

### 26.3. Sintaxis general y verbos

```
kubectl [VERBO] [TIPO_DE_OBJETO] <argumentos>
```

**Verbos principales:** `get`, `describe`, `create`, `apply`, `delete`, `logs`, `exec`, `scale`, `label`, `rollout`.

**Argumentos habituales:** `-n`/`--namespace NOMBRE`, `--all-namespaces`, `-o wide` (información extra), `-l clave=valor` (filtrar por label).

**Alias de tipos:** `ns` (namespace), `po` (pod), `deploy` (deployment), `svc` (service), `pv`, `pvc`. Otros: `statefulset`, `daemonset`, `ingress`, `node`, `configmap`, `secret`, `job`, `cronjob`.

**Consulta:**

```bash
kubectl get pods -n miapp -o wide
kubectl get all -n miapp                 # casi todos los objetos del Namespace
kubectl describe pod nginx-pod -n miapp
```

**Crear, modificar y borrar** (forma declarativa, la recomendada):

```bash
kubectl create -f fichero.yaml   # CREAR (falla si ya existe)
kubectl apply  -f fichero.yaml   # CREAR o MODIFICAR (idempotente)  ← preferido
kubectl delete -f fichero.yaml   # BORRAR lo definido en el fichero
```

**Logs y ejecución de comandos:**

```bash
kubectl logs NOMBRE_POD -c NOMBRE_CONTENEDOR -n NAMESPACE

kubectl exec NOMBRE_POD -c CONTENEDOR -n NS -- ls /
kubectl exec -it NOMBRE_POD -c CONTENEDOR -n NS -- bash
kubectl exec -it NOMBRE_POD -c CONTENEDOR -n NS -- mysql -u root -p
```

> Los **logs de un contenedor** son la **salida estándar** de su proceso principal.

**Escalado y despliegues:**

```bash
kubectl scale deployment kibana --replicas=10
kubectl rollout status deployment/kibana
kubectl rollout undo  deployment/kibana
kubectl label nodes ip-172-31-11-196 zona=EUROPA
```

> En general, `get`/`describe` se usan para **consultar**; para **crear/modificar/borrar configuraciones** se usa `apply`/`delete -f` sobre ficheros YAML versionados en Git.

### 26.4. Formatos de salida (`-o`)

El verbo `get` admite distintos formatos, muy útiles para inspeccionar o automatizar:

```bash
kubectl get pod nginx-pod -o yaml        # el objeto completo, tal como lo ve la API
kubectl get pod nginx-pod -o json
kubectl get pods -o wide                 # columnas extra (IP, nodo)
kubectl get pods -o name                 # solo los nombres (pod/nginx-pod)

# Extraer un dato concreto con JSONPath
kubectl get pod nginx-pod -o jsonpath='{.status.podIP}'

# Columnas a medida
kubectl get pods -o custom-columns=NOMBRE:.metadata.name,NODO:.spec.nodeName
```

### 26.5. Generar manifiestos sin crear nada (`--dry-run`)

Una técnica muy práctica: pedirle a la CLI que **genere el YAML** de un objeto sin llegar a crearlo, para usarlo como plantilla de partida.

```bash
kubectl create deployment miapp --image=nginx:1.21 --dry-run=client -o yaml > deployment.yaml
kubectl run nginx-pod --image=nginx:1.21 --dry-run=client -o yaml
```

> `--dry-run=client` construye el objeto **localmente** y lo muestra; `--dry-run=server` lo envía al apiserver para **validarlo** (comprueba esquema, cuotas, admission) pero **sin persistirlo**. Es la forma de comprobar que un manifiesto es válido antes de aplicarlo.

### 26.6. Acceso al clúster: login, kubeconfig y contextos

La configuración de conexión (clúster, usuario y Namespace por defecto) se guarda en un fichero **kubeconfig** (`~/.kube/config`). Cada combinación se llama **contexto**.

```bash
# OpenShift: autenticarse (genera/actualiza el kubeconfig)
oc login https://api.micluster:6443 -u usuario
oc whoami                          # quién soy
oc whoami --show-server            # contra qué clúster

# Gestión de contextos (kubectl y oc)
kubectl config get-contexts
kubectl config use-context mi-contexto
kubectl config set-context --current --namespace=miapp   # Namespace por defecto
oc project miapp                   # equivalente en OpenShift
```

### 26.7. Evaluar la salud del clúster

```bash
kubectl get nodes                  # estado de los nodos (Ready / NotReady)
kubectl get pods -A                # Pods de TODOS los Namespaces (-A = --all-namespaces)
kubectl top nodes                  # uso real de CPU/RAM por nodo (requiere metrics-server)
kubectl top pods -n miapp
kubectl get events -n miapp --sort-by=.lastTimestamp   # eventos recientes

# OpenShift: estado de los operadores de plataforma
oc get clusteroperators
```

> Esta capacidad de **consultar la API para evaluar la salud** de un clúster (nodos, Pods, eventos, operadores) es uno de los objetivos centrales del DO180. La **consola web de OpenShift** ofrece esta misma información de forma gráfica (Observe → Dashboards, y la vista de cada recurso).

---

## 27. Bloque OpenShift

**OpenShift** es la distribución de Kubernetes de Red Hat (la versión *open source* se llama **OKD**). Es **Kubernetes** con funcionalidades, objetos y herramientas adicionales para uso empresarial: seguridad reforzada, construcción de imágenes integrada, consola web, gestión de identidades. Todo lo anterior **sigue siendo válido**; este bloque añade lo específico.

### 27.1. El cliente `oc`

`oc` es un **superconjunto de `kubectl`**: admite la misma sintaxis y añade verbos propios.

```bash
oc login https://api.micluster:6443 -u usuario
oc new-project miproyecto
oc get pods                     # idéntico a kubectl
oc new-app …                    # crear una app a partir de código o imagen
oc start-build mibuild
oc rollout restart deploy/miapp
```

### 27.2. Project ≈ Namespace

Un **Project** es un **Namespace** con metadatos y políticas de acceso añadidos. Es la unidad con la que trabajan los usuarios.

```bash
oc new-project desarrollo
oc project desarrollo           # cambiar de proyecto activo
```

### 27.3. Route ≈ Ingress (con DNS externo automatizado)

OpenShift introdujo **Route** antes de que Kubernetes tuviera Ingress; hoy soporta ambos.

> Una **Route** es, conceptualmente, un **Ingress** + la **configuración automatizada del DNS externo**. Expone un Service al exterior a través del **Router** de OpenShift (un Ingress Controller basado en HAProxy), generándole un nombre de host accesible y gestionando la **terminación TLS** (`edge`, `passthrough`, `reencrypt`).

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata: { name: miapp }
spec:
  host: miapp.apps.micluster.com
  to:   { kind: Service, name: miapp-service }
  port: { targetPort: 8080 }
  tls:  { termination: edge }
```

| Kubernetes | OpenShift |
|-----------|-----------|
| Ingress Controller | **Router** (HAProxy) |
| Ingress | **Route** |

### 27.4. Construcción de imágenes en el clúster: S2I, BuildConfig e ImageStream

Una gran aportación de OpenShift es **construir imágenes dentro del propio clúster**, a partir del código fuente.

- **Source-to-Image (S2I)**: toma el **código fuente** de un repositorio y, combinándolo con una **imagen constructora** (*builder image*) del lenguaje, produce automáticamente una imagen lista para ejecutar (en muchos casos, sin Dockerfile).
- **BuildConfig**: objeto que **define cómo construir** una imagen (origen del código, estrategia — S2I, Docker… —, imagen base, disparadores). Cada ejecución genera un objeto **Build**.
- **ImageStream**: abstracción que **referencia y versiona** imágenes dentro de OpenShift; permite que un despliegue reaccione cuando aparece una nueva versión de la imagen.
- **Registry interno**: `image-registry.openshift-image-registry.svc:5000`, donde se publican las imágenes construidas (los CronJobs del curso referencian imágenes con esta forma: `…svc:5000/<proyecto>/<imagen>:<tag>`).

```
Código (Git) ─► BuildConfig (S2I) ─► Build ─► imagen ─► ImageStream ─► Registry interno ─► Deployment
```

### 27.5. DeploymentConfig (objeto histórico)

Antes de adoptar el `Deployment` estándar, OpenShift usaba **DeploymentConfig**, con *triggers* sobre ImageStreams y *hooks* de ciclo de vida. Aún aparece en clústeres y materiales antiguos, pero **hoy se recomienda `Deployment`**. Conviene reconocerlo, pero no es la opción preferente.

### 27.6. Seguridad: Security Context Constraints (SCC)

OpenShift es, por defecto, **más restrictivo** que un Kubernetes estándar. Las **SCC** controlan qué puede hacer un Pod: ejecutarse o no como `root`, rango de UID/GID, acceso a *host paths*, capacidades del kernel…

> Consecuencia práctica muy habitual: muchas imágenes públicas que asumen ejecutarse como **root** fallan en OpenShift bajo la SCC por defecto (`restricted`), que obliga a un **usuario no privilegiado con UID aleatorio**. Las imágenes deben estar preparadas (permisos de grupo adecuados, no asumir un UID fijo).

### 27.7. Compute: Machine vs. Node

OpenShift separa dos conceptos:

- **Machine**: la **máquina** física o virtual (el *host*).
- **Node**: el **proceso** de Kubernetes/OpenShift levantado sobre una máquina, que la integra en el clúster.

Así se pueden tener máquinas aprovisionadas **sin** el proceso de nodo activo, facilitando el **escalado** del clúster (objetos `MachineSet`).

### 27.8. Resumen de extras de OpenShift

| Concepto Kubernetes | En OpenShift |
|---------------------|--------------|
| `kubectl` | **`oc`** (superconjunto) |
| `Namespace` | **Project** |
| `Ingress` | **Route** (+ DNS externo y TLS automatizados) |
| Ingress Controller | **Router** (HAProxy) |
| `Deployment` | `Deployment` (recomendado) / **DeploymentConfig** (histórico) |
| (construcción externa de imágenes) | **S2I + BuildConfig + ImageStream + registry interno** |
| (sin equivalente directo) | **SCC** |
| `Node` | **Machine** (host) + **Node** (proceso) |

---

## 28. Glosario y tabla de equivalencias

### 28.1. Glosario rápido

- **Contenedor**: entorno aislado (red, FS, variables) donde se ejecutan procesos sobre un kernel Linux.
- **Imagen de contenedor**: paquete (`tar`) con una aplicación ya instalada y configurada, identificada por `repositorio:tag`.
- **Pod**: conjunto de contenedores que comparten IP, escalan juntos y se despliegan en el mismo host.
- **Sidecar**: contenedor auxiliar que acompaña al principal dentro del mismo Pod.
- **Plantilla de Pod**: descripción reutilizable de un Pod, base de los objetos generadores.
- **Deployment / StatefulSet / DaemonSet**: objetos que generan y mantienen Pods desde una plantilla.
- **Job / CronJob**: ejecución de tareas que terminan, puntuales o programadas.
- **Service**: IP fija de balanceo + nombre DNS interno para un conjunto de Pods (por labels).
- **Ingress / Ingress Controller**: reglas y proxy reverso para exponer servicios al exterior.
- **PV / PVC / StorageClass**: referencia a almacenamiento / petición / clase con provisionado automático.
- **ConfigMap / Secret**: configuración / datos sensibles, inyectables como variables o ficheros.
- **Namespace**: agrupación lógica con nombres únicos, unidad de permisos y cuotas.
- **LimitRange / ResourceQuota**: límites por contenedor/Pod y cuota total por Namespace.
- **kube-proxy / CoreDNS / etcd / scheduler / apiserver**: componentes del plano de control.

### 28.2. Tabla de equivalencias Kubernetes ↔ OpenShift

| Área | Kubernetes | OpenShift |
|------|------------|-----------|
| Cliente CLI | `kubectl` | `oc` |
| Agrupación lógica | Namespace | Project |
| Exposición externa | Ingress | Route |
| Proxy reverso | Ingress Controller | Router |
| Despliegue de Pods | Deployment | Deployment / DeploymentConfig |
| Construcción de imágenes | (externa) | S2I, BuildConfig, ImageStream |
| Registry | (externo) | Registry interno integrado |
| Seguridad de Pods | PodSecurity (admission) | SCC |
| Host vs proceso de nodo | Node | Machine + Node |

---

> **Carpeta de ejemplos.** Junto a este documento se incluye una carpeta `ejemplos/` con un fichero YAML por cada objeto, comentado en detalle, para usar como referencia y plantilla de partida.
