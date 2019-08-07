# Consideraciones previas

Esta guía no presenta configuraciones avanzadas, tales como filtrado de contenido web, técnicas de antispam y antivirus o filtrado de origen y destino de email; sino que está enfocada en exponer la integración de servicios vitales -proxy, chat y correo electrónico-, en una red corporativa con el servicio de directorio `Samba AD DC`; aunque pudieran incluirse en futuras revisiones.

Tendiendo en cuenta lo anterior, se pautan las siguientes premisas:

* Sistema Operativo: Debian GNU/Linux 9 Stretch 64bits (instalación base)
* Repositorio de paquetes distribución Debian 9 Stretch
* Repositorio de paquetes Samba 4.9
* Existencia de un servidor NTP superior
* Existencia de un servidor proxy padre
* Nombre de host Samba AD DC: `dc`
* Dirección IP Samba AD DC: `192.168.0.1`
* Nombre de host Squid Proxy Server: `proxy`
* Dirección IP Squid Proxy Server: `192.168.0.2`
* Nombre de host eJabberd XMPP Server: `jb`
* Dirección IP eJabberd XMPP Server: `192.168.0.3`
* Nombre de host Postfix/Dovecot/Roundcube Mail Server: `mail`
* Dirección IP Postfix/Dovecot/Roundcube Mail Server: `192.168.0.4`
* Nombre de dominio: `example.tld`
* Los hosts miembros del dominio deben usar Samba AD DC como servidor DNS y de tiempo

## Configuración de los parámetros de red

### Samba AD DC Server

`nano /etc/network/interfaces`

    auto lo
    iface lo inet loopback

    auto enp0s3
    iface enp0s3 inet static
        address 192.168.0.1/24
        gateway 192.168.0.254
        dns-nameservers 127.0.0.1
        dns-search example.tld

`nano /etc/hosts`

    127.0.0.1       localhost
    192.168.0.1     dc.example.tld      dc

    `nano /etc/resolv.conf`

    domain example.tld
    nameserver 127.0.0.1

### Squid Proxy Server

`nano /etc/network/interfaces`

    auto lo
    iface lo inet loopback

    auto enp0s3
    iface enp0s3 inet static
        address 192.168.0.2/24
        gateway 192.168.0.254
        dns-nameservers 192.168.0.1
        dns-search example.tld

`nano /etc/hosts`

    127.0.0.1       localhost
    192.168.0.2     proxy.example.tld      proxy

`nano /etc/resolv.conf`

    domain example.tld
    nameserver 192.168.0.1

### eJabberd XMPP Server

`nano /etc/network/interfaces`

    auto lo
    iface lo inet loopback

    auto enp0s3
    iface enp0s3 inet static
        address 192.168.0.3/24
        gateway 192.168.0.254
        dns-nameservers 192.168.0.1
        dns-search example.tld

`nano /etc/hosts`

    127.0.0.1       localhost
    192.168.0.3     jb.example.tld      jb

`nano /etc/resolv.conf`

    domain example.tld
    nameserver 192.168.0.1

### Postfix/Dovecot/Roundcube Mail Server

`nano /etc/network/interfaces`

    auto lo
    iface lo inet loopback

    auto enp0s3
    iface enp0s3 inet static
        address 192.168.0.4/24
        gateway 192.168.0.254
        dns-nameservers 192.168.0.1
        dns-search example.tld

`nano /etc/hosts`

    127.0.0.1       localhost
    192.168.0.4     mail.example.tld      mail

`nano /etc/resolv.conf`

    domain example.tld
    nameserver 192.168.0.1

## Sincronización de tiempo

* Utilizar el cliente NTP de systemd.

`timedatectl set-ntp true`

* Definir el servidor de tiempo superior.

`mv /etc/systemd/timesyncd.conf{,.org}`
`nano /etc/systemd/timesyncd.conf`

Samba AD DC Server

    [Time]
    NTP=ntp.tld

Squid Proxy Server

    [Time]
    NTP=dc.example.tld

eJabberd XMPP Server

    [Time]
    NTP=dc.example.tld

Postfix/Dovecot/Roundcube Mail Server

    [Time]
    NTP=dc.example.tld

* Reiniciar el servicio cliente NTP de systemd

`systemctl restart systemd-timesyncd`

* Verificar el estado de la sincronización

`timedatectl status`
`journalctl --since -1h -u systemd-timesyncd`