# Instalación y configuración de Postfix/Dovecot Mail Server e integración con Samba AD DC

## Instalación de paquetes necesarios

`export DEBIAN_FRONTEND=noninteractive`
`apt install postfix-pcre postfix-ldap dovecot-core dovecot-ldap dovecot-pop3d dovecot-imapd dovecot-lmtpd ldap-utils mailutils`
`unset DEBIAN_FRONTEND`

## Configuración del sistema

Crear grupo y usuario locales para el almacén de buzones `vmail`

```
groupadd -g 5000 vmail
useradd -m -g 5000 -u 5000 -d /var/vmail -s /usr/sbin/nologin -c "Virtual Mailbox Storage" vmail
```

Crear certificado de seguridad TLS/SSL y asignar permisos

```
openssl req -x509 -nodes -days 3650 -sha512 \
    -subj "/C=CU/ST=Provincia/L=Ciudad/\
        O=EXAMPLE TLD/OU=IT/CN=mail.example.tld/\
        emailAddress=postmaster@example.tld/" \
    -addext "subjectAltName = DNS:smtp.example.tld,\
        DNS:pop3.example.tld,DNS:imap.example.tld,\
        DNS:webmail.example.tld,IP:192.168.0.4" \
    -newkey rsa:4096 \
    -out /etc/ssl/certs/exampleMail.crt \
    -keyout /etc/ssl/private/exampleMail.key
```

```
openssl dhparam -out /etc/ssl/dh2048.pem 2048
chmod 0444 /etc/ssl/certs/exampleMail.crt
chmod 0400 /etc/ssl/private/exampleMail.key
```

Comprobar correcta creación del certificado

```
openssl x509 -in /etc/ssl/certs/exampleMail.crt -text -noout
```

## Integración con Samba AD DC

Crear nueva Cuenta de Usuario del Dominio para el servicio `postfix`

```
samba-tool user create 'postfix' 'P@s$w0rd.345' \
    --surname='Dovecot Roundcube' \
    --given-name='Postfix' \
    --company='EXAMPLE' \
    --description='Mail Service Account'
```

```
samba-tool user setexpiry postfix --noexpiry
```

Crear buzón de correo electrónico para almacén de mensajes

```
samba-tool user create 'archive' 'P@s$w0rd.678' \
    --userou='OU=ACME' \
    --surname='Mail Storage' \
    --given-name='Archive' \
    --company='ACME' \
    --description='Archive Mail Storage Account' \
    --mail='archive@example.tld'
```

```
samba-tool user setexpiry archive --noexpiry
```

Crear registros DNS

```
samba-tool dns add localhost example.tld mail A '192.168.0.4' -U 'administrator'%'P@s$w0rd.123'
samba-tool dns add localhost 0.168.192.in-addr.arpa 4 PTR 'mail.example.tld.' -U 'administrator'%'P@s$w0rd.123'
samba-tool dns add localhost example.tld @ MX 'mail.example.tld 10' -U 'administrator'%'P@s$w0rd.123'
samba-tool dns add localhost example.tld @ TXT "'v=spf1 a:example.tld mx -all'" -U 'administrator'%'P@s$w0rd.123'
samba-tool dns add localhost example.tld smtp CNAME 'mail.example.tld' -U 'administrator'%'P@s$w0rd.123'
samba-tool dns add localhost example.tld pop3 CNAME 'mail.example.tld' -U 'administrator'%'P@s$w0rd.123'
samba-tool dns add localhost example.tld imap CNAME 'mail.example.tld' -U 'administrator'%'P@s$w0rd.123'
samba-tool dns add localhost example.tld webmail CNAME 'mail.example.tld' -U 'administrator'%'P@s$w0rd.123'
```

Crear nueva Unidad Organizativa `Email` para grupos de correo electrónico, perteneciente a `ACME`

```
samba-tool ou create 'OU=Email,OU=ACME,DC=example,DC=tld' --description='Email Groups Organizational Unit'
```

Crear Grupos de Usuarios de correo electrónico

```
samba-tool group add Everyone \
    --groupou='OU=Email,OU=ACME' \
    --description='All Users Email Group' \
    --mail='everyone@example.tld'

samba-tool group add Management \
    --groupou='OU=Email,OU=ACME' \
    --description='Management Email Group' \
    --mail='management@example.tld'

samba-tool group add Support \
    --groupou='OU=Email,OU=ACME' \
    --description='Technical Support Email Group' \
    --mail='support@example.tld'
```

Añadir usuarios a los grupos creados

```
samba-tool group addmembers 'Everyone' sheldon,leonard,rajesh
samba-tool group addmembers 'Management' sheldon
samba-tool group addmembers 'Support' rajesh,sheldon
```

## Configuración del servicio `postfix`

Realizar copia de seguridad de los ficheros de configuración

