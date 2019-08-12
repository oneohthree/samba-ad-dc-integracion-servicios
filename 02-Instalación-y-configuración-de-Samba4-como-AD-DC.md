# Instalación y configuración de Samba4 como AD DC

Las distribucións de Debian 9/10 cuentan en sus repositorios de paquetes con las versiones de Samba 4.5.16 y 4.9.5, respectivamente; las cuales no contienen algunas mejoras para la gestión de Unidades Organizativas mediante la herramienta `samba-tool`. Es por ello que se recomienda usar un repositorio de paquetes de la versión 4.9.6 o superior. En esta guía se usará el que proporciona el grupo francés [Tranquil IT Systems](http://samba.tranquil.it/debian/).

## Instalación de paquetes necesarios

Deshabilitar la interacción de configuración y proceder con la instalación de paquetes.

```
export DEBIAN_FRONTEND=noninteractive
apt install samba krb5-user winbind libnss-winbind net-tools
unset DEBIAN_FRONTEND
```

## Preparación del aprovisionamiento

Detener y deshabilitar todos los servicios relacionados con Samba.

```
systemctl stop samba-ad-dc smbd nmbd winbind
systemctl disable samba-ad-dc smbd nmbd winbind
```

Opcionalmente se puede hacer una copia del archivo de configuración inicial de Samba ya que se sobrescribe durante el aprovisionamiento.

```
mv /etc/samba/smb.conf{,.org}
```

## Aprovisionamiento Samba Active Directory Domain Controller

### Modo interactivo

```
samba-tool domain provision --use-rfc2307 --interactive
```

Aceptar los valores por defecto a menos que se desee lo contrario.

### Modo no interactivo

```
samba-tool domain provision \
    --server-role=dc \
    --use-rfc2307 \
    --dns-backend=SAMBA_INTERNAL \
    --realm=EXAMPLE.TLD \
    --domain=EXAMPLE \
    --function-level=2008_R2 \
    --adminpass='P@s$w0rd.123'
```

Editar el archivo de configuración de Samba resultante.

`nano /etc/samba/smb.conf`

```
[global]
    dns forwarder = 127.0.0.1
    netbios name = DC
    realm = EXAMPLE.TLD
    server role = active directory domain controller
    workgroup = EXAMPLE
    idmap_ldb:use rfc2307 = yes
    ldap server require strong auth = no
[netlogon]
    path = /var/lib/samba/sysvol/example.tld/scripts
    read only = No
    create mask = 0700
    directory mask = 0644
[sysvol]
    path = /var/lib/samba/sysvol
    read only = No
    create mask = 0700
    directory mask = 0644
```
La directiva `ldap server require strong auth = no` en la sección `[global]` se utiliza para permitir el acceso por el puerto `tcp\389`.

Las directivas `create mask = 0700` y `directory mask = 0644` en las secciones `[netlogon]` y `[sysvol]` son para la correcta asignación de permisos tanto a ficheros como directorios.

## Configuración de Kerberos

Durante el aprovisionamiento, Samba crea un archivo de configuración con los valores necesarios para el correcto funcionamiento del AD DC.

Utilizar la configuración de Kerberos generada durante el aprovisionamiento.

```
mv /etc/krb5.conf{,.org}
ln -s /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

Editar configuración de Kerberos

`nano /etc/krb5.conf`

```
[libdefaults]
    default_realm = EXAMPLE.TLD
    dns_lookup_realm = false
    dns_lookup_kdc = true
[realms]
    EXAMPLE.TLD = {
        kdc = dc.example.tld
        master_kdc = dc.example.tld
        admin_server = dc.example.tld
        default_domain = example.tld
    }
[domain_realm]
    .example.tld = EXAMPLE.TLD
    example.tld = EXAMPLE.TLD
```

Iniciar, verificar el estado y habilitar el servicio de Samba AD DC.

```
systemctl unmask samba-ad-dc
systemctl start samba-ad-dc
systemctl status samba-ad-dc
systemctl enable samba-ad-dc
```

Evitar que la cuenta del usuario `Administrator` expire.

```
samba-tool user setexpiry Administrator --noexpiry
```

Reiniciar el servidor

## Comprobaciones

Comprobar el nivel del dominio.

```
samba-tool domain level show
```

Resolución del nombre de domino, `FQDN` y `hostname` por la dirección IP estática.

```
ping -c4 example.tld
ping -c4 dc.example.tld
ping -c4 dc
```

Solicitar ticket de Kerberos

```
kinit Administrator@EXAMPLE.TLD
```

Listar tickets de Kerberos en caché

```
klist
```
