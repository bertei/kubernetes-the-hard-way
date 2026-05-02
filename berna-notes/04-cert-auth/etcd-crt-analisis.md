## etcd - Store del Cluster

**Qué es:**
- Base de datos distribuida (clave-valor)
- Almacena TODO el estado de K8s: pods, services, deployments, etc
- Ejecuta en 2+ nodos (HA) - si 1 etcd cae, el otro sigue funcionando

**Certificado etcd-server.crt + etcd-server.key:**

| Aspecto | Detalle |
|---------|---------|
| **Tipo** | Server Certificate |
| **Por qué server?** | etcd ACTÚA COMO SERVIDOR (etcd1 ↔ etcd2 se conectan) |
| **SANs** | IPs donde etcd es válido: 192.168.56.11, 192.168.56.12, 127.0.0.1 |
| **Flujo TLS** | etcd2 se conecta a etcd1 → etcd1 presenta cert → etcd2 verifica SANs |

**Por qué SANs importan en etcd:**
- etcd1 necesita conectarse a etcd2 por su IP
- etcd1 verifica: "¿Este cert es válido para 192.168.56.12?" 
- Si el SAN no incluye 192.168.56.12 → rechazo (hostname mismatch)
- Por eso SANs incluye TODAS las IPs donde etcd puede estar

**Diferencia conceptual:**
- Client cert (admin): "Yo soy admin, confiá en mí por mi nombre"
- Server cert (etcd): "Yo soy válido para ESTAS IPs, conectáte a mí"

Escenario: etcd1 (192.168.56.11) necesita replicar datos a etcd2 (192.168.56.12)

etcd1: "Necesito conectarme a 192.168.56.12:2380"
etcd2 responde: "OK, aquí está mi etcd-server.crt (que tiene SANs: [192.168.56.11, 192.168.56.12, 127.0.0.1])"
etcd1 verifica: "¿Este cert es válido para la IP 192.168.56.12?"
Busca en SANs
Encuentra IP.2 = 192.168.56.12 ✅
Verifica que CA lo firmó ✅
CONFÍA en etcd2
Se abre conexión TLS segura ✅