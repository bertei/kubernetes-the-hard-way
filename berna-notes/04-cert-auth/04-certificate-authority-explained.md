# Lab 04: Certificate Authority - Conceptos + Ejecución

**Objetivo:** Generar la infraestructura de confianza (PKI) que asegura TODO el cluster

**Tiempo estimado:** 1-2 horas (lectua + ejecución)

**Outcome:** 9 certificados firmados por CA, listos para todos los componentes K8s

---

## PHASE 1: Conceptual Foundation

### ¿Por Qué Certificados? (Security Fundamentals)

#### El Problema: Kubernetes es Distribuido

Kubernetes tiene MUCHAS máquinas hablando entre sí por la red:

```
                    api-server
                    /    |    \
                   /     |     \
              kubelet  etcd   controller-manager
               /  \      |       /
             pod  node   node   node
```

**La red es INSEGURA por defecto:**
```
❌ Sin certificados:
  kubelet → "Dame info de mi pod"
  api-server → "Ok, aquí está TODOS los datos"
  
  Problema: ¿Cómo sabe api-server que ese kubelet es legítimo?
  Problema: ¿Qué pasa si alguien intercepta el tráfico?
```

#### Los 3 Problemas que Resuelven los Certificados

| Problema | Solución | Cómo |
|----------|----------|------|
| **Encriptación** | Datos secretos en tránsito | TLS (HTTPS para máquinas) |
| **Autenticación** | "¿Eres realmente quién dices ser?" | Certificado = ID verificable |
| **Autorización** | "¿Qué estás permitido hacer?" | Roles en el certificado |

#### Ejemplo Real: Con Certificados

```
kubelet (192.168.56.21) → api-server (192.168.56.11)

1. kubelet: "Aquí está mi certificado (firmado por CA)"
   └─ certificate: kubelet.crt + kubelet.key

2. api-server verifica:
   ✅ ¿Este cert está firmado por MI ca.crt? SÍ
   ✅ ¿Está vigente (no expirado)? SÍ
   ✅ ¿Es realmente un kubelet? SÍ (CN=kubelet)

3. api-server CONFÍA en kubelet
   └─ Abre conexión TLS (datos encriptados)
   └─ kubelet auténtico ✓

4. kubelet hace solicitud: "Dame info del pod"
   └─ Viaja ENCRIPTADO por TLS
   └─ api-server verifica: ¿Puede un kubelet pedir esto? SÍ
   └─ Responde con datos ✅
```

#### Sin Certificados = Sin Kubernetes

```
❌ Kubernetes NO FUNCIONA sin TLS/Certificados:
  - Componentes no se confían entre sí
  - Datos viajan en PLANO (cualquiera puede espiar)
  - No hay manera de verificar identidad
  - Cualquiera puede pretender ser api-server
  
✅ Por eso Lab 04 es CRÍTICO:
  - Todo lo que hacemos después depende de estos certificados
  - Si fallan los certs, falla TODO el cluster
```

---

### ¿Qué es una CA? (Certificate Authority)

#### El Concepto: Oficina de Pasaportes

Una **CA** es una **entidad confiable** que firma (emite) certificados:

```
CA = Certificate Authority = Oficina de Pasaportes

La CA tiene:
├─ ca.crt (su certificado público - su "sello oficial")
├─ ca.key (su clave privada - máquina para hacer sellos, SECRETO)
└─ Poder de FIRMAR certificados (nadie más puede)

Cuando alguien quiere un certificado:
1. Pide: "Quiero un cert para el API server"
2. CA verifica: "¿Es realmente el API server?"
3. CA firma: Usa ca.key para firmar
4. Resultado: api-server.crt (público) + api-server.key (secreto del API server)

Cualquiera con ca.crt puede VERIFICAR que la CA lo firmó
```

#### La Cadena de Confianza

```
┌─────────────────────────────────────────────┐
│         CA (Certificate Authority)          │
│   (controlplane01 es la CA en tu setup)    │
│                                             │
│  ca.crt (PÚBLICO - distribuido a todos)   │
│  ca.key (SECRETO - solo la CA)            │
└─────────────────────────────────────────────┘
          │ firma │ firma │ firma │ firma
          ▼       ▼       ▼       ▼
      ┌────────┬──────────┬──────────┬──────┐
      │ Admin  │ API Srv  │   etcd   │ etc. │
      │ Cert   │  Cert    │  Cert    │      │
      └────────┴──────────┴──────────┴──────┘

Cada certificado dice:
"Yo soy [COMPONENT], y la CA me firmó"

Cada componente verifica:
"¿Mi CA (ca.crt) firmó este certificado? SÍ → Confío"
```

