# Service Accounts - Identidad de Pods

## ¿Qué es una Service Account?

**Definición simple:**
- Service Account (SA) = Identidad para Pods
- Es la forma en que Kubernetes **autentica a los pods** ante el API server
- Similar a usuarios en Linux, pero para Pods

**Diferencia importante:**
```
Certificados:
├─ Los usan COMPONENTES (kubelet, api-server, controller-manager)
└─ Son pares público-privado en archivos

Service Accounts:
├─ Los usan PODS (aplicaciones corriendo en el cluster)
└─ Se distribuyen como TOKENS (JWT) montados en el pod
```

---

## El Problema: ¿Cómo Se Autentica un Pod?

### Escenario

```
Pod A (nginx) corriendo en node01
- Necesita leer un secret del API server
- Necesita lister servicios disponibles
- Problema: ¿Cómo prueba que es un pod legítimo?

Sin Service Account:
❌ API server: "¿Quién eres? No tengo forma de verificar"
💥 Rechaza la solicitud

Con Service Account:
✅ Pod: "Soy pod-nginx, con token JWT firmado por ti"
✅ API server: "Verifico el token, confío en ti"
✅ Acceso permitido
```

---

## Flujo Completo: Service Account + Tokens

### Paso 1: Pod Arranca

```
Kubernetes:
1. Crea un Pod en el namespace default
2. Asigna Default Service Account (default/default)
3. Genera un JWT token para ese pod
4. Monta el token en el pod como archivo: /var/run/secrets/kubernetes.io/serviceaccount/token
```

**Archivo en el pod:**
```bash
# Dentro del pod:
$ cat /var/run/secrets/kubernetes.io/serviceaccount/token

eyJhbGciOiJSUzI1NiIsImtpZCI6IjBqMUZNLVFlQVIzLWFTSUJtM1KPQ...
# (JWT token firmado por API server)
```

### Paso 2: Pod Conecta al API server

```
Pod:
1. Lee el token de /var/run/secrets/kubernetes.io/serviceaccount/token
2. Conecta a API server en https://kubernetes.default.svc
3. Envía el token en el header: "Authorization: Bearer <token>"

API server:
1. Recibe el token JWT
2. Verifica la firma usando service-account.crt
   (La clave privada correspondiente es service-account.key)
3. Comprueba: ¿Es válido? ¿No está expirado?
4. Decodifica el token y obtiene: "namespace=default, serviceaccount=default"
5. CONFÍA y permite el acceso (si los permisos lo permiten)

Ejemplo de JWT decodificado:
{
  "iss": "kubernetes/serviceaccount",
  "kubernetes.io/serviceaccount/namespace": "default",
  "kubernetes.io/serviceaccount/name": "default",
  "kubernetes.io/serviceaccount/uid": "abc123",
  ...
}
```

### Paso 3: Solicitud Autenticada

```
Pod a API server:
GET /api/v1/namespaces/default/secrets
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...

API server verifica:
✅ ¿Token es válido? SÍ
✅ ¿Está firmado por mi service-account.key? SÍ
✅ ¿Qué permisos tiene? (RBAC) → default/default en namespace default
✅ ¿Puede leer secrets? Depende del RBAC
→ Responde con los secrets (o rechaza si no tiene permisos)
```

---

## Service Account vs Certificados

### Componentes (kubelet, api-server, etc.)

```
kubelet:
├─ Usa: kubelet.crt + kubelet.key (archivos en el filesystem)
├─ Conecta a: API server
├─ Dice: "Soy kubelet firmado por CA"
└─ API server verifica: ¿CA lo firmó? SÍ → Confía
```

### Pods (aplicaciones)

```
Pod (aplicación):
├─ Usa: JWT token (montado como secret)
├─ Conecta a: API server
├─ Dice: "Soy pod-nginx con este JWT token"
└─ API server verifica: ¿Mi service-account.key lo firmó? SÍ → Confía
```

**¿Por qué diferente?**
- Certificados = Para componentes del cluster (configuración estática)
- Tokens = Para pods (dinámicos, se crean/destruyen constantemente)

---

## El Certificado: service-account.crt + service-account.key

### Rol en el Cluster

| Aspecto | Detalle |
|---------|---------|
| **Qué es** | Par de claves para firmar JWT tokens de pods |
| **Dónde está** | controlplane (en el API server) |
| **service-account.key** | Privada - API server la usa para FIRMAR tokens |
| **service-account.crt** | Pública - Distribuida a todos (pods la verifican) |
| **Proceso** | API server firma → Pod verifica con este cert |

### Flujo de Firma y Verificación

```
En el API server (genera el token):
1. Pod nuevo arranca en namespace default
2. API server lee service-account.key (privada)
3. Firma este contenido:
   {
     "namespace": "default",
     "serviceaccount": "default",
     "uid": "abc123"
   }
4. Resultado: JWT token (eyJhbGciOiJSUzI1NiIs...)

En el Pod (verifica el token):
1. Pod necesita verificar que el token es legítimo
2. Lee /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
   (Esta es service-account.crt distribuida a TODOS los nodos)
3. Descodifica el token y verifica la firma
4. "¿Fue firmado con la clave privada de esta CA? SÍ → Confío"
```

