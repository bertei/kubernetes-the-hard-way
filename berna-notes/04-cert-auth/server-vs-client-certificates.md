# Server vs Client Certificates - SANs Explicado

## La Regla de Oro

```
┌─ SERVER CERTIFICATE (El servidor prueba su identidad) ─────┐
│                                                             │
│ El servidor RECIBE conexiones desde MÚLTIPLES direcciones  │
│ → NECESITA SANs (Subject Alternative Names)                │
│                                                             │
│ Ejemplos: kube-apiserver, etcd-server                      │
└─────────────────────────────────────────────────────────────┘

┌─ CLIENT CERTIFICATE (El cliente prueba su identidad) ──────┐
│                                                             │
│ El cliente INICIA conexión hacia UN SERVIDOR específico     │
│ → NO necesita SANs                                          │
│                                                             │
│ Ejemplos: admin.crt, kube-controller-manager.crt, etc.     │
└─────────────────────────────────────────────────────────────┘
```

---

## Analogía: Hotel vs Huéspedes

### El Hotel (Server Certificate - API Server)

```
┌─ Hotel Central ────────────────────────────────────┐
│                                                    │
│ El hotel es accesible desde MÚLTIPLES direcciones:│
│                                                    │
│ ├─ Dirección oficial: Calle Principal 100         │
│ ├─ GPS/Google Maps: COORDENADAS_XYZ               │
│ ├─ Por teléfono lo llaman: "Hotel Central"        │
│ ├─ Reservaciones usando: otro número              │
│ └─ Por referencia local: "Near the stadium"       │
│                                                    │
│ Los huéspedes lo atacan desde 5 direcciones       │
│ diferentes                                         │
│                                                    │
│ EL HOTEL NECESITA un certificado que diga:        │
│ "Soy válido en TODAS estas 5 direcciones"         │
│                                                    │
│ → Certificate con SANs = [.11, .12, .30, 10.96.0.1, localhost]
└────────────────────────────────────────────────────┘
```

### Los Huéspedes (Client Certificates)

```
┌─ Huéspedes ────────────────────────────────────────┐
│                                                    │
│ Cada huésped:                                      │
│                                                    │
│ ├─ Solo participa en UNA dirección: su habitación │
│ ├─ Se conecta HACIA el hotel (no es atacado aquí) │
│ ├─ Siempre usa el mismo teléfono para llamar      │
│ └─ No necesita probar validez desde 5 lugares     │
│                                                    │
│ Los huéspedes NO necesitan SANs:                  │
│ ├─ Admin.crt: "Soy admin" (punto)                │
│ ├─ Controller-manager.crt: "Soy controller-mgr"  │
│ └─ Scheduler.crt: "Soy scheduler"                 │
│                                                    │
│ → Certificate SIN SANs (más simple)               │
└────────────────────────────────────────────────────┘
```

---

## Los Casos Específicos en Tu Lab

### 1. kube-apiserver.crt (SERVER - NECESITA SANs)

```
¿QUÉ ES?
├─ Servidor que RECIBE conexiones
├─ Múltiples clientes lo atacan desde diferentes direcciones

¿QUIÉN SE CONECTA A ÉL?
├─ kubectl → desde 192.168.56.30 (load balancer)
├─ kubelet → desde 10.96.0.1 (service IP internal)
├─ controller-manager → desde localhost
├─ scheduler → desde localhost
├─ etcd → desde 192.168.56.11
└─ ... muchos otros

¿CUÁNTAS DIRECCIONES DIFERENTES? 5+

SOLUCIÓN: Le dices al certificado "eres válido en TODAS"
SANs = [.11, .12, .30, 10.96.0.1, localhost, kubernetes, ...]

FILE CONFIG: openssl.cnf
├─ extendedKeyUsage = serverAuth
├─ subjectAltName = @alt_names
└─ [alt_names] con todas las IPs/DNS
```

### 2. admin.crt, kube-controller-manager.crt, etc. (CLIENTS - SIN SANs)

```
¿QUÉ SON?
├─ Clientes que INICIAN conexiones
├─ Siempre conectan AL API server (dirección conocida)
├─ NO son servidores (no reciben conexiones)

¿HACIA DÓNDE SE CONECTAN?
├─ admin → API server en 192.168.56.30
├─ controller-manager → API server en 10.96.0.1
├─ scheduler → API server en 10.96.0.1
└─ kube-proxy → API server en 10.96.0.1

¿CUÁNTAS DIRECCIONES? 1 (la dirección del API server)

NO NECESITAN SANs:
├─ admin.crt: "Soy admin" (suficiente)
├─ kube-controller-manager.crt: "Soy controller-manager"
└─ etc.

FILE CONFIG: Ninguno (openssl básico, sin config file)
```