#### Por Qué Esto Funciona

```
Analógía: Pasaportes internacionales

País A = CA
Sello del País A = ca.crt

Ciudadano Juan:
- Va a País A
- Solicita pasaporte
- País A lo firma con su sello
- Juan obtiene: pasaporte (verificable)

Juan llega a País B:
- Muestra pasaporte
- País B verifica: "¿Este sello es del País A que confío?" SÍ
- País B CONFÍA en Juan (porque confía en País A)
- Juan entra ✅

En Kubernetes:

kubelet llega a api-server:
- Muestra certificado (firmado por CA)
- API server verifica: "¿La CA que confío lo firmó?" SÍ
- API server CONFÍA en kubelet (porque confía en CA)
- kubelet obtiene datos ✅
```

---

### Los 9 Certificados (Qué Componente Usa Cuál)

#### Tabla de Referencia Rápida

| # | Certificado | Componente | Propósito | Quién lo Verifica | Tipo |
|---|-------------|-----------|----------|------------------|------|
| 1 | **ca.crt + ca.key** | CA (controlplane01) | Raíz de confianza, firma todos | Todos | Root |
| 2 | **admin.crt + key** | kubectl (tú) | Acceso admin al cluster | API Server | Client |
| 3 | **kube-apiserver.crt + key** | API Server | Identidad del API server | kubectl, kubelet, etcd | Server |
| 4 | **kube-controller-manager.crt + key** | Controller Manager | Autentica con API Server | API Server | Client |
| 5 | **kube-scheduler.crt + key** | Scheduler | Autentica con API Server | API Server | Client |
| 6 | **etcd-server.crt + key** | etcd | Seguridad entre etcd peers | etcd peers, API Server | Server |
| 7 | **kubelet.crt + key** | kubelet (workers) | Autentica con API Server | API Server | Client |
| 8 | **kube-proxy.crt + key** | kube-proxy (workers) | Autentica con API Server | API Server | Client |
| 9 | **service-account.crt + key** | Pods (Service Accounts) | Confianza pod → API Server | Pods | Service Account |
| BONUS | **apiserver-kubelet-client.crt + key** | API Server (como cliente) | Conectarse a kubelet | kubelet | Client |

#### Desglose por Componente

##### 1. CA (Certificate Authority)

```
ca.crt = Certificado raíz (PÚBLICO)
ca.key = Clave privada de la CA (SECRETO - NUNCA copiar)

Propósito:
- ca.crt: Distribuye a TODOS los nodos (para verificar otros certs)
- ca.key: Mantén SOLO en controlplane01 (para firmar nuevos certs)

Ejemplo:
$ ls -la ca.*
-rw-r--r-- ca.crt  (público, 1KB)
-rw------- ca.key  (secreto, permisos 600)
```

##### 2. Admin Certificate (Para Ti - kubectl)

```
admin.crt + admin.key

Propósito: Tú como ADMIN del cluster
- CN=admin (el nombre del usuario)
- O=system:masters (el GRUPO - da permisos totales)

Cuando haces `kubectl get pods`:
kubect lee admin.crt + admin.key
→ Se conecta a API server con estos certs
→ API server verifica: "¿CN=admin? ¿O=system:masters?" SÍ
→ API server: "Eres admin, puedes ver TODO"

Analogía:
- admin.crt = Tu pasaporte
- O=system:masters = Sello de "ADMINISTRADOR"
```

##### 3. API Server Certificate (El Componente Más Importante)

