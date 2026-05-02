# TCP/TLS: Cómo Funcionan los Certificados - Perspectiva Técnica

## Las Capas OSI Involucradas

```
┌────────────────────────────────────────────────────┐
│ LAYER 7: Application (kubectl, controller-manager) │
├────────────────────────────────────────────────────┤
│ LAYER 6: Presentation (encoding, compression)      │
├────────────────────────────────────────────────────┤
│ LAYER 5: Session (TLS Handshake AQUÍ)              │
│          ├─ Certificate exchange                   │
│          ├─ Key agreement                          │
│          └─ Cipher suite negotiation               │
├────────────────────────────────────────────────────┤
│ LAYER 4: Transport (TCP - 6443 para HTTPS)         │
│          ├─ SYN/ACK handshake                     │
│          ├─ Connection established                 │
│          └─ Stream-based communication             │
├────────────────────────────────────────────────────┤
│ LAYER 3: Network (IP routing, TCP packets)         │
├────────────────────────────────────────────────────┤
│ LAYER 2: Data Link (Ethernet frames)               │
├────────────────────────────────────────────────────┤
│ LAYER 1: Physical (cables, signals)                │
└────────────────────────────────────────────────────┘

LA MAGIA PASA EN LAYER 5: TLS Handshake
(Después de que TCP ya esté conectado - LAYER 4)
```

---

## Ejemplo 1: Controller-Manager Se Conecta a API Server

### Preparación (Antes de la Conexión)

```
┌─ controlplane01 ──────────────────────────┐
│ kube-controller-manager:                  │
│ ├─ kube-controller-manager.key (private)  │
│ ├─ kube-controller-manager.crt (cert)     │
│ └─ ca.crt (para verificar el API server)  │
│                                           │
│ Lee config: "Conecta a 10.96.0.1:6443"   │
└───────────────────────────────────────────┘
```

### TCP Connection (LAYER 4)

```
1. CREACIÓN DE CONEXIÓN

controller-manager:
$ socket = socket(AF_INET, SOCK_STREAM)
$ socket.connect("10.96.0.1", 6443)

TCP Three-Way Handshake:
├─ Client → Server: SYN (seq=x)
├─ Server → Client: SYN-ACK (seq=y, ack=x+1)
└─ Client → Server: ACK (seq=x+1, ack=y+1)

RESULTADO: Conexión TCP ESTABLECIDA (sin cifrar)
En este punto aún pueden interceptar datos
```

### TLS Handshake (LAYER 5) - El Punto Crítico

```
2. TLS HANDSHAKE PHASE 1: ClientHello

controller-manager envía:
┌─ TLS ClientHello ─────────────────────────┐
│ TLS Version: 1.2                          │
│ Cipher Suites Soportadas:                │
│ ├─ ECDHE_RSA_WITH_AES_256_GCM_SHA384     │
│ ├─ ECDHE_RSA_WITH_AES_128_GCM_SHA256     │
│ └─ ... (lista de lo que soporto)         │
│ Compression Methods: none                 │
│ Extensions: SNI = "kubernetes"  ← IMPORTANTE
└───────────────────────────────────────────┘

La extensión SNI (Server Name Indication) es CRÍTICA:
└─ "Quiero conectar al servidor: kubernetes"
└─ El servidor usará esto para elegir qué certificado enviar
```

### TLS Handshake (LAYER 5) - PHASE 2: ServerHello + Certificate

```
3. API SERVER RESPONDE CON ServerHello + Certificate

API server envía:
┌─ TLS ServerHello ─────────────────────────┐
│ TLS Version: 1.2                          │
│ Selected Cipher: ECDHE_RSA_WITH_AES_256_GCM_SHA384
│ Compression: none                         │
└───────────────────────────────────────────┘

┌─ Server Certificate ──────────────────────┐
│ AQUÍ VIENE EL CERTIFICADO REAL            │
│                                           │
│ Certificate: kube-apiserver.crt           │
│ ├─ Subject: CN=kube-apiserver, O=Kubernetes
│ ├─ Issuer: CN=KUBERNETES-CA, O=Kubernetes
│ ├─ Public Key: [RSA 2048 bits]            │
│ ├─ Signature por CA: [firma digital]      │
│ └─ SubjectAltNames:                       │
│    ├─ kubernetes                          │
│    ├─ kubernetes.default                  │
│    ├─ kubernetes.default.svc.cluster.local
│    ├─ IP: 10.96.0.1  ← IMPORTANTE        │
│    ├─ IP: 192.168.56.11                   │
│    ├─ IP: 192.168.56.12                   │
│    ├─ IP: 192.168.56.30                   │
│    └─ IP: 127.0.0.1                       │
└───────────────────────────────────────────┘

┌─ API Server envia también: ServerKeyExchange
│ (parámetros para ECDHE key agreement)
└─────────────────────────────────────────────
```