### 3. etcd-server.crt (SERVER - NECESITA SANs)

```
¿QUÉ ES?
├─ Servidor de base de datos de Kubernetes
├─ Corre en AMBOS control planes (.11 y .12)

¿QUIÉN SE CONECTA A ÉL?
├─ etcd1 (en .11) ↔ etcd2 (en .12) [cluster peer]
├─ API server (en .11 o .12)
└─ Desde ambas IPs

¿CUÁNTAS DIRECCIONES? 2

SOLUCIÓN: Le dices "eres válido en .11 Y .12"
SANs = [.11, .12, localhost]

FILE CONFIG: openssl-etcd.cnf
├─ extendedKeyUsage = serverAuth (es servidor)
├─ subjectAltName = @alt_names
└─ [alt_names] con .11, .12, 127.0.0.1
```

---

## El Caso Especial: apiserver-kubelet-client.crt

### El Rol Normal (Esperado)

```
┌─ NORMAL: Kubelet → API Server ────────────────────┐
│                                                    │
│ Kubelet INICIA conexión hacia API Server         │
│                                                    │
│ Kubelet: "Hola, soy kubelet"                      │
│ Presenta: kubelet.crt                             │
│                                                    │
│ API Server verifica:                              │
│ ├─ ¿Está firmado por MI ca.crt? SÍ               │
│ ├─ ¿CN=system:kubelet? SÍ                        │
│ └─ "OK, eres un kubelet legítimo"                │
│                                                    │
│ Resultado: Kubelet puede pedir información       │
└────────────────────────────────────────────────────┘
```

### El Rol Extra (Menos Común)

```
┌─ EXTRA: API Server → Kubelet ─────────────────────┐
│                                                    │
│ A veces el API server necesita conectarse AL      │
│ kubelet para:                                      │
│ ├─ Obtener logs del pod                          │
│ ├─ Ejecutar comandos (kubectl exec)              │
│ ├─ Port-forward                                   │
│ └─ Debugging                                      │
│                                                    │
│ PERO: El API server también necesita un cert cuando│
│ actúa como CLIENTE                                │
│                                                    │
│ API Server: "Hola, soy el API server"             │
│ Presenta: apiserver-kubelet-client.crt            │
│                                                    │
│ Kubelet verifica:                                 │
│ ├─ ¿Está firmado por MI ca.crt? SÍ               │
│ ├─ ¿CN=kube-apiserver-kubelet-client? SÍ         │
│ ├─ ¿O=system:masters? SÍ (admin!)                │
│ └─ "OK, eres API server, te doy acceso"          │
│                                                    │
│ Resultado: API server obtiene logs/exec           │
└────────────────────────────────────────────────────┘
```

### ¿Por Qué NO Tiene SANs Como API Server?

```
API Server cuando actúa como CLIENTE:
├─ SE CONECTA hacia kubelet (en node01 por ejemplo)
├─ Siempre hacia la MISMA dirección (el kubelet)
├─ NO es atacado desde múltiples direcciones
├─ NO necesita SANs

PERO: Necesita O=system:masters
├─ Porque no todos los clientes pueden pedir logs/exec
├─ Solo el admin (API server) debe poder hacerlo
└─ La O= dice:"Tengo permisos de administrador"

FILE CONFIG: openssl-kubelet.cnf
├─ extendedKeyUsage = clientAuth (es cliente)
├─ NO hay subjectAltName
└─ NO hay [alt_names]
```

---

## ¿Qué Son los Archivos .cnf?

### Archivos de Configuración OpenSSL

Los `.cnf` son **archivos de configuración para OpenSSL** que especifican propiedades del certificado.

### openssl.cnf (API Server - CON SANs)

