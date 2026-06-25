# Problema TLS — Diagnóstico y Solución Completa

**Alumno:** Junior Javier Santos Perez  
**Matrícula:** 2024-1599  
**Fecha:** 24 de junio de 2026  

---

## ¿Qué es TLS?

**TLS** (Transport Layer Security) es un protocolo criptográfico que protege las comunicaciones en red garantizando tres cosas:

| Garantía | Significado |
|----------|-------------|
| **Confidencialidad** | Los datos viajan cifrados — nadie puede leerlos aunque los intercepte |
| **Integridad** | Los datos no pueden modificarse en tránsito sin que se detecte |
| **Autenticación** | Garantiza que te conectas al servidor correcto mediante certificados |

### Versiones de TLS

| Versión | Año | Estado |
|---------|-----|--------|
| TLS 1.0 | 1999 | ❌ Deprecado — vulnerable |
| TLS 1.1 | 2006 | ❌ Deprecado — vulnerable |
| TLS 1.2 | 2008 | ⚠️ Aún usado |
| TLS 1.3 | 2018 | ✅ Recomendado actual |

---

## ¿Qué es OpenSSL?

**OpenSSL** es la librería de código abierto que implementa TLS en Linux. Es el motor que hace funcionar el cifrado — sin OpenSSL no hay TLS.

En este laboratorio Kali Linux tenía instalado **OpenSSL 3.5.5** (enero 2026), una versión muy reciente que por seguridad **bloquea TLS 1.0 y 1.1 completamente**.

```bash
openssl version
```
```
OpenSSL 3.5.5 27 Jan 2026 (Library: OpenSSL 3.5.5 27 Jan 2026)
```

---

## El Problema

### Causa raíz

**FortiGate 7.0.3** (lanzado en 2021) negocia SSL VPN usando **TLS 1.0** por defecto.  
**OpenSSL 3.5.5** en Kali Linux 2026 tiene **TLS 1.0 y 1.1 bloqueados** por ser inseguros.

Cuando el cliente intentaba conectar ocurría esto:

```
Linux (OpenSSL 3.5.5)          FortiGate 7.0.3
        |                              |
        |--- "Hola, uso TLS 1.2" ----->|
        |                              |
        |<-- "Solo acepto TLS 1.0" ----|
        |                              |
        |❌ RECHAZADO                  |
        | tlsv1 alert protocol version |
```

### Error original

Al intentar conectar con el comando básico:

```bash
sudo openfortivpn 10.15.99.1:443 -u vpnuser -p MiClaveSegura123
```

FortiGate devolvía este error:

```
ERROR: SSL_connect: error:0A00042E:SSL routines::tlsv1 alert protocol version
You might want to try --insecure-ssl or specify a different --cipher-list
INFO:  Closed connection to gateway.
INFO:  Could not log out.
```

---

## Intentos de Solución Fallidos

### Intento 1 — `--insecure-ssl`

```bash
sudo openfortivpn 10.15.99.1:443 -u vpnuser -p MiClaveSegura123 --insecure-ssl
```

**Resultado:** ❌ Mismo error  
**Por qué falló:** `--insecure-ssl` solo desactiva la verificación del certificado, no cambia la versión de TLS.

---

### Intento 2 — Cambiar cipher list

```bash
sudo openfortivpn 10.15.99.1:443 -u vpnuser -p MiClaveSegura123 --insecure-ssl --cipher-list "AES128-SHA"
```

**Resultado:** ❌ Mismo error  
**Por qué falló:** El problema no era el cipher sino la versión del protocolo TLS.

---

### Intento 3 — Forzar TLS mínimo desde la línea de comando

```bash
sudo openfortivpn 10.15.99.1:443 -u vpnuser -p MiClaveSegura123 --insecure-ssl --min-tls=1.0
```

**Resultado:** ❌ Mismo error  
**Por qué falló:** El parámetro `--min-tls` no estaba disponible en esa versión.

---

### Intento 4 — Modificar `/etc/ssl/openssl.cnf`

Se editó el archivo de configuración de OpenSSL para bajar las restricciones:

```bash
sudo nano /etc/ssl/openssl.cnf
```

Se cambió:
```
[system_default_sect]
MinProtocol = TLSv1.2        # Original
CipherString = DEFAULT:@SECLEVEL=2
```

Por:
```
[system_default_sect]
MinProtocol = TLSv1.0        # Modificado
CipherString = DEFAULT:@SECLEVEL=0
```

**Resultado:** ❌ Mismo error  
**Por qué falló:** `openfortivpn` forzaba TLS 1.2 internamente en el código fuente, ignorando completamente la configuración del sistema. Esto se confirmó en los logs:

```
DEBUG: Setting minimum protocol version to: 0x303.
```
> `0x303` es el código hexadecimal de TLS 1.2 — el programa lo forzaba internamente.

---

### Intento 5 — Comandos TLS en FortiGate

Se intentó configurar la versión TLS desde el FortiGate:

```bash
config vpn ssl settings
    set tlsv1-0 enable
    set tlsv1-2 enable
    set ssl-min-proto-ver tls1-2
    set enc-algorithm high
end
```

