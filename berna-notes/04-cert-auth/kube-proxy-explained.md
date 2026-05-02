# kube-proxy - Load Balancer del Cluster

## ¿Qué es kube-proxy?

**Definición:**
- Componente que corre en **CADA WORKER** (como kubelet)
- Implementa los **Servicios de Kubernetes**
- Intercepta tráfico destinado a IPs virtuales (ServiceIP) y lo redirige a Pods reales

---

## Ejemplo de Qué Hace

### Escenario: Usuario crea un Service

```yaml
kind: Service
spec:
  type: ClusterIP
  clusterIP: 10.0.0.50      # IP virtual (NO existe en ningún interface)
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 8080
```

**Resultado:**
- Service tiene IP virtual: `10.0.0.50`
- Service apunta a 3 Pods (endpoints que matchean `app: nginx`)

### Flujo: Un Pod Conecta al Service

```
1. Pod en node01: "curl 10.0.0.50:80"
   ↓
2. Kernel intenta enrutar 10.0.0.50
   → IP virtual, no existe en la red
   ↓
3. kube-proxy (en node01) INTERCEPTA:
   "Alguien quiere conectar a 10.0.0.50:80"
   ↓
4. kube-proxy pregunta a API server:
   "¿Cuáles son los endpoints para el service 10.0.0.50?"
   ↓
5. API server responde:
   "Los endpoints son: Pod1 (10.244.1.10:8080), Pod2 (10.244.1.11:8080), Pod3 (10.244.3.30:8080)"
   ↓
6. kube-proxy instala regla iptables:
   10.0.0.50:80 → Pod1:8080 (o Pod2 o Pod3, round-robin)
   ↓
7. Tráfico se redirige a un Pod real ✅
```

---

## ¿Por Qué "Proxy"?

Un proxy **toma requests y las redirige** hacia su destino real.

### Sin API server = Proxy inútil

```
❌ Sin información del API server:
  kube-proxy: "Alguien quiere conectar a 10.0.0.50"
             "¿A dónde redireciono? No tengo información..."

✅ Con API server:
  kube-proxy: "Alguien quiere conectar a 10.0.0.50"
  kube-proxy pregunta: "API server, dame los endpoints"
  API server: "Los endpoints son: Pod1, Pod2, Pod3"
  kube-proxy: "OK, instalo iptables para redireccionar"
```

---

## El Certificado: kube-proxy.crt + kube-proxy.key

### Tipo: Client Certificate

| Aspecto | Detalle |
|---------|---------|
| **Tipo** | Client Certificate |
| **Por qué client?** | kube-proxy INICIA conexiones al API server (es cliente, no servidor) |
| **CN** | system:kube-proxy (identidad del componente) |
| **Rol** | Permite leer servicios y endpoints del API server |
| **Flujo TLS** | kube-proxy → API server para obtener información |

---

## Flujo de Autenticación (Con Certificado)

### Paso 1: kube-proxy Arranca

```
kube-proxy (en worker):
1. Lee kube-proxy.crt + kube-proxy.key
2. Se conecta a API server en 192.168.56.30:6443
3. Presenta kube-proxy.crt + kube-proxy.key
4. Dice: "Soy system:kube-proxy, firmado por CA"
```

### Paso 2: API Server Verifica

```
API server:
1. Verifica: ¿Este cert está firmado por MI ca.crt? ✅
2. Verifica: ¿CN=system:kube-proxy? ✅
3. CONFÍA en kube-proxy
4. Permite: Acceso a servicio/endpoints
```

### Paso 3: kube-proxy Configura iptables

```
kube-proxy:
1. Recibe la lista de servicios y endpoints (info confiable del API server)
2. Instala reglas iptables en el kernel:
   "Si tráfico destino a 10.0.0.50 → redirige a Pod1 o Pod2 o Pod3"
3. Servicios funciona ✅
```

### Sin Certificado = Fallo Total

```
❌ Sin cert:
  API server rechaza: "No confío en ti, rechazado"
  ↓
  kube-proxy no obtiene endpoints
  ↓
  kube-proxy no instala reglas iptables
  ↓
  Servicios NO FUNCIONAN
  ↓
  Pod A no puede conectar a Pod B a través del Service
```

---

## Comparación: Client Certificates en Kubernetes

Los siguientes componentes usan **client certificates** para conectarse al API server:

| Componente | CN | Qué Obtiene | Rol |
|-----------|-----|-----------|-----|
| **kubectl (admin)** | admin | TODO los recursos | Admin (system:masters) |
| **controller-manager** | system:kube-controller-manager | Crear/modificar recursos | Control de controladores |
| **kubelet** | system:node:NODE_NAME | Sus Pods, secrets, configmaps | Worker node |
| **kube-proxy** | system:kube-proxy | Servicios y endpoints | Proxy de servicios |

**Punto clave:** Cada uno usa client cert para demostrar su identidad y obtener **diferentes permisos** según su rol.

---

## Resumen

| Aspecto | Descripción |
|---------|-----------|
| **¿Qué es?** | Componente que intercepta tráfico a IPs virtuales (Services) y lo redirige a Pods reales |
| **¿Dónde corre?** | En CADA WORKER (como kubelet) |
| **¿Cómo funciona?** | Consulta API server para obtener endpoints, instala reglas iptables |
| **¿Por qué client cert?** | Necesita conectarse al API server para obtener información de servicios |
| **¿Qué sucede sin cert?** | API server rechaza conexiones, no obtiene endpoints, servicios no funcionan |
| **CN del cert** | system:kube-proxy |

---

## Checklist de Entendimiento

- [ ] Entiendo que kube-proxy intercepta tráfico a IPs virtuales
- [ ] Entiendo que kube-proxy consulta al API server para obtener endpoints
- [ ] Entiendo que sin client cert, no puede hablar con el API server
- [ ] Entiendo que sin kube-proxy, los Servicios no funcionan
- [ ] Entiendo la diferencia entre Service IP (virtual) y Pod IP (real)
- [ ] Entiendo que iptables es el mecanismo de redirección