```bash
cat > openssl.cnf <<EOF
[req]
req_extensions = v3_req              # "Usa extensiones v3"
distinguished_name = req_distinguished_name

[req_distinguished_name]             # (vacío, no necesitamos)

[v3_req]                             # Extensiones v3
basicConstraints = critical, CA:FALSE
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth        # ← "SOY UN SERVIDOR"
subjectAltName = @alt_names          # ← "NECESITO SANs"

[alt_names]                          # Lista de "identidades válidas"
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1
IP.2 = 192.168.56.11
IP.3 = 192.168.56.12
IP.4 = 192.168.56.30
IP.5 = 127.0.0.1
EOF
```

**Qué debes entender:**
```
[req] = Sección de SOLICITUD de certificado
  ├─ req_extensions: "Qué extensiones usar"
  └─ distinguished_name: "Qué campos pedir"

[v3_req] = Extensiones V3 del certificado
  ├─ basicConstraints: "No es una CA"
  ├─ keyUsage: "Para qué se puede usar esta clave"
  ├─ extendedKeyUsage = serverAuth: "Soy un servidor"
  └─ subjectAltName: "Necesito SANs"

[alt_names] = LA LISTA REAL de SANs
  ├─ DNS.X: direcciones por nombre
  └─ IP.X: direcciones por IP
```

### openssl-kubelet.cnf (API Server → Kubelet Client - SIN SANs)

```bash
cat > openssl-kubelet.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name

[req_distinguished_name]

[v3_req]
basicConstraints = critical, CA:FALSE
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth        # ← "SOY UN CLIENTE"
                                     # ← NO hay subjectAltName
EOF
```

**Qué cambió:**
```
extendedKeyUsage = clientAuth        ← "Soy UN CLIENTE"
NO hay: subjectAltName               ← "No necesito SANs"
NO hay: [alt_names]                  ← "No necesito lista"
```

### openssl-etcd.cnf (etcd Server - CON SANs)

```bash
cat > openssl-etcd.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name

[req_distinguished_name]

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names          # ← "NECESITO SANs (soy servidor)"

[alt_names]                          # Lista reducida (solo 2 IPs)
IP.1 = 192.168.56.11
IP.2 = 192.168.56.12
IP.3 = 127.0.0.1
EOF
```

---

## Tabla Resumen: Quién Necesita Qué

| Certificado | Tipo | ¿SANs? | Razón | Config File |
|-------------|------|--------|-------|-------------|
| **ca.crt** | Root CA | ❌ NO | Es la CA, solo firma | N/A |
| **admin.crt** | Client | ❌ NO | Se conecta a 1 lugar (API) | Ninguno |
| **kube-apiserver.crt** | Server | ✅ SÍ | Recibido desde 5+ direcciones | openssl.cnf |
| **kube-controller-manager.crt** | Client | ❌ NO | Se conecta a 1 lugar (API) | Ninguno |
| **kube-scheduler.crt** | Client | ❌ NO | Se conecta a 1 lugar (API) | Ninguno |
| **etcd-server.crt** | Server | ✅ SÍ | Recibido desde 2+ direcciones | openssl-etcd.cnf |
| **kube-proxy.crt** | Client | ❌ NO | Se conecta a 1 lugar (API) | Ninguno |
| **apiserver-kubelet-client.crt** | Client | ❌ NO | API se conecta a kubelet (1 lugar) | openssl-kubelet.cnf |
| **service-account.crt** | Special | ❌ NO | Usado para firmar tokens | Ninguno |

---

## Diferencias Clave en la Creación

### Cliente (Sin SANs) - Ejemplo: admin.crt

```bash
# PASO 1: Generar clave privada
openssl genrsa -out admin.key 2048

# PASO 2: Crear CSR (SIN config file)
openssl req -new -key admin.key \
  -subj "/CN=admin/O=system:masters" \
  -out admin.csr

# PASO 3: Firmar (SIN -extensions ni -extfile)
openssl x509 -req -in admin.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out admin.crt \
  -days 1000

# NOTA: Muy simple, no necesita config file
```

### Servidor (Con SANs) - Ejemplo: kube-apiserver.crt

```bash
# PASO 1: Generar clave privada
openssl genrsa -out kube-apiserver.key 2048

# PASO 2: Crear CSR (CON config file)
openssl req -new -key kube-apiserver.key \
  -subj "/CN=kube-apiserver/O=Kubernetes" \
  -out kube-apiserver.csr \
  -config openssl.cnf          ← INCLUYE config

# PASO 3: Firmar (CON -extensions y -extfile)
openssl x509 -req -in kube-apiserver.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out kube-apiserver.crt \
  -extensions v3_req \         ← APLICA extensiones
  -extfile openssl.cnf \       ← DEL config file
  -days 1000

# NOTA: Más complejo, necesita config file con SANs
```

