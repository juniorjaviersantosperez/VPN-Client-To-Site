# Comandos — VPN Client-To-Site SSL VPN (Desde Cero)

**Alumno:** Junior Javier Santos Perez | **Matrícula:** 2024-1599

---

## PASO 1 — VPC

```bash
ip 10.15.99.2/24 10.15.99.1
save
```

---

## PASO 2 — FortiGate

### Hostname e interfaces
```bash
config system global
    set hostname "SERVIDOR-FGT"
end

config system interface
    edit "port1"
        set mode dhcp
        set allowaccess ping https ssh
    next
    edit "port2"
        set ip 10.15.99.1 255.255.255.0
        set allowaccess ping https ssh
    next
end
```

### Ruta por defecto
```bash
config router static
    edit 1
        set gateway 192.168.129.2
        set device "port1"
    next
end
```

### Pool de IPs para clientes VPN
```bash
config firewall address
    edit "SSLVPN_TUNNEL_ADDR1"
        set type iprange
        set start-ip 10.212.134.200
        set end-ip 10.212.134.210
    next
end
```

### Portal SSL VPN
```bash
config vpn ssl web portal
    edit "full-access"
        set ip-pools "SSLVPN_TUNNEL_ADDR1"
        set tunnel-mode enable
        set split-tunneling disable
    next
end
```

### Servicio SSL VPN
```bash
config vpn ssl settings
    set servercert "Fortinet_Factory"
    set tunnel-ip-pools "SSLVPN_TUNNEL_ADDR1"
    set port 443
    set source-interface "port1"
    set source-address "all"
    set default-portal "full-access"
end
```

### Usuario y grupo VPN
```bash
config user local
    edit "vpnuser"
        set type password
        set passwd MiClaveSegura123
    next
end

config user group
    edit "sslvpn-group"
        set member "vpnuser"
    next
end
```

### Políticas de firewall
```bash
config firewall policy
    edit 1
        set name "SSLVPN-to-LAN"
        set srcintf "ssl.root"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set groups "sslvpn-group"
        set nat enable
    next
    edit 2
        set name "LAN-to-WAN"
        set srcintf "port2"
        set dstintf "port1"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
    next
end
```

### Verificación FortiGate
```bash
get system interface
show vpn ssl settings
show vpn ssl web portal
show firewall policy
get vpn ssl monitor
```

---

## PASO 3 — Linux (Cliente VPN)

### Instalar dependencias y compilar openfortivpn
```bash
sudo apt install -y autoconf automake libssl-dev pkg-config ppp git

git clone https://github.com/adrienverge/openfortivpn.git
cd openfortivpn

sed -i 's/if (tunnel->config->min_tls > 0) {/if (tunnel->config->min_tls > 0) {\n\t\t\tif (tunnel->config->min_tls > TLS1_VERSION)\n\t\t\t\ttunnel->config->min_tls = TLS1_VERSION;/' src/tunnel.c

./autogen.sh
./configure --prefix=/usr/local --sysconfdir=/etc
make
sudo make install
```

### Bajar restricciones TLS en OpenSSL
```bash
sudo sed -i 's/MinProtocol = TLSv1.2/MinProtocol = TLSv1.0/' /etc/ssl/openssl.cnf
sudo sed -i 's/CipherString = DEFAULT:@SECLEVEL=2/CipherString = DEFAULT:@SECLEVEL=0/' /etc/ssl/openssl.cnf
```

### Archivo de configuración VPN
```bash
sudo nano /etc/openfortivpn/config
```
```
host = 192.168.129.154
port = 443
username = vpnuser
password = MiClaveSegura123
insecure-ssl = true
trusted-cert = 286efc05555bc05560acbbb14928b35b93b7da0e6a47e1a6701bc983704f1190
```

---

## PASO 4 — Pruebas ANTES de conectar VPN

```bash
traceroute 10.15.99.2
ping -c 3 10.15.99.2
```

---

## PASO 5 — Conectar VPN

```bash
sudo openfortivpn 192.168.129.154:443 -u vpnuser \
  --insecure-ssl \
  --trusted-cert 286efc05555bc05560acbbb14928b35b93b7da0e6a47e1a6701bc983704f1190
```

---

## PASO 6 — Pruebas DESPUÉS de conectar VPN

```bash
ip a show ppp0
ip route
traceroute 10.15.99.2
ping -c 3 10.15.99.1
ping -c 3 10.15.99.2
```

---

## PASO 7 — Verificación en FortiGate (mientras VPN está activa)

```bash
get vpn ssl monitor
show vpn ssl settings
show vpn ssl web portal
show firewall policy
```
