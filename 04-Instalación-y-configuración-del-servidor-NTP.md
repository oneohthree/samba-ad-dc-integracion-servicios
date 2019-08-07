TODO: ntp vs chrony

# Instalación y configuración del servidor NTP

El servidor Samba AD DC actuará como servidor de tiempo (Network Time Protocol Server - NTP Server) propiciando la sincronización de los relojes de los hosts y sistemas informáticos existentes en su entorno de red.

## Instación de paquetes necesarios

`apt install ntpdate ntp`

## Configuración del servidor NTP

`mv /etc/ntp.conf{,.org}`  
`nano /etc/ntp.conf`

    driftfile /var/lib/ntp/ntp.drift
    logfile /var/log/ntpd.log
    statistics loopstats peerstats clockstats
    filegen loopstats file loopstats type day enable
    filegen peerstats file peerstats type day enable
    filegen clockstats file clockstats type day enable
    server 127.127.1.1
    fudge 127.127.1.1 stratum 12
    server ntp.tld iburst prefer
    ntpsigndsocket /var/lib/samba/ntp_signd
    restrict -4 default kod notrap nomodify nopeer noquery mssntp
    restrict default mssntp
    restrict ntp.example.tld mask 255.255.255.255 nomodify notrap nopeer noquery
    restrict 127.0.0.1
    restrict ::1
    restrict source notrap nomodify noquery
    broadcast 192.168.0.255

* Establecer permisos.

`chgrp ntp /var/lib/samba/ntp_signd`  
`usermod -a -G staff ntp`

* Reiniciar el servicio.

`systemctl restart ntp`

# Comprobaciones

`systemctl status ntp`  
`ntpdate -vqd ntp.tld`  
`ntpq -p`