```
cp /etc/postfix/main.cf{,.org}
cp /etc/postfix/master.cf{,.org}
```

Declarar dominio de correo a gestionar

```
postconf -e "mydomain = example.tld"
postconf -e "smtpd_sasl_local_domain = example.tld"
postconf -e "virtual_mailbox_domains = example.tld"
```

Definir transporte virtual del dominio de correo

```
postconf -e "virtual_transport = lmtp:unix:private/dovecot-lmtp"
```

Definir usuarios virtuales de correo electrónico

`postconf -e "smtpd_sender_login_maps = proxy:ldap:/etc/postfix/virtual_sender_login_maps.cf"`
`nano /etc/postfix/virtual_sender_login_maps.cf`

```
server_host = dc.example.tld
server_port = 389
version = 3
bind = yes
start_tls = no
bind_dn = postfix@example.tld
bind_pw = P@s$w0rd.345
search_base = OU=ACME,DC=example,DC=tld
scope = sub
query_filter = (&(userPrincipalName=%s)(objectClass=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))
result_attribute = userPrincipalName
debuglevel = 0
```

Definir buzón almacén de correo electrónico

```
postconf -e "always_bcc = archive@example.tld"
```

Definir buzones virtuales de correo electrónico

```
postconf -e "virtual_minimum_uid = 5000"
postconf -e "virtual_uid_maps = static:5000"
postconf -e "virtual_gid_maps = static:5000"
postconf -e "virtual_mailbox_base = /var/vmail"
postconf -e "virtual_mailbox_maps = proxy:ldap:/etc/postfix/virtual_mailbox_maps.cf"
```

`nano /etc/postfix/virtual_mailbox_maps.cf`

```
server_host = dc.example.tld
server_port = 389
version = 3
bind = yes
start_tls = no
bind_dn = postfix@example.tld
bind_pw = P@s$w0rd.345
search_base = OU=ACME,DC=example,DC=tld
scope = sub
query_filter = (&(objectClass=person)(userPrincipalName=%s)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))
result_attribute = userPrincipalName
result_format = %d/%u/Maildir/
debuglevel = 0
```

Definir listas y aliases virtuales de correo electrónico

`postconf -e "virtual_alias_maps = proxy:ldap:/etc/postfix/virtual_list_maps.cf, proxy:ldap:/etc/postfix/virtual_alias_maps.cf"`
`nano /etc/postfix/virtual_list_maps.cf`

```
server_host = dc.example.tld
server_port = 389
version = 3
bind = yes
start_tls = no
bind_dn = postfix@example.tld
bind_pw = P@s$w0rd.345
search_base = OU=ACME,DC=example,DC=tld
scope = sub
query_filter = (&(objectClass=person)(memberOf=cn=%u,OU=Email,OU=ACME,DC=example,DC=tld)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))
result_attribute = userPrincipalName
debuglevel = 0
```

`nano /etc/postfix/virtual_alias_maps.cf`

```
server_host = dc.example.tld
server_port = 389
version = 3
bind = yes
start_tls = no
bind_dn = postfix@example.tld
bind_pw = P@s$w0rd.345
search_base = OU=ACME,DC=example,DC=tld
scope = sub
query_filter = (&(objectClass=person)(otherMailbox=%s)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))
result_attribute = userPrincipalName
debuglevel = 0
    

**NOTA**: El atributo `otherMailbox` puede editarse utilizando el comando `samba-tool user edit <username>` o mediante las herramientas administrativas GUI `RSAT` o `Apache Directory Studio`.

Habilitar puerto seguro `TCP\587 Submission` y establecer comunicación con `dovecot`

`nano /etc/postfix/master.cf`

```
submission      inet    n       -       y       -       10      smtpd
    -o syslog_name=postfix/submission
    -o smtpd_tls_security_level=encrypt
    -o smtpd_sasl_auth_enable=yes
    -o smtpd_tls_cert_file=/etc/ssl/certs/exampleMail.crt
    -o smtpd_tls_key_file=/etc/ssl/private/exampleMail.key
    -o smtpd_client_restrictions=permit_sasl_authenticated,reject
dovecot         unix    -       n       n       -       -       pipe
    flags=DRhu user=vmail:vmail argv=/usr/lib/dovecot/dovecot-lda -f ${sender} -d ${recipient}
```

Comprobaciones

```
postmap -q leonard@example.tld ldap:/etc/postfix/virtual_sender_login_maps.cf
    leonard@example.tld

postmap -q rajesh@example.tld ldap:/etc/postfix/virtual_mailbox_maps.cf
    example.tld/rajesh/Maildir/

postmap -q everyone@example.tld ldap:/etc/postfix/virtual_list_maps.cf
    leonard@example.tld
    rajesh@example.tld
    sheldon@example.tld

postmap -q postmaster@example.tld ldap:/etc/postfix/virtual_alias_maps.cf
    john.doe@example.tld