```
kube-apiserver.crt + kube-apiserver.key

Propósito: API Server PRUEBA su identidad
- Cuando kubectl se conecta: "Soy realmente el API server"
- Cuando kubelet se conecta: "Soy realmente el API server"
- Cuando etcd se conecta: "Soy realmente el API server"

Subject Alternative Names (MUY IMPORTANTE):
- 192.168.56.11 (controlplane01)
- 192.168.56.12 (controlplane02)
- 192.168.56.30 (load balancer)
- 10.96.0.1 (servicio API interno)
- kubernetes, kubernetes.default, etc. (DNS names)

Por qué SANs importan:
- Si kubectl se conecta a 192.168.56.30 (load balancer)
- Pero cert solo tiene 192.168.56.11
- → ERROR: "hostname mismatch"
- → kubectl NO CONFÍA (aunque sea el cert correcto)

Analogía:
- El pasaporte del API server incluye:
  "VÁLIDO EN: 192.168.56.11, 192.168.56.12, 192.168.56.30, 10.96.0.1"
- Si solo dice "VÁLIDO EN: 192.168.56.11"
- Y intentas usarlo en 192.168.56.30
- → Rechazo (aunque sea el mismo servidor)
```

##### 4. Controller Manager & Scheduler Certificates

```
kube-controller-manager.crt + key
kube-scheduler.crt + key

Propósito: Estos componentes se COMUNICAN con API Server
- Dicen: "Yo soy el controller-manager"
- API server verifica el cert
- Luego confía

Similares:
- CN = system:kube-controller-manager
- CN = system:kube-scheduler
- O = grupo del componente

No tienen SANs (porque son CLIENTES, no servidores)
```

##### 5. etcd Server Certificate

```
etcd-server.crt + etcd-server.key

Propósito: etcd prueba su identidad
- Entre etcd peers (etcd1 ↔ etcd2)
- Cuando API server se conecta a etcd

Subject Alternative Names:
- 192.168.56.11 (controlplane01 - etcd está aquí)
- 192.168.56.12 (controlplane02 - etcd está aquí)
- 127.0.0.1 (localhost - para verificaciones locales)

Por qué SANs importan:
- etcd1 (en .11) necesita conectar a etcd2 (en .12)
- El cert debe tener AMBAS IPs
```

##### 6. kubelet Certificate (Workers)

```
kubelet.crt + kubelet.key

Propósito: Cada worker prueba su identidad con API server
- kubelet (en worker) se conecta a API server
- API server verifica: "¿Eres realmente un kubelet?"

Nota: En este lab NO lo generamos aún
(Se genera en Lab 11 - TLS Bootstrapping)
Pero ya sabemos qué es
```

##### 7. kube-proxy Certificate

```
kube-proxy.crt + kube-proxy.key

Propósito: kube-proxy (en workers) se autentica con API server
- Similar al kubelet cert
- kube-proxy necesita información de servicios del API server

Nota: Se usa en workers (node01, node02)
```

##### 8. Service Account Certificate

```
service-account.crt + service-account.key

Propósito: Pods confían en API server
- Cada pod corre un "service account" (una identidad)
- El pod necesita confiar en que el API server es legítimo
- Usa service-account.crt para verificar

Este cert es ESPECIAL:
- Lo usan los PODS (no los componentes)
- El API server lo usa para FIRMAR tokens de service accounts
```

##### 9. API Server → Kubelet Client Certificate

```
apiserver-kubelet-client.crt + apiserver-kubelet-client.key

Propósito: API server se autentica COMO CLIENTE a kubelet
- Normalmente: kubelet ← se conecta a → API server
- Pero a veces: API server → se conecta a → kubelet (para logs, exec)
- Entonces API server necesita un certificado para probar identidad

Rol especial:
- O=system:masters (permiso de admin)
- Le permite hacer cosas que otros clientes no pueden
```

---

### Cómo Los Componentes Usan Certificados

#### Flujo 1: kubectl Conecta a API Server

```
TÚ en Windows:
$ kubectl get pods

kubectl:
1. Lee admin.crt + admin.key (de kubeconfig)
2. Conecta a API server en 192.168.56.30:6443
3. Presenta admin.crt + admin.key
4. Dice: "Soy admin (CN=admin), O=system:masters"

API server:
1. Verifica: ¿Este cert está firmado por MI ca.crt? SÍ
2. Verifica: ¿CN=admin? SÍ
3. Verifica: ¿O=system:masters? SÍ (ADMIN!)
4. CONFÍA en kubectl
5. Responde: "Aquí están todos los pods"

Datos viajan ENCRIPTADOS por TLS ✅
```