### TLS Handshake (LAYER 5) - PHASE 3: VALIDACIÓN en Controller-Manager

```
4. CONTROLLER-MANAGER VALIDA EL CERTIFICADO

Este es el PUNTO CRÍTICO donde ocurre la magia:

┌─ Controller-manager verifica ──────────────────┐
│                                               │
│ PASO 1: ¿Está el certificado firmado por CA? │
│ ├─ Tengo: ca.crt (clave pública de CA)       │
│ ├─ Leo: firma digital en el cert del API     │
│ ├─ Uso ca.crt para verificar la firma        │
│ └─ ✓ SÍ, está correctamente firmado          │
│                                               │
│ PASO 2: ¿Ha expirado?                        │
│ ├─ Leo: validFrom = 2025-01-01               │
│ ├─ Leo: validUntil = 2028-01-01             │
│ ├─ Fecha actual: 2026-02-07                  │
│ └─ ✓ NO, sigue válido                        │
│                                               │
│ PASO 3: ¿HOSTNAME MATCHING (CRÍTICO PARA SANs)
│ ├─ ¿A qué IP intenté conectar?               │
│ │  10.96.0.1                                 │
│ ├─ ¿A qué hostname mapea? (si usé DNS)       │
│ │  kubernetes.default                        │
│ ├─ Leo SANs del certificado:                 │
│ │  [kubernetes, kubernetes.default,         │
│ │   kubernetes.default.svc.cluster.local,   │
│ │   IP:10.96.0.1, IP:192.168.56.11, ...]   │
│ ├─ Busco: ¿Está 10.96.0.1 en la lista?      │
│ └─ ✓ SÍ, lo encontré!                        │
│                                               │
│ PASO 4: ¿Otros checks?                       │
│ ├─ ¿Basic constraints CA:FALSE? (no es CA)   │
│ └─ ✓ OK                                      │
│                                               │
│ RESULTADO FINAL: ✅ CERTIFICADO VÁLIDO       │
└───────────────────────────────────────────────┘

Si CUALQUIERA de estos checks falla:
└─ TLS CONNECTION ABORTED
└─ Error: "x509: certificate verification failed"
└─ Controller-manager NO se conecta
```

### TLS Handshake (LAYER 5) - PHASE 4: ClientKeyExchange + ChangeCipherSpec

```
5. CONTROLLER-MANAGER ENVÍA ClientKeyExchange

Controller-manager:
┌─ ClientKeyExchange ────────────────────────┐
│ [MaterialPara ECDHE key agreement]         │
│ (parámetros que solo API server puede usar)
└────────────────────────────────────────────┘

┌─ ChangeCipherSpec ─────────────────────────┐
│ "A partir de ahora, cifra TODOS los datos" │
│ ├─ Ambos derivan: shared secret             │
│ ├─ Ambos derivan: session keys              │
│ └─ Algoritmo: ECDHE_RSA_WITH_AES_256_GCM   │
└────────────────────────────────────────────┘

┌─ Finished ─────────────────────────────────┐
│ HMAC de todo el handshake                  │
│ (prueba de que ambos llegamos al mismo key)|
└────────────────────────────────────────────┘
```

### TLS Handshake (LAYER 5) - PHASE 5: Respuesta del API Server

```
6. API SERVER RESPONDE CON ChangeCipherSpec + Finished

API server:
┌─ ChangeCipherSpec ─────────────────────────┐
│ "A partir de ahora, cifro TODOS los datos" │
└────────────────────────────────────────────┘

┌─ Finished ─────────────────────────────────┐
│ HMAC de todo el handshake                  │
└────────────────────────────────────────────┘

RESULTADO: TLS TUNNEL ESTABLECIDO ✅
```