```

Reiniciar el servicio. (Ver Anexos para una visión general de los ficheros de configuración).

```
systemctl restart postfix.service
```

## Configuración del servicio `dovecot`

Realizar salva de seguridad del fichero de configuración principal

```
cp /etc/dovecot/dovecot.conf{,.org}
```

Crear script de alerta de sobreuso de cuota y asignar permiso de ejecución

`nano /usr/local/bin/quota-warning`

```
#!/bin/bash

PERCENT=${1}
USER=${2}
DOMAIN=${USER#*@}

cat << EOT | /usr/lib/dovecot/dovecot-lda -d ${USER} -o "plugin/quota=maildir:User quota:noenforcing"
From: no-reply@${DOMAIN}
Subject: ALERTA: USO DE CUOTA SUPERIOR AL ${PERCENT}%

ESTIMADO(A) USUARIO(A),

SU BUZON DE CORREO ACTUALMENTE UTILIZA MAS DEL ${PERCENT}% DE LA CUOTA ASIGNADA.
ELIMINE ALGUNOS CORREOS PARA LIBERAR ESPACIO Y PODER SEGUIR RECIBIENDO EMAILS.

ESTE ES UN MENSAJE AUTOMATIZADO DEL SISTEMA, NO RESPONDER
EOT

exit 0
```

`chmod +x /usr/local/bin/quota-warning`

Integrar con Samba AD DC. (Ver Anexos para una visión general del fichero de configuración)

`nano /etc/dovecot/dovecot.conf`

```
userdb {
    args = /etc/dovecot/dovecot-ldap.conf
    driver = ldap
}
passdb {
    args = /etc/dovecot/dovecot-ldap.conf
    driver = ldap
}
```

`nano /etc/dovecot/dovecot-ldap.conf`

```
hosts = dc.example.tld:389
auth_bind = yes
ldap_version = 3
dn = postfix@example.tld
dnpass = P@s$w0rd.345
base = OU=ACME,DC=example,DC=tld
deref = never
scope = subtree
user_filter = (&(objectClass=person)(userPrincipalName=%u)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))
user_attrs = maxStorage=quota_rule=*:bytes=%$
pass_filter = (&(objectClass=person)(userPrincipalName=%u)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))
pass_attrs = userPassword=password
default_pass_scheme = SHA512-CRYPT
```

**NOTA**: El atributo `maxStorage` puede editarse utilizando el comando `samba-tool user edit <username>` o mediante las herramientas administrativas GUI `RSAT` o `Apache Directory Studio`.

Reiniciar el servicio

```
systemctl restart dovecot.service
```

## Configuración del servicio `roundcube`

Integrar con Samba AD DC. (Ver Anexos para una visión general del fichero de configuración)

`nano /opt/roundcube/config/config.inc.php`

```
...
// Samba AD DC Address Book
$config['autocomplete_addressbooks'] = array('sql', 'global_ldap_abook');
$config['ldap_public']["global_ldap_abook"] = array(
    'name'              => 'Mailboxes',
    'hosts'             => array('dc.example.tld'),
    'port'              => 389,
    'use_tls'           => false,
    'ldap_version'      => '3',
    'network_timeout'   => 10,
    'user_specific'     => false,
    'base_dn'       => 'OU=ACME,DC=example,DC=tld',
    'bind_dn'       => 'postfix@example.tld',
    'bind_pass'     => 'P@s$w0rd.345',
    'writable'      => false,
    'search_fields' => array(
        'mail',
        'cn',
        'sAMAccountName',
        'displayName',
        'sn',
        'givenName',
    ),
    'fieldmap' => array(
        'name'          => 'cn',
        'surname'       => 'sn',
        'firstname'     => 'givenName',
        'title'         => 'title',
        'email'         => 'mail:*',
        'phone:work'    => 'telephoneNumber',
        'phone:mobile'  => 'mobile',
        'phone:workfax' => 'facsimileTelephoneNumber',
        'street'        => 'street',
        'zipcode'       => 'postalCode',
        'locality'      => 'l',
        'department'    => 'department',
        'notes'         => 'description',
        'photo'         => 'jpegPhoto',
     ),
     'sort'          => 'cn',
     'scope'         => 'sub',
     'filter'        => '(&(|(objectclass=person))(!(mail=archive@example.tld))(!(userAccountControl:1.2.840.113556.1.4.803:=2)))',
     'fuzzy_search'  => true,
     'vlv'           => false,
     'sizelimit'     => '0',
     'timelimit'     => '0',
     'referrals'     => false,
     'group_filters' => array(
         'departments' => array(
         'name'    => 'Lists',
         'scope'   => 'sub',
         'base_dn' => 'OU=Email,OU=ACME,DC=example,DC=tld',
         'filter'  => '(objectClass=group)',
         ),
      ),
  );
    ...
```