#### Flujo 2: kubelet Conecta a API Server

```
kubelet (en node01):
1. Lee kubelet.crt + kubelet.key
2. Conecta a API server en 192.168.56.30:6443
3. Presenta kubelet.crt
4. Dice: "Soy kubelet (CN=system:node:node01)"

API server:
1. Verifica: ¿Firmado por MI ca.crt? SÍ
2. Verifica: ¿CN=system:node:node01? SÍ (es un worker node)
3. CONFÍA
4. Responde: "Aquí están tus pods"

kubelet ahora es AUTÉNTICO ✅
```

#### Flujo 3: etcd Cluster Interno

```
etcd1 (en controlplane01):
1. Lee etcd-server.crt + etcd-server.key
2. Se conecta a etcd2 (en controlplane02)
3. Presenta etcd-server.crt
4. Dice: "Soy etcd en 192.168.56.11"

etcd2:
1. Verifica: ¿Firmado por CA? SÍ
2. Verifica: ¿SANs incluye 192.168.56.11? SÍ
3. CONFÍA en etcd1
4. Intercambian datos ENCRIPTADOS ✅

Resultado: etcd cluster seguro y replicado
```

---

## PHASE 2: Technical Deep Dive

### Los 3 Pasos Para Generar Un Certificado

Cada certificado sigue este flujo:

```
PASO 1: Generar Private Key
        ↓
PASO 2: Crear Certificate Signing Request (CSR)
        ↓
PASO 3: Firmar CSR con CA
        ↓
RESULTADO: certificate.crt (público) + certificate.key (secreto)
```

#### Paso 1: Generate Private Key

```bash
openssl genrsa -out nombre.key 2048
```

**Qué hace:**
- Genera una clave privada RSA de 2048 bits
- La guarda en `nombre.key`
- Esta clave es tu SECRETO (nunca la compartas)

**Analogía:**
- Tu huella dactilar (única, privada)
- Matemáticamente única e imposible de forjar

**Output:**
```
Generating RSA private key, 2048 bit long modulus (2 primes)
.................
```

#### Paso 2: Create Certificate Signing Request (CSR)

```bash
openssl req -new -key nombre.key \
  -subj "/CN=nombre/O=grupo" \
  -out nombre.csr
```

**Qué hace:**
- Lee tu private key (`nombre.key`)
- Crea una SOLICITUD de certificado
- Dice: "Quiero un certificado para: nombre (CN), grupo (O)"
- FIRMA la solicitud CON tu private key (prueba que eres tú)

**Componentes del -subj:**
```
/CN=admin          = Common Name (tu nombre/identidad)
/O=system:masters  = Organization (tu grupo/rol)
```

**Analogía:**
- Vas a la oficina de pasaportes
- Dices: "Yo soy Juan, quiero un pasaporte para viajar"
- Firmas la solicitud (prueba que eres Juan)

**Output:**
```
Certificate Request:
    Subject: CN = admin, O = system:masters
    ...
```

#### Paso 3: Sign CSR with CA

```bash
openssl x509 -req -in nombre.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out nombre.crt \
  -days 1000
```

**Qué hace:**
- Lee el CSR (`nombre.csr`)
- Lee la CA public cert (`ca.crt`)
- Lee la CA private key (`ca.key`)
- LA CA FIRMA EL CSR (crea un certificado válido)
- Guarda el resultado en `nombre.crt`
- Válido por 1000 días

**Quién puede firmar:**
- SOLO la CA (porque solo ella tiene ca.key)
- Sin ca.key, no puedes firmar

**Analogía:**
- Oficina de pasaportes recibe tu solicitud
- Verifica: ¿Es Juan realmente? SÍ
- Usa su sello OFICIAL para firmar
- Te entregan: pasaporte (certificado)
- El sello es IMPOSIBLE de falsificar (matemáticamente)

**Output:**
```
Signature ok
Certificate: Signature ok
Getting CA certificate from ca.crt
...
```

---

### Subject Alternative Names (SANs) - CRÍTICO

#### Por Qué SANs Importan

Algunos certificados (API Server, etcd) son **server certificates** (servidores).

Un servidor puede ser alcanzable por MUCHOS nombres:

