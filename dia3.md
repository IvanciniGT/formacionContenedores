
oc          VERBO     TIPO_DE_OBJETO  <args> --namespace    NOMBRE_DE_NAMESPACE
kubectl                                      -n

TIPOS_DE_OBJETOS:

Node        - Representa una máquina que tengo en el cluster de kubernetes
Namespace   - Entorno aislado donde trabajar dentro del cluster

Pod         - Conjunto de contenedores que despliegan y escalan juntos.
Plantillas de pods:
    Deployment          Plantilla de Pod + Número de réplicas
    Statefulset         Plantilla de Pod + Plantilla de Volumenes + Número de réplicas
    Daemonset           Plantilla de Pod + de la que Kubernetes monta 1 réplica or nodo

ConfigMaps
Secrets

Services
Ingress
IngressClasses
PersistentVolumeClaims
PersistenVolumes
StorageClasses

....

VERBOS:

    get         Listar los objetos de un tipo
    describe    Ver el detalle de un objeto
    delete