### Application Data (LAYER 7) - Ahora Seguro

```
7. AHORA CONTROLLER-MANAGER PUEDE ENVIAR REQUESTS

Controller-manager envía (CIFRADO por TLS):
┌─ HTTP/2 GET Request ──────────────────────┐
│ GET /api/v1/replicasets HTTP/2            │
│ Authorization: Bearer <token>              │
│ ...                                        │
│                                            │
│ Encriptado con:                           │
│ ├─ symmetric key (AES-256)                │
│ ├─ HMAC para autenticidad                 │
│ └─ solo API server puede descifrarlo      │
└────────────────────────────────────────────┘

API server recibe (DESENCRIPTA):
├─ Verifica HMAC (¿datos intactos?)
├─ Desencripta con su symmetric key
├─ Lee: GET /api/v1/replicasets
├─ Ahora VERIFICA AUTORIZACIÓN:
│  ├─ Lee del certificado: CN=system:kube-controller-manager
│  ├─ Consulta RBAC: ¿Puede leer replicasets?
│  └─ SÍ, lo permite
└─ Responde (TAMBIÉN CIFRADO POR TLS)
```

---

## ¿QUÉ PASA SIN SANs CORRECTOS?

### Escenario: Falta 10.96.0.1 en SANs

```
API server cert tiene SANs: [kubernetes, kubernetes.default, 
                             192.168.56.11, 192.168.56.12, 192.168.56.30,
                             127.0.0.1]

PERO FALTA: 10.96.0.1

Controller-manager intenta conectar a: 10.96.0.1:6443

TLS Handshake PHASE 3 (Validación):

┌─ Controller-manager valida ─────────────────┐
│ PASO 3: HOSTNAME MATCHING                   │
│ ├─ ¿A qué IP intenté conectar?              │
│ │  10.96.0.1                                │
│ ├─ Leo SANs del certificado:                │
│ │  [kubernetes, kubernetes.default,         │
│ │   192.168.56.11, 192.168.56.12,          │
│ │   192.168.56.30, 127.0.0.1]              │
│ ├─ Busco: ¿Está 10.96.0.1 en la lista?     │
│ └─ ❌ NO, NO lo encuentro!                  │
│                                             │
│ RESULTADO:                                  │
│ x509: certificate is valid for               │
│ 192.168.56.11, not 10.96.0.1               │
│                                             │
│ TLS CONNECTION ABORTED ❌                   │
│                                             │
│ AUNQUE:                                     │
│ ✅ Certificado está firmado correctamente  │
│ ✅ No ha expirado                           │
│ ✅ Es del API server correcto               │
│ ❌ PERO hostname NO matcha SANs             │
└─────────────────────────────────────────────┘

RESULTADO: Controller-manager no puede conectar
AUNQUE el certificado sea VÁLIDO matemáticamente
```

---

## Ejemplo 2: API Server → Kubelet (Logs/Exec)

### ¿Por Qué Necesita Otro Certificado?

```
ESCENARIO NORMAL:
kubelet (client) → API server (server)
└─ kubelet.crt: "Soy kubelet"

ESCENARIO ESPECIAL:
kubectl logs <pod>
├─ kubectl → API server: "Dame logs del pod"
│  (usando admin.crt)
├─ API server → kubelet: "Conseguime los logs"
│  (aquí API server actúa como CLIENT)
└─ Kubelet verifica: "¿Quién eres?" (¿De verdad eres API server?)

PROBLEMA: ¿Cómo kubelet sabe que es realmente el API server?
SOLUCIÓN: apiserver-kubelet-client.crt (API server como cliente)
```

### TCP + TLS Handshake: API Server → Kubelet