```
API Server puede ser accedido por:
- localhost (desde la misma máquina)
- 127.0.0.1 (IP loopback)
- 192.168.56.11 (IP controlplane01)
- 192.168.56.12 (IP controlplane02)
- 192.168.56.30 (IP load balancer)
- kubernetes (DNS name)
- kubernetes.default (DNS name)
- kubernetes.svc (DNS name)
- kubernetes.svc.cluster.local (FQDN)
- 10.96.0.1 (IP del servicio interno)

SI EL CERTIFICADO NO INCLUYE TODOS ESTOS:
→ TLS falla cuando alguien intenta conectar
```

#### Ejemplo de Error Sin SANs Correctos

```
Escenario:
- API server cert solo tiene SANs: [192.168.56.11]
- kubectl intenta conectar a: 192.168.56.30 (load balancer)

Resultado:
$ kubectl get pods
error: x509: certificate is valid for 192.168.56.11, not 192.168.56.30

kubectl RECHAZA el certificado (aunque sea válido)
aunque sea del mismo servidor, porque hostname NO COINCIDE
```

#### Cómo Especificar SANs

OpenSSL no acepta SANs como parámetro. Necesita un archivo `.cnf`:

```bash
cat > openssl.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name

[v3_req]
basicConstraints = critical, CA:FALSE
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
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

**Luego al crear el CSR:**
```bash
openssl req -new -key kube-apiserver.key \
  -subj "/CN=kube-apiserver/O=Kubernetes" \
  -out kube-apiserver.csr \
  -config openssl.cnf     ← Usa el config con SANs
```

**Luego al firmar:**
```bash
openssl x509 -req -in kube-apiserver.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -out kube-apiserver.crt \
  -extensions v3_req \                    ← Aplica las extensiones
  -extfile openssl.cnf \                  ← Del config file
  -days 1000
```

---

## PHASE 3: Ejecución Paso a Paso (Lab 04)

### Pre-Ejecución: Checklist

Antes de empezar, asegúrate que:

- [ ] Estás en controlplane01 (`vagrant ssh controlplane01`)
- [ ] `hostname` retorna `controlplane01`
- [ ] `cfssl` está disponible (si usas cfssl en lugar de openssl)
- [ ] Tienes acceso a `/etc/hosts` (debe estar configurado por Vagrant)
- [ ] Creas una carpeta para los certs: `mkdir -p ~/certs && cd ~/certs`

**Verification:**
```bash
vagrant ssh controlplane01
hostname  # Debe ser: controlplane01
cd ~/
mkdir -p certs
cd certs
pwd       # Debe ser: /home/vagrant/certs
```

---

### Step 1: Set Environment Variables

**Por qué:**
- API Server y etcd certs necesitan las IPs de todos los control planes
- El script leerá `/etc/hosts` para obtenerlas

**Ejecuta:**
```bash
CONTROL01=$(dig +short controlplane01)
CONTROL02=$(dig +short controlplane02)
LOADBALANCER=$(dig +short loadbalancer)
SERVICE_CIDR=10.96.0.0/24
API_SERVICE=$(echo $SERVICE_CIDR | awk 'BEGIN {FS="."} ; { printf("%s.%s.%s.1", $1, $2, $3) }')
```

**Verifica:**
```bash
echo $CONTROL01      # Debe ser: 192.168.56.11
echo $CONTROL02      # Debe ser: 192.168.56.12
echo $LOADBALANCER   # Debe ser: 192.168.56.30
echo $API_SERVICE    # Debe ser: 10.96.0.1
```

**Si algo no aparece:**
```bash
# Probablemente /etc/hosts no está configurado
cat /etc/hosts
# Debe incluir:
# 192.168.56.11 controlplane01
# 192.168.56.12 controlplane02
# etc.

