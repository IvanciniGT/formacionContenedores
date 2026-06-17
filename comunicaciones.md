cliente: oc ->  api-server
                            *.micluster.com -> 192.168.1.200



192.168.1.101:30080
192.168.1.102:30080
192.168.1.103:30080
192.168.1.111:30080
192.168.1.112:30080
192.168.1.113:30080                                 http://miapp

     BC                   miapp:192.168.1.200     MenchuPC
      |                      |                       |
  192.168.1.200:80         DNS                  192.168.3.210      192.168.0.0/16
+--+-------------------------+----------------------+-------------- RED DE MI EMPRESA
+= 192.168.1.101-Nodo1-Maestro
||                Linux
||                      Netfilter
||                              10.10.2.101:3333 => 10.10.1.104:3306
||                              10.10.2.102:80 => 10.10.1.101:80 | 10.10.1.102:80  
||                              192.168.1.101:30080 => 10.10.2.102:80
||                containerd
||                                   scheduler
||                                   controller-manager
||                                   api-server
||                                   coreDNS
||                                      mibbdd=10.10.2.101
||                                      minginx=10.10.2.102
||                                   kubeproxy
||                                   etcd
||                kubelet
||
+= 192.168.1.102-Nodo2-Maestro
||                Linux
||                      Netfilter
||                              10.10.2.101:3333 => 10.10.1.104:3306
||                              10.10.2.102:80 => 10.10.1.101:80 | 10.10.1.102:80  
||                              192.168.1.102:30080 => 10.10.2.102:80
||                containerd
||                                   scheduler
||                                   api-server
||                                   controller-manager
||                                   coreDNS
||                                      mibbdd=10.10.2.101
||                                      minginx=10.10.2.102
||                                   kubeproxy
||                                   etcd
||                kubelet
||
||
+= 192.168.1.103-Nodo3-Maestro
||                Linux
||                      Netfilter
||                              10.10.2.101:3333 => 10.10.1.104:3306
||                              10.10.2.102:80 => 10.10.1.101:80 | 10.10.1.102:80  
||                              192.168.1.103:30080 => 10.10.2.102:80
||                containerd
||                                   kubeproxy
||                                   etcd
||                kubelet
||
||
|| 
+= 192.168.1.111-Nodo1
||                Linux
||                      Netfilter
||                              10.10.2.101:3333 => 10.10.1.104:3306
||                              10.10.2.102:80 => 10.10.1.101:80 | 10.10.1.102:80  
||                              192.168.1.111:30080 => 10.10.2.102:80
||                containerd
||                                   kubeproxy
||                                   pod nginx
|+--10.10.1.101 ------------------------contenedor nginx
||                                           proceso nginx   10.10.1.101:80
||                                                miapp.conf
||                                                    cadenaConexionBBDD=mibbdd:3333
||                kubelet
||
+= 192.168.1.112-Nodo2
||                Linux
||                      Netfilter
||                              10.10.2.101:3333 => 10.10.1.104:3306
||                              10.10.2.102:80 => 10.10.1.101:80 | 10.10.1.102:80  
||                              192.168.1.112:30080 => 10.10.2.102:80
||                containerd
||                                   kubeproxy
||                                   pod mariadb
|+--10.10.1.104 ------------------------contenedor maridb
||                                           proceso mariadb   10.10.1.104:3306
||                kubelet
||
||
+= 192.168.1.113-Nodo3
 |                Linux
||                      Netfilter
||                              10.10.2.101:3333 => 10.10.1.104:3306
||                              10.10.2.102:80 => 10.10.1.101:80 | 10.10.1.102:80  
||                              192.168.1.113:30080 => 10.10.2.102:80
 |                containerd
 |                                   kubeproxy
||                                   pod nginx2
|+--10.10.1.102 ------------------------contenedor nginx
||                                           proceso nginx   10.10.1.102:80
||                                                miapp.conf
||                                                    cadenaConexionBBDD=mibbdd:3333
 |                kubelet
 |
 |
 +- Red virtual sobre la red física: 10.10.0.0/16


SERVICE: 
    ClusterIP = VIPA DE BALANCEO + ENTRADA EN DNS INTERNO DE KUBERNETES
                Virtual IP Address

    NodePort = CLUSTERIP + NAT (Portforwarding a nivel de cada host)... 
                    en un puerto por encima del 30000

    LoadBalancer = NODEPORT + Kubernetes configura autom. el balanceador de carga externo.
    NOTA: Siempre que sea compatible con Kubernetes
        Si contrato un kubernetes de pago, siempre me dan un balanceador externo compatible.
        Si monto el Kubernetes Opensource de google... necesito montarlo yo:
        METALLB