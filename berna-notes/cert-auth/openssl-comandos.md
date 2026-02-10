### Comandos OpenSSL Para Inspeccionar Certificados

#### Por Qué Hacer Esto

Cuando generas certificados, debes verificar que sean correctos:
- ¿Tiene la CA los permisos correctos? (CA:TRUE)
- ¿Las fechas de validez son correctas?
- ¿Quién firmó este certificado?
- ¿Están relacionadas la clave privada y el certificado?

#### Comandos Más Útiles

##### 1. Ver un Certificado Completo

```bash
openssl x509 -in ca.crt -text -noout
```

Muestra:
- Versión y número serial
- Quién lo firmó (Issuer)
- Quién es (Subject)
- Cuándo caduca (Validity)
- Clave pública (modulus + exponent)
- Extensiones (CA:TRUE, etc.)
- Firma digital

##### 2. Ver Solo Información de Firma

```bash
openssl x509 -in ca.crt -noout -issuer
openssl x509 -in ca.crt -noout -subject

# Si son iguales = auto-firmado ✓
```

##### 3. Ver Validez

```bash
openssl x509 -in ca.crt -noout -dates

# Output:
# notBefore=Feb  6 03:20:00 2026 GMT
# notAfter=Aug  2 03:20:00 2029 GMT
```

##### 4. Ver Clave Privada

```bash
openssl rsa -in ca.key -text -noout | head -20
# ⚠️ SENSIBLE: Muestra tu secret
# NUNCA en producción
```

##### 5. Verificar que Clave y Cert Coinciden

```bash
# Obtén hash de la privada:
openssl rsa -in ca.key -noout -modulus | openssl md5

# Obtén hash del cert:
openssl x509 -in ca.crt -noout -modulus | openssl md5

# Si son iguales = son pareja ✓
```

##### 6. Ver CSR (Certificate Signing Request)

```bash
openssl req -in ca.csr -text -noout
```

Muestra:
- Subject (quien pide)
- Public Key Info
- Firma de la solicitud

##### 7. Verificar que Certificado es Válido

```bash
openssl verify -CAfile ca.crt ca.crt

# Output: ca.crt: OK (si es válido)
```

#### Ejercicio: Escaneá tu CA Certificate

```bash
cd ~/certs

# Paso 1: Ver toda la info
openssl x509 -in ca.crt -text -noout

# Busca:
# - Issuer: CN = KUBERNETES-CA (es él mismo = auto-firmado)
# - Subject: CN = KUBERNETES-CA (mismo = auto-firmado)
# - CA:TRUE (puede firmar otros)
# - NotAfter: 2029 (válido por 1000 días)

# Paso 2: Ver resumido
echo "Issuer:"; openssl x509 -in ca.crt -noout -issuer
echo "Subject:"; openssl x509 -in ca.crt -noout -subject
echo "Validity:"; openssl x509 -in ca.crt -noout -dates

# Paso 3: Ver si es CA
openssl x509 -in ca.crt -text -noout | grep -A1 "CA:"
# Debe mostrar: X509v3 Basic Constraints: critical
#               CA:TRUE
```

#### Temas Que Verás

```
X.509 = Estándar de certificados digitales
RSA = Algoritmo de encriptación asimétrica
Modulus = Número gigante que forma la clave
Exponent = 65537 (estándar)
sha256WithRSAEncryption = Algoritmo de firma (seguro)
Signature Value = Imposible de falsificar sin ca.key
```


ca.key = Texto Base64 que contiene:
├─ Números privados enormes
├─ Números primos enormes
└─ Matemática RSA compleja

ca.crt = Texto Base64 que contiene:
├─ Tu identidad (CN, O)
├─ Tu clave PÚBLICA (derivada de ca.key)
├─ Firma digital (imposible falsificar)
└─ Fechas de validez

ca.csr = Texto Base64 que contiene:
├─ Tu identidad (CN, O)
├─ Tu clave pública
└─ Tu firma (prueba que eres tú)

Formato PEM = Base64 + delimitadores (-----BEGIN/END-----)
NOT encriptado = simplemente codificado en texto