# Si no, agrega manualmente
```

**Documenta en tus notas:**
- [ ] Qué hace cada variable
- [ ] Por qué necesitamos SERVICE_CIDR
- [ ] Por qué esos valores específicos (10.96.0.0/24)

---

### Step 2: Create CA Certificate

**Por qué:**
- Esto es la RAÍZ de confianza
- Todos los otros certs serán firmados por este

**Ejecuta:**
```bash
{
  # Create private key for CA
  openssl genrsa -out ca.key 2048

  # Create CSR using the private key
  openssl req -new -key ca.key \
    -subj "/CN=KUBERNETES-CA/O=Kubernetes" \
    -out ca.csr

  # Self sign the csr using its own private key
  openssl x509 -req -in ca.csr \
    -signkey ca.key \
    -CAcreateserial \
    -out ca.crt \
    -days 1000
}
```

**Qué pasa:**
- Genera `ca.key` (privado)
- Crea `ca.csr` (solicitud auto-firmada)
- Genera `ca.crt` (certificado raíz, auto-firmado)

**Verifica:**
```bash
ls -la ca.*
# Output:
# -rw-r--r-- ca.crt
# -rw------- ca.key
# -rw-r--r-- ca.csr
```

**Documenta:**
- [ ] Cuál es la diferencia entre ca.key (privado) y ca.crt (público)
- [ ] Por qué ca.crt es auto-firmado (no hay otra CA)
- [ ] Por qué ca.key jamás se copia (es el secreto de la CA)

---

### Step 3: Generate Admin Certificate

**Por qué:**
- Este eres TÚ (kubectl)
- Te permite acceso ADMIN al cluster

**Ejecuta:**
```bash
{
  # Generate private key for admin user
  openssl genrsa -out admin.key 2048

  # Generate CSR for admin user
  openssl req -new -key admin.key \
    -subj "/CN=admin/O=system:masters" \
    -out admin.csr

  # Sign certificate for admin user using CA
  openssl x509 -req -in admin.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out admin.crt \
    -days 1000
}
```

**Qué pasa:**
- Genera `admin.key` (tu secreto)
- Crea `admin.csr` (tu solicitud)
- CA FIRMA con su ca.key → genera `admin.crt`

**Importante:**
- `CN=admin` = tu nombre de usuario
- `O=system:masters` = grupo ADMIN (permisos totales)

**Verifica:**
```bash
ls -la admin.*
# Debe haber: admin.key, admin.csr, admin.crt
```

**Documenta:**
- [ ] Qué significa CN=admin
- [ ] Qué significa O=system:masters (por qué "masters"?)
- [ ] Quién verifica este certificado (API server)

---

### Step 4: Generate Controller Manager Certificate

**Por qué:**
- Controller manager corre en controlplane
- Se comunica con API server
- Necesita probar su identidad

**Ejecuta:**
```bash
{
  openssl genrsa -out kube-controller-manager.key 2048

  openssl req -new -key kube-controller-manager.key \
    -subj "/CN=system:kube-controller-manager/O=system:kube-controller-manager" \
    -out kube-controller-manager.csr

  openssl x509 -req -in kube-controller-manager.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out kube-controller-manager.crt \
    -days 1000
}
```

**Documenta:**
- [ ] Por qué controller manager necesita un cert
- [ ] Quién verifica este certificado
- [ ] Por qué CN empieza con "system:"

---

### Step 5: Generate Scheduler Certificate

**Similar a Controller Manager**

**Ejecuta:**
```bash
{
  openssl genrsa -out kube-scheduler.key 2048

  openssl req -new -key kube-scheduler.key \
    -subj "/CN=system:kube-scheduler/O=system:kube-scheduler" \
    -out kube-scheduler.csr

  openssl x509 -req -in kube-scheduler.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out kube-scheduler.crt \
    -days 1000
}
```

---

### Step 6: Generate API Server Certificate (CON SANs)

**Por qué:**
- EL CERTIFICADO MÁS IMPORTANTE
- Todos se conectan al API server
- DEBE tener SANs para todas las maneras de acceso

**Primero, crea openssl.cnf con SANs:**

```bash
cat > openssl.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[v3_req]
basicConstraints = critical, CA:FALSE
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
IP.1 = ${API_SERVICE}
IP.2 = ${CONTROL01}
IP.3 = ${CONTROL02}
IP.4 = ${LOADBALANCER}
IP.5 = 127.0.0.1
EOF
```

**Luego, genera certs:**

```bash
{
  openssl genrsa -out kube-apiserver.key 2048

  openssl req -new -key kube-apiserver.key \
    -subj "/CN=kube-apiserver/O=Kubernetes" \
    -out kube-apiserver.csr \
    -config openssl.cnf

  openssl x509 -req -in kube-apiserver.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out kube-apiserver.crt \
    -extensions v3_req \
    -extfile openssl.cnf \
    -days 1000
}
```

**⚠️ CRÍTICO:**
- Si SANs están mal aquí, kubectl no funciona
- Verifica el openssl.cnf antes de ejecutar

**Documenta:**
- [ ] Por qué API server cert es especial (SANs)
- [ ] Qué sucede si falta un SAN
- [ ] Por qué incluimos 192.168.56.30 (load balancer)

---

### Step 7: Generate API Server → Kubelet Client Certificate

**Por qué:**
- API server a veces necesita conectarse A kubelet (para logs, exec)
- Necesita un cert para probar su identidad como cliente

**Primero, crea openssl-kubelet.cnf:**

```bash
cat > openssl-kubelet.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[v3_req]
basicConstraints = critical, CA:FALSE
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
EOF
```

**Ejecuta:**

```bash
{
  openssl genrsa -out apiserver-kubelet-client.key 2048

  openssl req -new -key apiserver-kubelet-client.key \
    -subj "/CN=kube-apiserver-kubelet-client/O=system:masters" \
    -out apiserver-kubelet-client.csr \
    -config openssl-kubelet.cnf

  openssl x509 -req -in apiserver-kubelet-client.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out apiserver-kubelet-client.crt \
    -extensions v3_req \
    -extfile openssl-kubelet.cnf \
    -days 1000
}
```

---

### Step 8: Generate etcd Server Certificate (CON SANs)

**Por qué:**
- etcd corre en AMBOS control planes (cluster replicado)
- Necesita comunicarse entre etcd peers
- Necesita que API server pueda conectarse

**Crea openssl-etcd.cnf:**

```bash
cat > openssl-etcd.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
IP.1 = ${CONTROL01}
IP.2 = ${CONTROL02}
IP.3 = 127.0.0.1
EOF
```

**Ejecuta:**

```bash
{
  openssl genrsa -out etcd-server.key 2048

  openssl req -new -key etcd-server.key \
    -subj "/CN=etcd-server/O=Kubernetes" \
    -out etcd-server.csr \
    -config openssl-etcd.cnf

  openssl x509 -req -in etcd-server.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out etcd-server.crt \
    -extensions v3_req \
    -extfile openssl-etcd.cnf \
    -days 1000
}
```

---

### Step 9: Generate kube-proxy Certificate

**Por qué:**
- kube-proxy corre en cada worker
- Se comunica con API server
- Necesita probar identidad

**Ejecuta:**

```bash
{
  openssl genrsa -out kube-proxy.key 2048

  openssl req -new -key kube-proxy.key \
    -subj "/CN=system:kube-proxy/O=system:node-proxier" \
    -out kube-proxy.csr

  openssl x509 -req -in kube-proxy.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out kube-proxy.crt \
    -days 1000
}
```

---

### Step 10: Generate Service Account Key Pair

**Por qué:**
- Pods usan service accounts
- API server firma tokens con esta clave
- Pods verifican tokens contra esta clave

**Ejecuta:**

```bash
{
  openssl genrsa -out service-account.key 2048

  openssl req -new -key service-account.key \
    -subj "/CN=service-accounts/O=Kubernetes" \
    -out service-account.csr

  openssl x509 -req -in service-account.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out service-account.crt \
    -days 1000
}
```

---

### Step 11: Verify All Certificates

**Por qué:**
- Asegurar que todos los certs están bien ANTES de copiarlos
- `cert_verify.sh` te dice exactamente qué falta o está mal

**Descarga cert_verify.sh:**

```bash
# Desde el repo:
# Si estás en la carpeta del repo:
cd ~/kubernetes-the-hard-way