---

## El Flujo Completo: Por Qué Existen Estos Certificados

```
┌─────────────────────────────────────────────────────┐
│ NORMAL: Kubelet → API Server                        │
├─────────────────────────────────────────────────────┤
│                                                     │
│ 1. kubelet.crt (client)                             │
│    "Hola, soy kubelet en node01"                    │
│    API server: "✓ Eres legítimo, dame acceso"      │
│                                                     │
│ 2. Kubelet puede:                                   │
│    - Leer especificación de pods                    │
│    - Reportar estado del nodo                       │
│    - Pedir información                              │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ EXTRA: API Server → Kubelet (logs/exec)             │
├─────────────────────────────────────────────────────┤
│                                                     │
│ 1. apiserver-kubelet-client.crt (client)            │
│    "Hola, soy el API server"                        │
│    O=system:masters: "Soy admin"                    │
│    Kubelet: "✓ Eres admin, te doy todo acceso"     │
│                                                     │
│ 2. API server puede:                                │
│    - kubectl logs pod-name                          │
│    - kubectl exec pod-name -- command               │
│    - kubectl port-forward pod-name 80:8080         │
│    - Debugging                                      │
└─────────────────────────────────────────────────────┘
```

---

## Resumen Simple

```
LA PREGUNTA CLAVE: ¿Soy un SERVIDOR o un CLIENTE?

┌─ SERVIDOR ────────────────────────────────┐
│ Recibo conexiones desde MÚLTIPLES lugares  │
│ → NECESITO SANs (para todas esas lugares)  │
│                                            │
│ Ejemplos:                                  │
│ ├─ API server (atacado desde .11, .12,    │
│ │             .30, 10.96.0.1, localhost)  │
│ └─ etcd (atacado desde .11, .12)           │
└────────────────────────────────────────────┘

┌─ CLIENTE ─────────────────────────────────┐
│ Conecto SIEMPRE al mismo lugar (API srv)   │
│ → NO necesito SANs                         │
│                                            │
│ Ejemplos:                                  │
│ ├─ admin (conecta a API)                  │
│ ├─ controller-manager (conecta a API)     │
│ ├─ scheduler (conecta a API)              │
│ ├─ kube-proxy (conecta a API)             │
│ └─ apiserver-kubelet-client (conecta a    │
│    kubelet, pero siempre el mismo)        │
└────────────────────────────────────────────┘
```

---

## Cheat Sheet Rápido

### ¿Necesito SANs?

```
❓ ¿Recibo conexiones desde MÚLTIPLES direcciones?
  ├─ SÍ → ✅ NECESITO SANs (soy servidor)
  └─ NO → ❌ NO necesito SANs (soy cliente)
```

### ¿Necesito Config File (.cnf)?

```
❓ ¿Necesito SANs?
  ├─ SÍ → ✅ Config file CON [alt_names]
  └─ NO → ❌ Sin config file (o muy simple)
```

### ¿Qué extendedKeyUsage Uso?

```
❓ ¿Soy servidor o cliente?
  ├─ SERVIDOR → extendedKeyUsage = serverAuth
  └─ CLIENTE → extendedKeyUsage = clientAuth
```

---

## Ejercicio: Identifica Server vs Client

Para cada certificado, di si es Server o Client:

1. **admin.crt** → ¿? (respuesta al final)
2. **kube-apiserver.crt** → ¿?
3. **kube-controller-manager.crt** → ¿?
4. **etcd-server.crt** → ¿?
5. **apiserver-kubelet-client.crt** → ¿?
6. **kube-proxy.crt** → ¿?

---

## Respuestas

```
1. admin.crt → CLIENT
   (conecta al API server)

2. kube-apiserver.crt → SERVER
   (recibe de kubectl, kubelet, etcd, etc.)

3. kube-controller-manager.crt → CLIENT
   (conecta al API server)

4. etcd-server.crt → SERVER
   (recibe de etcd peers y API server)

5. apiserver-kubelet-client.crt → CLIENT
   (API server conecta al kubelet)

6. kube-proxy.crt → CLIENT
   (conecta al API server)
```

---

¿Ahora está cristalino? **La regla de oro es: Server = múltiples direcciones = SANs. Cliente = una dirección conocida = sin SANs** 🔐