---

## Analogía: Firma de Documentos

```
Oficina de migraciones (API server):

1. Juan llega como "nuevo ciudadano" (pod arranca)
2. Oficina crea un pasaporte JWT:
   - Datos: nombre=Juan, ciudadanía=default, uid=abc123
   - Firma CON SU CLAVE PRIVADA (service-account.key)
   - Resultado: Pasaporte firmado

3. Juan viaja al extranjero (pod conecta a API server)
4. Muestra el pasaporte (token)
5. Extranjero verifica:
   - "¿Este pasaporte está firmado por Oficina de Migraciones?"
   - Busca la firma PÚBLICA (service-account.crt)
   - ✅ "Sí, es legítimo"

Sin esta firma:
- Cualquiera podría forjar un pasaporte
- Extranjero rechazaría a Juan: "No confío"
```

---

## Aclaración: Service Account y Cloud Providers

### Lo que Mencionaste (AWS + IRSA)

**Sí, es correcto. Hay dos niveles:**

```
1. NATIVO K8s:
   Pod → (usa JWT token) → API server
   
2. Con IRSA (AWS):
   Pod → (usa JWT token) → API server
        ↓ (el token se convierte en)
   Pod → (asume rol AWS) → AWS API
        ↓
   Pod → Acceso a S3, DynamoDB, etc.

El flow IRSA es:
- Pod obtiene JWT token de K8s
- Pod envía el JWT al OIDC provider de AWS
- AWS verifica el JWT (lo firmó K8s)
- AWS emite temporal AWS credentials
- Pod usa esas credentials para acceder a AWS
```

**En Lab 04 (este):**
- Solo generamos el service-account.crt + key
- El JWT se genera dinámicamente cuando el pod arranca
- No nos preocupamos por AWS (eso es una integración posterior)

---

## Default Service Account

### Qué hace K8s por defecto

```
Cuando se crea un namespace:
1. K8s crea automáticamente una SA llamada "default"
2. K8s crea un secret con el JWT token
3. Cuando un pod arranca SIN especificar SA:
   - Se asigna automáticamente a "default"
   - Se monta el token en /var/run/secrets
```

**Ejemplo:**
```yaml
# Pod simple (sin especificar SA)
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:latest
    
# Kubernetes automáticamente:
# - Asigna SA: default
# - Monta el token en: /var/run/secrets/kubernetes.io/serviceaccount/token
# - Monta la CA en: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

### Con SA Personalizada

```yaml
# Pod con SA específica
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: my-custom-sa  # Usa esta SA en lugar de default
  containers:
  - name: app
    image: myapp:latest
    
# Kubernetes:
# - Asigna SA: my-custom-sa
# - Monta su JWT token específico
```

---

## ¿Qué Pasa Sin Certificado service-account?

### Escenario de Fallo

```
Si la CA (service-account.crt) NO se distribuye a los nodos:

Pod:
1. Arranca y obtiene token JWT
2. Intenta conectar a API server
3. Envía el token

API server:
1. Recibe el token
2. Intenta verificar la firma: "¿Fue firmado con mi service-account.key?"
3. No puede verificar sin la clave pública correspondiente
4. ❌ RECHAZA: "Token no válido"

Resultado:
❌ Pod no puede comunicarse con API server
❌ No puede leer configmaps, secrets
❌ No puede descubrir servicios
💥 Aplicación falla
```

---

## Resumen

| Aspecto | Detalle |
|---------|---------|
| **¿Qué es SA?** | Identidad para pods en K8s |
| **¿Cómo se usan?** | Via JWT tokens montados en /var/run/secrets |
| **¿Quién crea tokens?** | API server usando service-account.key |
| **¿Quién verifica tokens?** | API server usando service-account.crt |
| **¿Quién distribuye cert?** | CA a todos los nodos |
| **Diferencia con certs** | Certs = componentes (archivos), Tokens = pods (JWT montado) |
| **Con cloud providers** | El token K8s se usa para asumir roles en AWS/GCP/Azure (IRSA) |

---

## Checklist de Entendimiento

- [ ] Entiendo que SA es identidad para pods
- [ ] Entiendo que los pods usan JWT tokens (no certificados directos)
- [ ] Entiendo que API server firma tokens con service-account.key
- [ ] Entiendo que service-account.crt es la clave pública (distribuida)
- [ ] Entiendo el flujo: pod arranca → obtiene token → conecta a API server
- [ ] Entiendo la diferencia entre certificados (componentes) y tokens (pods)
- [ ] Entiendo que sin service-account.crt correcta, los tokens no se pueden verificar
- [ ] Entiendo cómo IRSA usa el JWT de K8s para AWS
