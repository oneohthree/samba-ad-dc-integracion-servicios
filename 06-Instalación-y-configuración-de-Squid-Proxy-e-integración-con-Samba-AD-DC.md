# Instalación y configuración de Squid Proxy e integración con Samba AD DC

Crear registros DNS

```
samba-tool dns add localhost example.tld proxy A '192.168.0.2' -U 'administrator'%'P@s$w0rd.123'
samba-tool dns add localhost 0.168.192.in-addr.arpa 2 PTR 'proxy.example.tld.' -U 'administrator'%'P@s$w0rd.123'
```

Crear nueva Unidad Organizativa `Proxy` para grupos de navegación, perteneciente a `ACME`

```
samba-tool ou create 'OU=Proxy,OU=ACME,DC=example,DC=tld' --description='Proxy Groups Organizational Unit'
```

Crear nuevos grupos de navegación pertenecientes a la OU `Proxy`

```
samba-tool group add Intranet --groupou='OU=Proxy,OU=ACME' --description='.CU Access Group'
samba-tool group add Internet --groupou='OU=Proxy,OU=ACME' --description='Internet Access Group'
samba-tool group add Unrestricted --groupou='OU=Proxy,OU=ACME' --description='Unrestricted Access Group'
```

Crear nuevos usuarios de navegación pertenecientes a la OU `ACME`

```
samba-tool user create 'sheldon' 'Amy*123' \
    --userou='OU=ACME' \
    --surname='Cooper' \
    --given-name='Sheldon' \
    --department='PHYSICS' \
    --company='EXAMPLE' \
    --description='Intranet Access Account' \
    --mail='sheldon@example.tld'

samba-tool user create 'leonard' 'Penny*456' \
    --userou='OU=ACME' \
    --surname='Hofstadter' \
    --given-name='Leonard' \
    --department='PSYCHOLOGY' \
    --company='EXAMPLE' \
    --description='Internet Access Account' \
    --mail='leonard@example.tld'

samba-tool user create 'rajesh' 'Howard*789' \
    --userou='OU=ACME' \
    --surname='Koothrappali' \
    --given-name='Rajesh' \
    --department='ASTROLOGY' \
    --company='EXAMPLE' \
    --description='Unrestricted Access Account' \
    --mail='rajesh@example.tld'
```

Añadir usuarios a los grupos creados

```
samba-tool group addmembers 'Intranet' sheldon
samba-tool group addmembers 'Intranet' sheldon,leonard
samba-tool group addmembers 'Unrestricted' rajesh
```

## Instalación de paquetes necesarios

```
export DEBIAN_FRONTEND=noninteractive
apt install squid krb5-user msktutil libsasl2-modules-gssapi-mit
unset DEBIAN_FRONTEND
```

## Configuración de Kerberos

```
mv /etc/krb5.conf{,.org}
nano /etc/krb5.conf
```

```
[libdefaults]
    default_realm = EXAMPLE.TLD
    dns_lookup_realm = false
    dns_lookup_kdc = true
    clockskew = 3600
```

Generar archivo keytab

```
msktutil -c -b "CN=Computers" \
    -s HTTP/proxy.example.tld \
    -h proxy.example.tld \
    -k /etc/krb5.keytab \
    --computer-name PROXY \
    --upn HTTP/proxy.example.tld \
    --server dc.example.tld \
    --verbose
```

Establecer los permisos del archivo keytab

```
chown root:proxy /etc/krb5.keytab
chmod 640 /etc/krb5.keytab
```

Comprobar que Kerberos funciona

```
kinit -k HTTP/proxy.example.tld
klist
```

Comprobar que la cuenta de host se actualice correctamente

```
msktutil --auto-update --verbose --computer-name proxy
```

Agregar en crontab

`nano /etc/contrab`

```
59 23 * * * root msktutil --auto-update --verbose --computer-name proxy > /dev/null 2>&1
```

## Integración con Samba AD DC

Crear nueva Cuenta de Usuario para el servicio `squid3`

Esta cuenta sería usada para propiciar la autenticación básica LDAP en caso de fallar Kerberos ó para uso de gestores de descargas no compatibles con Kerberos ó en aquellas estaciones que no están unidas al dominio.

```
samba-tool user create 'squid3' 'P@s$w0rd.789' \
    --surname='Proxy Service' \
    --given-name='Squid3' \
    --company='EXAMPLE' \
    --description='Squid3 Proxy Service Account'
```

```
samba-tool user setexpiry squid3 --noexpiry
```

Editar el fichero `/etc/squid/squid.conf` y agregar los métodos de autenticación.

`nano /etc/squid/squid.conf`

```
...
# Kerberos authentication
auth_param negotiate program /usr/lib/squid/negotiate_kerberos_auth -r -s GSS_C_NO_NAME
auth_param negotiate children 20 startup=0 idle=1
auth_param negotiate keep_alive off

# Basic LDAP authentication (fallback)
auth_param basic program /usr/lib/squid/basic_ldap_auth -R -b "dc=example,dc=tld" -D squid3@example.tld -w "P@s$w0rd.789" -f (|(userPrincipalName=%s)(sAMAccountName=%s)) -h dc.example.tld
auth_param basic children 10
auth_param basic realm PROXY.EXAMPLE.TLD
auth_param basic credentialsttl 8 hours
...
```

## Comprobaciones

Usando autenticación Kerberos.

`/usr/lib/squid/ext_kerberos_ldap_group_acl -a -g Internet -D EXAMPLE.TLD`

```
leonard
OK
```

Usando autenticación básica LDAP

```
/usr/lib/squid/basic_ldap_auth -R -b "dc=example,dc=tld" \
    -D squid3@example.tld \
    -w "P@s$w0rd.789" \
    -f sAMAccountName=%s \
    -h dc.example.tld

sheldon Amy*123
OK
```

```
/usr/lib/squid/ext_ldap_group_acl -R -K -S -b "dc=example,dc=tld" \
    -D squid3@example.tld -w "P@s$w0rd.789" \
    -f "(&(objectClass=person)(sAMAccountName=%v)(memberof=cn=%g,ou=Proxy,ou=ACME,dc=example,dc=tld))" \
    -h dc.example.tld
    
rajesh Unrestricted
OK
```

Analizando trazas de navegación

`tailf -n100 /var/log/squid/access.log`
