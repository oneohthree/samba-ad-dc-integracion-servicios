# Instalación y configuración del servidor Bind9 DNS

Durante el aprovisionamiento se utilizó el `dns-backend=SAMBA_INTERNAL`, que provee un servidor DNS interno del paquete Samba; aunque funcional en un entorno básico, tiene determinadas desventajas, como son la asignación de servidores `DNS forwarders` y una caché de resolución lenta. Para suplir estas carencias, se instalará Bind9 integrándolo a Samba.

# Instalación de paquetes necesarios

```
apt install bind9 dnsutils
```

Editar el fichero `/etc/samba/smb.conf` y en la sección `[global]` añadir las directivas:

```
server services = -dns
nsupdate command = /usr/bin/nsupdate -g
```

Comentar ó eliminar la directiva `dns forwarder = 127.0.0.1`.

## Modificación del aprovisionamiento AD DC

Definir Bind9 como dns-backend.

```
mv /etc/bind/named.conf.local{,.org}
nano /etc/bind/named.conf.local
```

```
dlz "samba4" {
    database "dlopen /usr/lib/x86_64-linux-gnu/samba/bind9/dlz_bind9_10.so";
};
```

```
samba_upgradedns --dns-backend=BIND9_DLZ
chown bind /var/lib/samba/private/dns.keytab
mv /etc/bind/named.conf.options{,.org}
nano /etc/bind/named.conf.options
```

```
options {
    directory "/var/cache/bind";
    forwarders { ns.tld; };
    forward first;
    dnssec-validation no;
    auth-nxdomain no;
    listen-on-v6 { none; };
    tkey-gssapi-keytab "/var/lib/samba/private/dns.keytab";
    allow-query { any; };
    recursion yes;
};
```

```
mv /etc/default/bind9{,.org}
nano /etc/default/bind9
```

```
RESOLVCONF=no
OPTIONS="-4 -u bind"
```

`nano /var/lib/samba/private/named.conf.update`

```
grant *.example.tld wildcard *.0.168.192.in-addr.arpa. PTR TXT;  
grant local-ddns zonesub any;
```

Reiniciar los servicios

```
systemctl restart samba-ad-dc bind9
```

# Creación de zona inversa y registro PTR del servidor

```
samba-tool dns zonecreate localhost 0.168.192.in-addr.arpa -U 'administrator'%'P@s$w0rd.123'
samba-tool dns add localhost 0.168.192.in-addr.arpa 1 PTR 'dc.example.tld.' -U 'administrator'%'P@s$w0rd.123'
```

# Comprobaciones

Comprobar correcta ejecución del servidor Bind9 DNS.

```
netstat -tapn | grep 53
netstat -lptun
```

Comprobar registros DNS necesarios para el funcionamiento correcto de Samba AD DC

```
dig example.tld
host -t A dc.example.tld
host 192.168.0.1
host -t SRV _kerberos._udp.example.tld
host -t SRV _ldap._tcp.example.tld
```

Comprobar actualización automática de los registros DNS.

```
samba-tool dns query 127.0.0.1 example.tld @ ALL -U 'administrator'%'P@s$w0rd.123'
samba_dnsupdate --verbose --all-names
```