# Busca cert_verify.sh (está en apple-silicon/scripts/ o similar)
find . -name "cert_verify.sh"

# Si la encuentras:
cp apple-silicon/scripts/cert_verify.sh ~/certs/
chmod +x ~/certs/cert_verify.sh

# O descárgala desde repo:
curl -L https://raw.githubusercontent.com/kelseyhightower/kubernetes-the-hard-way/master/apple-silicon/scripts/cert_verify.sh -o ~/certs/cert_verify.sh
chmod +x ~/certs/cert_verify.sh
```

**Ejecuta verificación:**

```bash
cd ~/certs
./cert_verify.sh 1
```

**Output esperado:**
```
The selected option is 1, proceeding the certificate verification of Master node
ca cert and key found, verifying the authenticity
ca cert and key are correct
kube-apiserver cert and key found, verifying the authenticity
kube-apiserver cert and key are correct
kube-controller-manager cert and key found, verifying the authenticity
kube-controller-manager cert and key are correct
...
[todos los certs dicen "are correct"]
```

**Si hay ERRORES:**
- Lee el error cuidadosamente
- Probablemente un SAN faltante o cert no firmado bien
- Regenera ese certificado específico

**Documenta:**
- [ ] Todos los certs pasaron verificación
- [ ] Screenshot o output de `cert_verify.sh 1`

---

### Step 12: Distribute Certificates

**Por qué:**
- Cada componente necesita su cert
- Control planes necesitan TODOS los certs
- Workers necesitan solo ca.crt, kube-proxy.crt, kube-proxy.key (ahora)

**Copia a control planes:**

```bash
for instance in controlplane01 controlplane02; do
  scp -o StrictHostKeyChecking=no \
    ca.crt ca.key \
    kube-apiserver.key kube-apiserver.crt \
    apiserver-kubelet-client.crt apiserver-kubelet-client.key \
    service-account.key service-account.crt \
    etcd-server.key etcd-server.crt \
    kube-controller-manager.key kube-controller-manager.crt \
    kube-scheduler.key kube-scheduler.crt \
    ${instance}:~/