```
API server inicia conexión a kubelet:10250:

┌─ LAYER 4: TCP Three-Way Handshake ────────────┐
│ API → Kubelet: SYN
│ Kubelet → API: SYN-ACK
│ API → Kubelet: ACK
│ ✓ Conexión TCP establecida
└───────────────────────────────────────────────┘

┌─ LAYER 5: TLS Handshake ──────────────────────┐
│ TLS ClientHello:                              │
│ ├─ API server envía su soporte de cipher suites
│ └─ API server: "Soy: kube-apiserver-kubelet-client"
│                                               │
│ Kubelet responde:                            │
│ ├─ Selecciona cipher suite                    │
│ └─ Envía kubelet certificate (su CA.crt)     │
│                                               │
│ API server valida kubelet cert:              │
│ ├─ ¿Está firmado por MI ca.crt? ✓ SÍ        │
│ ├─ ¿Es realmente un kubelet? ✓ SÍ (CN=...)  │
│ └─ API server CONFÍA                         │
│                                               │
│ API server presenta su cert:                 │
│ apiserver-kubelet-client.crt:                │
│ ├─ CN=kube-apiserver-kubelet-client          │
│ ├─ O=system:masters                          │
│ ├─ Firmado por: CA                           │
│ └─ SIN SANs (es cliente, no server)          │
│                                               │
│ Kubelet valida API server cert:              │
│ ├─ ¿Está firmado por MI ca.crt? ✓ SÍ        │
│ ├─ ¿O=system:masters? ✓ SÍ (admin!)         │
│ └─ Kubelet CONFÍA                            │
│                                               │
│ ✓✓ Mutual TLS (mTLS) establecido             │
└───────────────────────────────────────────────┘
```

---

## ¿Cuándo Necesitas SANs?

### Regla Técnica de TLS

```
En TLS 1.2+, hostname verification es OBLIGATORIO en clientes seguros.

Esto significa que OpenSSL/Go/Python:
├─ Lee el hostname al cual intentaste conectar
├─ Lee SANs del certificado
├─ Busca si hostname/IP está en SANs
└─ Si NO está, rechaza (aunque cert sea válido)

```

### Server Certificate Necesita SANs Porque:

```
┌─ Acción 1: Alguien se conecta a IP 192.168.56.30 ────┐
│ Cliente verifica:                                     │
│ ├─ "¿Estoy conectado a 192.168.56.30?"              │
│ ├─ "¿192.168.56.30 está en SANs?" SÍ ✓             │
│ └─ Confía                                            │
└──────────────────────────────────────────────────────┘

┌─ Acción 2: Otro se conecta a DNS kubernetes ────────┐
│ Cliente verifica:                                    │
│ ├─ "¿Me conecté a kubernetes?"                     │
│ ├─ "¿kubernetes está en SANs?" SÍ ✓                │
│ └─ Confía                                           │
└────────────────────────────────────────────────────┘

┌─ Acción 3: Otro se conecta a 10.96.0.1 ──────────┐
│ Cliente verifica:                                  │
│ ├─ "¿Me conecté a 10.96.0.1?"                    │
│ ├─ "¿10.96.0.1 está en SANs?" SÍ ✓              │
│ └─ Confía                                         │
└────────────────────────────────────────────────────┘

Si alguien falta:
└─ TLS rechaza la conexión (hostname mismatch)
```

### Client Certificate NO Necesita SANs Porque:

```
┌─ Controller-manager conecta A api-server ─────────┐
│ Controller-manager sabe dónde conecta:            │
│ ├─ "Voy al API server en 10.96.0.1:6443"         │
│ ├─ Controller-manager INICIA la conexión          │
│ ├─ No hay "hostname mismatch"                     │
│ │  (él sabe a dónde va)                          │
│ └─ Pero PRUEBA identidad: "Soy controller-manager"
└──────────────────────────────────────────────────┘

El servidor (API) no necesita buscar en SANs del cliente.
El servidor simplemente verifica:
├─ ¿Está el certificado firmado? ✓
├─ ¿CN=system:kube-controller-manager? ✓
└─ "OK, eres controller-manager legítimo"
```

---

## Las 3 Capas de Seguridad

### Capa 1: TLS Transport Security (LAYER 4-5)

