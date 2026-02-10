### Las 3 Capas de Networking en Kubernetes

#### Capa 1: Node Network (RED FÍSICA)
```
RED: 192.168.56.0/24
Quién la configura: Vagrantfile (VirtualBox)
Cuándo: Lab 02 (cuando haces vagrant up)
IPs:
├─ 192.168.56.11 (controlplane01)
├─ 192.168.56.12 (controlplane02)
├─ 192.168.56.21 (node01)
├─ 192.168.56.22 (node02)
└─ 192.168.56.30 (loadbalancer)

Propósito: Nodos comunican entre sí (física)
```

#### Capa 2: Service Network (RED VIRTUAL DE SERVICIOS)
```
RED: 10.96.0.0/24 (SERVICE_CIDR - TÚ lo eliges)
Quién la configura: TÚ (en Lab 04)
Cuándo: Lab 08 cuando inicia el API server
SANs en el API server cert:
├─ 10.96.0.1 (API Service - virtual)
├─ 192.168.56.11 (API Server real - controlplane01)
├─ 192.168.56.12 (API Server real - controlplane02)
├─ 192.168.56.30 (Load balancer - virtual)
└─ kubernetes, kubernetes.default, etc. (DNS)

Propósito: Servicios virtuales dentro de Kubernetes
Ejemplo:
├─ 10.96.0.1 = API Service
├─ 10.96.0.10 = kube-dns (Lab 15)
└─ 10.96.x.x = Otros servicios que crees

NO la configura: Weave/Calico (eso es Capa 3)
```

#### Capa 3: Pod Network (RED VIRTUAL DE PODS)
```
RED: 10.244.0.0/16 (típicamente, configurable)
Quién la configura: CNI (Weave, Calico, etc.)
Cuándo: Lab 13 (Configure Pod Networking)
IPs:
├─ 10.244.1.x = Pods en node01
├─ 10.244.2.x = Pods en node02
└─ etc.

Propósito: Cada pod obtiene una IP virtual en esta red
```

### El Timeline Completo

```
LAB 03: Client Tools
└─ Preparas herramientas (kubectl, cfssl)

LAB 04: AQUÍ - Certificates
├─ Defines: SERVICE_CIDR = 10.96.0.0/24
├─ Calculas: API_SERVICE = 10.96.0.1
├─ Generas cert del API server CON SAN: 10.96.0.1
└─ Distribuyes certs

LAB 07: Bootstrap etcd
├─ Inicia etcd con certs
└─ Almacena estado de Kubernetes

LAB 08: Bootstrap API Server
├─ Inicia con flag: --service-cluster-ip-range=10.96.0.0/24
├─ El API server dice: "Soy 10.96.0.1 dentro de Kubernetes"
└─ Ahora Kubernetes "existe"

LAB 10: Configure kubelet
├─ Kubelet en workers conecta a 10.96.0.1 (API Service)
└─ No necesita saber que el API real está en 192.168.56.11

LAB 13: Configure Pod Networking
├─ Instala Weave/Calico (CNI)
├─ Configura Pod Network: 10.244.0.0/16
└─ Pods obtienen IPs en 10.244.x.x

RESULTADO:
├─ Nodos se ven por: 192.168.56.x (física, real)
├─ Servicios se ven por: 10.96.0.x (virtual, API configura)
└─ Pods se ven por: 10.244.x.x (virtual, CNI configura)
```

### Servicios Core en Controlplane01

```
Componentes que CORREN (procesos Linux):

1. kube-apiserver
   - Corre en: 192.168.56.11 (IP real)
   - Accesible por: 10.96.0.1 (IP virtual del servicio)
   - Propósito: Brain del cluster

2. etcd
   - Corre en: 192.168.56.11 (y replicado en .12)
   - No tiene IP virtual (es backend)
   - Propósito: Base de datos

3. kube-controller-manager
   - Corre en: 192.168.56.11
   - Conecta a API via: 10.96.0.1
   - Propósito: Loops de control

4. kube-scheduler
   - Corre en: 192.168.56.11
   - Conecta a API via: 10.96.0.1
   - Propósito: Coloca pods en nodos

Servicios VIRTUALES creados por API (no son procesos, son abstracciones):

5. kubernetes (el API Service)
   - IP virtual: 10.96.0.1
   - Puerta de entrada a todo

6. kube-dns (Lab 15)
   - IP virtual: 10.96.0.10 (típicamente)
   - Resuelve nombres de servicios

7. Otros servicios que crees
   - IPs virtuales: 10.96.0.x
   - Los creas con: kubectl create service
```

### Por Qué SERVICE_CIDR No Lo Configura Weave/Calico

```
Confusión común:
"Weave/Calico configuran la red, ¿no configuran SERVICE_CIDR?"

NO. Son capas diferentes:

Weave/Calico = Pod Network (10.244.x.x)
├─ Configura IPs de PODS
├─ Pods en diferentes nodos pueden hablar
└─ Lab 13

API Server = Service Network (10.96.x.x)
├─ Configura IPs de SERVICIOS
├─ Servicios son abstracciones sobre pods
└─ Lab 08
```

### La Clave: API_SERVICE Es Virtual

```
192.168.56.11 = IP REAL
  ↑
  El servidor está AQUÍ físicamente

10.96.0.1 = IP VIRTUAL
  ↑
  Los clientes creen que está AQUÍ dentro de K8s
  
Kubernetes traduce:
  Cliente: "Voy a 10.96.0.1"
  → Kubernetes: "Eso es 192.168.56.11"
  → Conexión establecida

Si el API server se mueve:
  Cliente: "Voy a 10.96.0.1" (sin cambios)
  → Kubernetes: "Ahora es 192.168.56.12" (cambio interno)
  → Conexión establecida
```
