# Ejemplos de objetos de Kubernetes / OpenShift

Un fichero por tipo de objeto, **comentado y listo para copiar y pegar**. Acompañan a `DO180-Documentacion.md`.

## Convenciones

- Indentación de **2 espacios** (la convención más extendida en YAML de Kubernetes).
- Imágenes con **tag fijo** (nunca `latest`), siguiendo la buena práctica del curso.
- Todos los manifiestos son **válidos**: se aplican con `kubectl apply -f <fichero>` (o `oc apply -f` en OpenShift).

## Orden sugerido de aplicación

```bash
# 1) Crear el Namespace y trabajar dentro de él
kubectl apply -f 00-namespace.yaml
kubectl config set-context --current --namespace=formacion   # (o: oc project formacion)

# 2) Configuración y almacenamiento antes que las cargas que los usan
kubectl apply -f 07-configmap.yaml
kubectl apply -f 08-secret.yaml
kubectl apply -f 11-persistentvolume.yaml
kubectl apply -f 10-persistentvolumeclaim.yaml

# 3) Cargas de trabajo
kubectl apply -f 01-pod.yaml
kubectl apply -f 03-deployment.yaml
kubectl apply -f 04-statefulset.yaml

# 4) Comunicaciones
kubectl apply -f 05-service.yaml
kubectl apply -f 06-ingress.yaml
```

## Índice de ficheros

| Fichero | Objeto(s) | Bloque |
|---------|-----------|--------|
| `00-namespace.yaml` | Namespace | Estructura |
| `01-pod.yaml` | Pod (mínimo) | Workload |
| `02-pod-multicontenedor.yaml` | Pod (initContainers + sidecar + volumen) | Workload |
| `03-deployment.yaml` | Deployment (estrategia, recursos, probes) | Workload |
| `04-statefulset.yaml` | StatefulSet (volumeClaimTemplates) | Workload |
| `05-service.yaml` | Service: ClusterIP / NodePort / LoadBalancer | Red |
| `06-ingress.yaml` | Ingress | Red |
| `07-configmap.yaml` | ConfigMap | Configuración |
| `08-secret.yaml` | Secret | Configuración |
| `09-pod-consumo-configmap-secret.yaml` | Pod que consume ConfigMap/Secret | Configuración |
| `10-persistentvolumeclaim.yaml` | PersistentVolumeClaim | Almacenamiento |
| `11-persistentvolume.yaml` | PersistentVolume | Almacenamiento |
| `12-pod-probes-recursos.yaml` | Pod con probes y requests/limits | Fiabilidad |
| `13-horizontalpodautoscaler.yaml` | HorizontalPodAutoscaler | Fiabilidad |
| `14-limitrange.yaml` | LimitRange | Gobierno |
| `15-resourcequota.yaml` | ResourceQuota | Gobierno |
| `16-networkpolicy.yaml` | NetworkPolicy | Gobierno |
| `17-afinidades.yaml` | nodeSelector / nodeAffinity / podAntiAffinity | Gobierno |
| `18-job.yaml` | Job | Batch |
| `19-cronjob.yaml` | CronJob | Batch |
| `20-route-openshift.yaml` | Route | OpenShift |
| `21-buildconfig-imagestream-openshift.yaml` | ImageStream + BuildConfig (S2I) | OpenShift |

> Los ficheros `20-*` y `21-*` son **exclusivos de OpenShift**: usan `oc apply -f` y no funcionan en un Kubernetes puro.