```
TCP + TLS proporciona:

1. CONFIDENTIALITY (Confidencialidad)
   ├─ Datos ENCRIPTADOS
   ├─ Solo quien tiene la clave symmetric puede leer
   └─ Algoritmo: AES-256-GCM

2. INTEGRITY (Integridad)
   ├─ HMAC de datos
   ├─ Si datos se modifican, HMAC falla
   └─ Detección de tampering

3. CERTIFICATE VERIFICATION (Verificación de Identidad)
   ├─ ¿Quién eres realmente?
   ├─ Validación de firma digital
   └─ SANs para hostname matching
```

### Capa 2: Authentication (LAYER 7 - Application)

```
Una vez TLS está establecido:

API server puede verificar:
├─ CN del certificado: "Eres controller-manager"
├─ O del certificado: "Perteneces a system:kube-controller-manager"
└─ Resultado: Identidad confirmada
```

### Capa 3: Authorization - RBAC (LAYER 7 - Application)

```
Una vez autenticado, se pregunta: ¿Qué puedes hacer?

API server consulta RBAC:
├─ Usuario: controller-manager
├─ Grupo: system:kube-controller-manager
├─ Pregunta: ¿Puede leer replicasets?
├─ RBAC responde: "Sí, tiene bindings de role"
└─ Permitido ✓
```

---

## Resumen: Por Qué Necesitas Diferentes Certificados

### admin.crt (CLIENT)

```
TCP/TLS Handshake:

1. kubectl conecta a api-server:6443
2. TLS: "Hola, aquí está mi admin.crt"
3. API Server valida:
   ├─ ¿Está firmado? ✓
   ├─ ¿CN=admin? ✓
   └─ ✓ Confía
4. Conexión encriptada establecida ✓

NO necesita SANs porque:
└─ kubectl sabe dónde conecta (hostname fijo)
└─ No hay ambigüedad de dirección
```

### kube-apiserver.crt (SERVER)

```
TCP/TLS Handshake:

1. Alguien conecta a api-server:6443 (desde múltiples IPs)
2. TLS: "Aquí está mi certificado"
3. Cliente valida:
   ├─ ¿Está firmado? ✓
   ├─ ¿A qué IP intenté conectar? 10.96.0.1
   ├─ ¿Está en SANs? ¿VEAMOS...
   │  SANs = [kubernetes, ..., 10.96.0.1, ...]
   │  ✓ SÍ, está
   └─ ✓ Confía
4. Conexión encriptada establecida ✓

NECESITA SANs porque:
└─ Es atacado desde MÚLTIPLES IPs/hostnames
└─ Cada cliente verifica que su dirección de conexión esté en SANs
└─ Sin SANs: hostname mismatch → conexión rechazada
```

### apiserver-kubelet-client.crt (CLIENT)

```
TCP/TLS Handshake:

1. API server conecta a kubelet:10250
2. TLS: "Aquí está mi apiserver-kubelet-client.crt"
3. Kubelet valida:
   ├─ ¿Está firmado? ✓
   ├─ ¿O=system:masters? ✓ (tiene permisos)
   └─ ✓ Confía
4. Conexión encriptada establecida ✓

NO necesita SANs porque:
└─ API server INICIA la conexión (sabe a dónde va)
└─ Kubelet no necesita buscar en SANs del cliente
└─ Solo verifica identidad: "¿Eres API server admin?"
```

---

## Visualización: El Flujo Completo de Bytes

### Controller-Manager → API Server

```
BYTE 0:  TCP SYN packet → API server:6443

BYTE X:  ┌─ TLS ClientHello (plain) ───────────┐
         │ Version: TLS 1.2                    │
         │ Cipher Suites: [...]                │
         │ SNI: "kubernetes"                   │
         └─────────────────────────────────────┘

BYTE Y:  ┌─ TLS ServerHello (plain) ───────────┐
         │ Selected Cipher: AES-256-GCM        │
         │ Version: TLS 1.2                    │
         └─────────────────────────────────────┘

BYTE Z:  ┌─ Certificate (plain) ───────────────┐
         │ kube-apiserver.crt                  │
         │ ├─ Subject: CN=kube-apiserver       │
         │ ├─ SANs: [10.96.0.1, ...]          │
         │ ├─ Signature: [firma digital]       │
         │ └─ Public Key: [RSA 2048]          │
         └─────────────────────────────────────┘

BYTE W:  ┌─ ClientKeyExchange (plain) ‾───────┐
         │ ECDHE parámetros                    │
         └─────────────────────────────────────┘

BYTE V:  ┌─ ChangeCipherSpec (plain) ─────────┐
         │ "Ahora usamos AES-256 para todo"   │
         └─────────────────────────────────────┘

BYTE U:  ┌─ Finished (ENCRIPTADO) ────────────┐
         │ HMAC de handshake (con AES-256)    │
         └─────────────────────────────────────┘

BYTE T:  ┌─ Application Data (ENCRIPTADO) ───┐
         │ GET /api/v1/replicasets            │
         │ (completamente cifrado)            │
         └─────────────────────────────────────┘

HASTA AQUÍ: TODO ENCRIPTADO, AUTENTICADO, VERIFICADO ✓
```