**Resultado:** ❌ Todos fallaron con `command parse error`  
**Por qué falló:** FortiOS 7.0.3 no tiene esos parámetros disponibles en la licencia de evaluación.

---

## La Solución Definitiva

### ¿Por qué fue necesario compilar?

La versión de `openfortivpn` instalada en Kali tenía el siguiente código en `src/tunnel.c` que forzaba TLS 1.2 sin permitir cambiarlo:

```c
// Línea 1068 — src/tunnel.c
if (tunnel->config->min_tls > 0) {
    SSL_CTX_set_min_proto_version(tunnel->ssl_context,
                                   tunnel->config->min_tls);
}
```

El problema era que internamente siempre establecía `min_tls = 0x303` (TLS 1.2), sin importar lo que dijera la configuración del sistema.

**Compilar** significa tomar el código fuente del programa (la receta original), **modificar una línea**, y volver a construir el programa con ese cambio aplicado.

---

### Paso 1 — Instalar dependencias de compilación

```bash
sudo apt install -y autoconf automake libssl-dev pkg-config ppp git
```

---

### Paso 2 — Descargar el código fuente

```bash
git clone https://github.com/adrienverge/openfortivpn.git
cd openfortivpn
```

---

### Paso 3 — Verificar la línea problemática

```bash
grep -n "min_tls" src/tunnel.c src/config.c src/config.h | head -30
```

Esto confirmó que en la línea 1068 de `tunnel.c` estaba el código que forzaba la versión TLS mínima.

---

### Paso 4 — Aplicar el parche (la modificación)

Este es el comando clave que modificó el código fuente:

```bash
sed -i 's/if (tunnel->config->min_tls > 0) {/if (tunnel->config->min_tls > 0) {\n\t\t\tif (tunnel->config->min_tls > TLS1_VERSION)\n\t\t\t\ttunnel->config->min_tls = TLS1_VERSION;/' src/tunnel.c
```

**¿Qué hace `sed`?**  
`sed` es una herramienta de Linux que busca y reemplaza texto dentro de archivos, como el "buscar y reemplazar" de Word pero para código.

**¿Qué cambió exactamente?**

Antes del parche:
```c
if (tunnel->config->min_tls > 0) {
    SSL_CTX_set_min_proto_version(...);  // Usaba TLS 1.2 siempre
}
```

Después del parche:
```c
if (tunnel->config->min_tls > 0) {
    if (tunnel->config->min_tls > TLS1_VERSION)   // ← LÍNEA AGREGADA
        tunnel->config->min_tls = TLS1_VERSION;    // ← LÍNEA AGREGADA
    SSL_CTX_set_min_proto_version(...);  // Ahora usa TLS 1.0
}
```

**En español simple:** le dijimos al programa — *"si la versión mínima que vas a usar es mayor que TLS 1.0, bájala a TLS 1.0"*.

---

### Paso 5 — Compilar e instalar

```bash
./autogen.sh
./configure --prefix=/usr/local --sysconfdir=/etc
make
sudo make install
```

| Comando | Función |
|---------|---------|
| `./autogen.sh` | Prepara los scripts de configuración |
| `./configure` | Verifica dependencias y prepara la compilación |
| `make` | Traduce el código fuente a lenguaje de máquina |
| `sudo make install` | Instala el programa compilado en el sistema |

---

### Paso 6 — Conectar con el programa parcheado

```bash
sudo openfortivpn 192.168.129.154:443 -u vpnuser -p MiClaveSegura123 --insecure-ssl --trusted-cert 286efc05555bc05560acbbb14928b35b93b7da0e6a47e1a6701bc983704f1190
```

**Resultado:** ✅ Conexión exitosa

Los logs confirmaron que ahora usaba TLS 1.0 (`0x301`):
```
DEBUG: Setting minimum protocol version to: 0x301.
INFO:  Connected to gateway.
INFO:  Authenticated.
INFO:  Remote gateway has allocated a VPN.
INFO:  Interface ppp0 is UP.
INFO:  Tunnel is up and running.
```

---

## Resumen del Problema y Solución

| | Detalle |
|--|---------|
| **Causa** | FortiGate 7.0.3 usa TLS 1.0; OpenSSL 3.5.5 lo bloquea por ser inseguro |
| **Error** | `SSL routines::tlsv1 alert protocol version` |
| **Por qué fallaron los otros intentos** | `openfortivpn` forzaba TLS 1.2 internamente en el código fuente |
| **Solución** | Compilar `openfortivpn` desde código fuente con parche que permite TLS 1.0 |
| **Comando clave** | `sed -i` para modificar `src/tunnel.c` antes de compilar |
| **En producción** | La solución correcta sería actualizar FortiGate a v7.2+ que soporta TLS 1.2/1.3 |

---

## ¿Por qué TLS 1.0 es inseguro?

TLS 1.0 tiene vulnerabilidades conocidas como:

- **POODLE** — permite descifrar tráfico cifrado
- **BEAST** — permite interceptar y descifrar sesiones
- **CRIME** — permite robar cookies de sesión

Por eso OpenSSL 3.x y todos los navegadores modernos lo tienen bloqueado. En un entorno de producción **nunca se debería usar TLS 1.0** — la solución real es actualizar el FortiGate.