done
```

**Copia a workers:**

```bash
for instance in node01 node02; do
  scp -o StrictHostKeyChecking=no \
    ca.crt \
    kube-proxy.crt kube-proxy.key \
    ${instance}:~/
done
```

**Verifica que llegó:**

```bash
# En controlplane01:
ls ~/*.crt ~/*.key | wc -l  # Debe ser: > 10

# En node01:
ssh node01 "ls ~/*.crt ~/*.key"
# Debe mostrar: ca.crt, kube-proxy.crt, kube-proxy.key
```

**Documenta:**
- [ ] Todos los certs fueron copiados
- [ ] No copiaste ca.key a workers (SECRETO)
- [ ] Verificaste que llegaron

---

## Post-Execution: Documentation

### Cosas Que Debes Entender Ahora

Después de completar Lab 04, deberías poder responder:

1. **¿Qué es una CA y por qué controlplane01 es la CA?**
2. **¿Cuáles son los 9 certificados y qué componente usa cada uno?**
3. **¿Por qué algunos certs tienen SANs y otros no?**
4. **¿Qué pasa si omito un SAN del API server cert?**
5. **¿Por qué ca.key NUNCA se copia a otros nodos?**
6. **¿Cuál es el flujo de verificación cuando kubectl conecta a API server?**

### Checklist Final

- [ ] Todos los 9 certificados generados
- [ ] `cert_verify.sh 1` dice "all correct"
- [ ] Certs distribuidos a todos los nodos
- [ ] Documentado en tus notas: qué hace cada cert
- [ ] Entiendes los 3 pasos: genrsa → req → x509
- [ ] Entiendes por qué SANs importan
- [ ] No copiaste ca.key a workers

---

## Próximo Lab: Lab 05 (Kubeconfigs)

Los certificados que generaste AHORA se usan en Lab 05 para crear **kubeconfigs**:

```
Kubeconfig = Configuración de kubectl
├─ Dónde está el API server (URL)
├─ Qué certificado usar (admin.crt + admin.key)
├─ Qué CA confiar (ca.crt)
└─ Contextos (qué cluster/usuario/namespace)

Ejemplo:
~/.kube/config
├─ clusters:
│  ├─ name: kubernetes
│  └─ certificate-authority: ca.crt
├─ users:
│  ├─ name: admin
│  └─ client-certificate: admin.crt
│     client-key: admin.key
└─ context: admin → kubernetes (quién connecta a dónde)
```

Lab 05 es corto porque ya tienes todos los certs listos. 🚀

---

**¿Listo para empezar?** 🔐

Avísame cuando termines cada step y estaré aquí para debuggear si algo falla.