---

## Testing Real: Qué Pasa Si Falta un SAN

### Intenta conectar a API server desde tu laptop

```bash
# Tienes ca.crt, admin.crt localmente
# El API server corre en controlplane01 (192.168.56.11)

# Intento 1: Conectar vía IP
$ openssl s_client -connect 192.168.56.11:6443 \
  -cert admin.crt -key admin.key -CAfile ca.crt

# Resultado:
Verify return code: 0 (ok)
✓ FUNCIONA (192.168.56.11 está en SANs)

# Intento 2: Conectar vía load balancer (192.168.56.30)
$ openssl s_client -connect 192.168.56.30:6443 \
  -cert admin.crt -key admin.key -CAfile ca.crt

# Resultado (si .30 NO está en SANs):
Verify return code: 19 (self signed certificate)
error 19 at 0 depth lookup: self signed certificate
❌ FALLA (392.168.56.30 NO está en SANs)

# Intento 3: Conectar vía hostname
$ openssl s_client -connect kubernetes:6443 \
  -cert admin.crt -key admin.key -CAfile ca.crt

# Resultado (si "kubernetes" está en SANs):
Verify return code: 0 (ok)
✓ FUNCIONA ("kubernetes" está en SANs)
```

---

## Resumen Técnico

```
┌─ CLIENTE (Controller-Manager) ────────────────┐
│ 1. Inicia TCP a 10.96.0.1:6443               │
│ 2. TCP established ✓                         │
│ 3. Envía TLS ClientHello                     │
│ 4. Recibe TLS ServerHello + Certificate      │
│ 5. Verifica:                                  │
│    ├─ Firma digital (con ca.crt)             │
│    ├─ SANs (si fuera hostname basado)        │
│    └─ Expiración                             │
│ 6. Envía ClientKeyExchange + Finished        │
│ 7. TLS Tunnel ESTABLECIDO                    │
│ 8. Envía: GET /api/v1/replicasets (CIFRADO) │
│ 9. Recibe: lista de replicasets (CIFRADO)    │
│ 10. Cierra conexión (TLS + TCP)              │
└───────────────────────────────────────────────┘

┌─ SERVIDOR (API Server) ───────────────────────┐
│ 1. Escucha en :6443                          │
│ 2. Acepta TCP from controller-manager         │
│ 3. Recibe TLS ClientHello                    │
│ 4. Envía TLS ServerHello + Certificate       │
│    (kube-apiserver.crt CON SANs)             │
│ 5. Recibe ClientKeyExchange + Finished       │
│ 6. TLS Tunnel ESTABLECIDO                    │
│ 7. Recibe: GET /api/v1/replicasets (CIFRADO)│
│ 8. Desencripta (usando symmetric key)        │
│ 9. Verifica HMAC (¿datos intactos?)         │
│ 10. Lee certificado del cliente:             │
│     ├─ CN=system:kube-controller-manager    │
│     └─ Consulta RBAC: OK, puede leer         │
│ 11. Ejecuta solicitud                        │
│ 12. Envía respuesta (CIFRADO)                │
└───────────────────────────────────────────────┘
```

**El concepto clave:** SANs no es un cortafuegos, es parte del **TLS hostname verification** - el cliente verifica que el nombre/IP al que se conecta esté en SANs del certificado. Sin SANs correctos, TLS rechaza aunque el certificado sea matemáticamente válido.

