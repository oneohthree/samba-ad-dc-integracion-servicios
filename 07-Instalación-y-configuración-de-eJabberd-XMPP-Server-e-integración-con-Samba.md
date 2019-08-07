# Instalación y configuración de eJabberd XMPP Server e integración con Samba AD DC

## Instalación de paquetes necesarios

```
apt install ejabberd
```

Definir el nombre de dominio del servidor `eJabberd`

```
cp /etc/ejabberd/ejabberd.yml{,.org}
nano /etc/ejabberd/ejabberd.yml
```

```
hosts:
  - "example.tld"
```

## Creación de registros DNS

```
samba-tool dns add localhost example.tld jb A '192.168.0.3' -U 'administrator'%'P@s$w0rd.123'
samba-tool dns add localhost 0.168.192.in-addr.arpa 3 PTR jb.example.tld.' -U 'administrator'%'P@s$w0rd.123'
samba-tool dns add localhost example.tld conference CNAME 'jb.example.tld' -U 'administrator'%'P@s$w0rd.123'
samba-tool dns add localhost example.tld _xmpp-client._tcp SRV 'jb.example.tld 5222 5 0' -U 'administrator'%'P@s$w0rd.123'
samba-tool dns add localhost example.tld _xmpp-server._tcp SRV 'jb.example.tld 5269 5 0' -U 'administrator'%'P@s$w0rd.123'
```

Comprobaciones

```
host -t SRV _xmpp-server._tcp.example.tld
host -t SRV _xmpp-client._tcp.example.tld
host -t A jb.example.tld
dig -t SRV @example.tld _xmpp-client._tcp.example.tld
dig -t SRV @example.tld _xmpp-server._tcp.example.tld
```

## Integración con Samba AD DC

Crear nueva Cuenta de Usuario para el servicio `ejabberd`.

```
samba-tool user create 'ejabberd' 'P@s$w0rd.012' \
    --surname='XMPP Service' \
    --given-name='eJabberd' \
    --company='EXAMPLE' \
    --description='eJabberd XMPP Service Account'
```

`samba-tool user setexpiry ejabberd --noexpiry`

Definir cuentas de usuarios con acceso administrativo al servicio.

`nano /etc/ejabberd/ejabberd.yml`

```
acl:
  admin:
    user:
      - "administrator@example.tld"
      - "john.doe@example.tld"
```

Editar el fichero `/etc/ejabberd/ejabberd.yml` de forma tal que las opciones de la sección `AUTHENTICATION` queden como sigue:

```
auth_password_format: scram
fqdn: "jb.example.tld"
auth_method: ldap
ldap_servers: "dc.example.tld"
ldap_encrypt: none
ldap_port: 389
ldap_rootdn: "ejabberd@example.tld"
ldap_password: "P@s$w0rd.012"
ldap_base: "OU=ACME,DC=example,DC=tld"
ldap_uids: {"sAMAccountName": "%u"}
ldap_filter: "(&(objectClass=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))"
```

## Compartiendo los roster de los usuarios

 Editar el fichero `/etc/ejabberd/ejabberd.yml` y añadir en la sección `MODULES`, debajo de la opción `mod_roster: {}`, el siguiente contenido:

```
mod_shared_roster_ldap:
  ldap_base: "OU=ACME,DC=example,DC=tld"
  ldap_groupattr: "department"
  ldap_groupdesc: "department"
  ldap_memberattr: "sAMAccountName"
  ldap_useruid: "sAMAccountName"
  ldap_userdesc: "cn"
  ldap_rfilter: "(objectClass=user)"
  ldap_filter: "(&(objectClass=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))"
```

## Personalizando las vCard de los usuarios

Editar el fichero `/etc/ejabberd/ejabberd.yml` y añadir en la sección `MODULES`, debajo de la opción `mod_time: {}`, el siguiente contenido:

```
mod_vcard_ldap:
  ldap_uids: {"sAMAccountName": "%u"}
  matches: infinity
  ldap_vcard_map:
    "NICKNAME": {"%s": ["givenName"]}
    "FN": {"%s": ["displayName"]}
    "EMAIL": {"%s": ["mail"]}
    "GIVEN": {"%s": ["givenName"]}
    "MIDDLE": {"%s": ["middleName"]}
    "FAMILY": {"%s": ["sn"]}
    "ORGNAME": {"%s": ["company"]}
    "ORGUNIT": {"%s": ["department"]}
    "TITLE": {"%s": ["title"]}
    "TEL": {"%s": ["telephoneNumber"]}
    ldap_search_fields:
      "User": "%u"
      "Full Name":  "displayName"
      "Email": "mail"
    ldap_search_reported:
      "Full Name": "FN"
      "Nickname": "NICKNAME"
      "Email": "EMAIL"
```

Reiniciar el servicio y comprobar su correcto funcionamiento.

```
systemctl restart ejabberd
systemctl status ejabberd
```

## Comprobaciones

Acceder a la web admnistrativa que provee eJabberd desde un navegador, iniciar sesión con un usuario administrador y revisar los parámetros de configuración establecidos.

```
https://jb.example.tld:5280/admin
```

Vale destacar que una vez intregado el servicio al AD DC, no es necesario realizar cambio alguno a los usuarios, por esta vía; pues son gestionados en el mismo AD DC.

Archivo de registros de eJabberd

```
tailf -n100 /var/log/ejabberd/ejabberd.log
```
